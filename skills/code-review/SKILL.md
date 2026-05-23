---
name: code-review
description: >
  Code Review skill：对 git diff 或 commit 范围内的代码变更进行系统化七层检查，输出按严重程度分层的检查报告。
  用户手动触发：`/code-review`、"帮我 review 一下"、"review 这些改动"、"CR 一下"、"看看这些代码"、"检查一下改动"等。
  支持参数指定 commit 范围（如 `/code-review HEAD~3..HEAD`）、分支名（如 `/code-review feat-ipc`，review 该分支从分叉点到最新的所有变更），无参数时默认 review `git diff HEAD`。
  当用户完成编码后说"帮我看看"、"有没有问题"等也应触发此 skill。
---

# Code Review

对 git diff 或 commit 范围内的代码变更进行系统化七层检查（L0-L6），输出按严重程度分层的检查报告。当前 agent 直接执行，不派发子 agent。

## 输入解析

按以下优先级确定 review 范围：

1. **用户显式指定 commit 范围**：`/code-review HEAD~3..HEAD`、`/code-review abc123..def456`
2. **用户指定分支名**：`/code-review feat-ipc`、`/code-review feature/my-branch`
   - 自动找到该分支与主干（main/master）的分叉点：`git merge-base main <branch>`
   - 然后 diff 从分叉点到分支最新：`git diff $(git merge-base main <branch>)..<branch>`
   - 如果用户指定了对比基准分支（如 `/code-review feat-ipc..develop`），则用指定的基准
   - 判断参数是否为分支名：运行 `git rev-parse --verify <arg>` 检查，若为有效 ref 且不是 `HEAD~N` 或 `commit..commit` 格式，则按分支模式处理
3. **用户指定单个 commit**：`/code-review HEAD~1` → 等价于 `HEAD~1..HEAD`
4. **无参数**：默认 `git diff HEAD`（staged + unstaged changes）

确定范围后，运行对应的 git diff 命令获取变更内容。如果 diff 为空，告知用户没有可 review 的变更。

## 检查流程

```
获取 diff → 变更文件 > 20 个时提示用户缩小范围 → 按文件分组 → 读取变更文件上下文 → 逐层检查（L0→L6）→ 汇总报告
```

### 读上下文

不只看 diff 行——还要读取变更文件的完整内容（至少变更函数的完整定义），理解改动的上下文。这是产出高质量 review 的关键：脱离上下文的 diff 容易误判。

### 读项目风格

扫描同目录/同模块的已有代码，作为 L4e（风格一致性）和 L6（一致性）的参照基线。以项目现有代码为准，不以"业界最佳实践"为准。

## 七层检查体系

按 L0→L6 顺序逐层检查。每个 finding 必须包含 `file:line` 引用，不允许模糊描述。

### L0 正确性（Critical）

- 逻辑 bug、条件判断错误
- 空指针 / NPE / undefined access
- 类型转换不安全
- Off-by-one、边界条件未处理
- 未处理的枚举分支 / switch-case 遗漏

### L1 安全性（Critical）

- SQL / XSS / 命令注入
- 硬编码凭证、密钥、token
- 权限校验遗漏
- 敏感信息写入日志
- 不安全的反序列化

### L2 架构（Important）

- 职责划分是否清晰（单一职责）
- 依赖方向是否正确（高层不应依赖低层实现细节）
- 破坏性 API 变更未标注
- 循环依赖引入

### L3 健壮性（Important）

- 资源未关闭（流、连接、文件句柄）
- 并发安全问题（共享可变状态、race condition）
- 缺少超时控制
- 异常处理过宽（catch Exception/Throwable）

### L4 务实性（Important）— Agent 特有检查

这一层是本 skill 的核心差异化——专门检查 Agent 生成代码中常见的"看起来对但实际有害"的模式。

#### 4a 过度抽象（Over-engineering）

检查信号：
- 新增的抽象层/接口/工厂只有 1 个实现
- 新增的抽象没有解决当前存在的实际问题，仅仅是"结构上看起来更优雅"
- 为假想的未来需求预留扩展点

判断标准（三问，任一不利即标记）：
1. 这个抽象 **解决了什么具体的、当前存在的问题**？（不能回答则过度）
2. 去掉这层抽象，代码会 **变得更难理解还是更容易理解**？
3. 消费者是否 >= 2 个？

#### 4b 伪解决方案（Workaround Masking）

检查信号：
- Mock/stub 了外部依赖并忽视了真实集成问题
- TODO 注释说"待对接"但没有对应 issue/ticket
- 绕过了真实 API 调用
- 硬编码了本应从外部获取的值

判断标准：workaround 是否有明确的清理计划和跟踪机制。

#### 4c 问题延迟暴露（Silent Failure）

检查信号：
- `catch` 块只 log 不 rethrow
- 空值时返回默认值而非 fail-fast
- `Optional.orElse(defaultValue)` 掩盖了不应出现的空值

判断标准：fallback 是否用在了不应该出现异常的路径上（应 fail-fast 的地方用了 fallback）。

#### 4d 注释噪音（Comment Noise）

检查信号：
- 对自明代码加注释（`// 获取用户列表` 在 `getUserList()` 上方）
- 大段设计方案文本贴入代码注释
- 每个方法都有 JSDoc/KDoc 但内容只是重复方法签名

判断标准：注释是否解释了 **为什么（why）** 而非 **是什么（what）**。

#### 4e 风格不一致（Style Inconsistency）

检查信号：
- 命名风格与项目不一致（如项目用驼峰但新代码用下划线）
- 注释风格与项目不一致（项目不写 JSDoc 但新代码全量 JSDoc）
- 缩进、括号、空行风格不同
- Error message 语言不一致

判断标准：以项目现有代码为准，不以"最佳实践"为准。

### L5 可维护性（Minor）

- 圈复杂度过高（深层嵌套 >= 4 层）
- 明显的重复代码
- 命名不清晰 / 误导性命名
- 变更的核心逻辑是否有对应测试

### L6 一致性（Minor）

- 代码格式风格（缩进、括号、空行）
- Error message 语言不一致
- 提交信息规范

## 输出格式

严格使用以下模板输出，不要输出空泛的 "looks good" 评价——没有问题就直接说"无问题"。

```markdown
## Code Review Report

**范围**: `<commit-range-or-diff-description>`（N files changed, +X -Y）

### Critical (必须修复)
1. **[L0 正确性] <问题标题>** — `<file>:<line>`
   - 问题: <具体描述>
   - 影响: <会导致什么后果>
   - 建议: <如何修复>

### Important (建议修复)
2. **[L4a 过度抽象] <问题标题>** — `<file>:<line-range>`
   - 问题: <具体描述>
   - 建议: <如何改进>

### Minor (可选优化)
3. **[L5 可维护性] <问题标题>** — `<file>:<line>`
   - 问题: <具体描述>
   - 建议: <如何改进>

### 总结
- Critical: N | Important: N | Minor: N
- **评估**: <修复 Critical 后可合并 / 建议修复 Important 后合并 / 可直接合并>
```

如果某个严重程度没有 findings，省略该 section（不要输出空的 section）。

## 边界规则

- 只检查变更的代码行及其直接上下文，不做全仓库审计
- 每个 finding 必须包含 `file:line` 引用
- 变更文件 > 20 个时，提示用户是否缩小范围再继续
- 没有问题时输出"无问题，可直接合并"，不需要凑 findings
