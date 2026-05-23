---
name: commit
description: Use this skill whenever the user wants to create a git commit, asks to "commit"、"提交"、"写 commit message"、调用 /commit，或 implementation 完成后准备落库时。生成符合本仓库规范的 commit message（中文、conventional 风格、可选 scope、可选 --revanxAiCoding 后缀），并完成 git commit。即使用户只说"帮我提交一下"也要触发本 skill。
---

# Commit

按本仓库规范生成 commit message 并提交。规范来自维护者长期沉淀的实践，目的是让 commit 历史**简洁清晰、可检索、不讲故事**。

## 规范定义

### 首行格式

```
<type>[(<scope>)]: <description> [--revanxAiCoding]
```

- **type**（必填，**只能从下表中选**）：

  | type | 含义 |
  |------|------|
  | `feat` | 新功能 |
  | `fix` | bug 修复 |
  | `refactor` | 重构（不改外部行为） |
  | `chore` | 杂项（版本号、配置、依赖升级等） |
  | `test` | 测试相关 |
  | `docs` | 文档 |
  | `style` | 代码风格（空格、格式化） |
  | `perf` | 性能优化 |
  | `build` | 构建系统/产物 |
  | `ci` | CI/CD 配置 |
  | `revert` | 回滚 |

- **scope**（可选）：模块/子项目名，用括号包裹。常见取值参考最近 commit（如 `valos`、`revanx-cloud`、`redoc`、`auth`、`inside-web-app` 等）。**单仓改动建议带 scope，跨多个无关模块可省略。**
- **description**（必填）：**中文**，陈述句，写"做了什么"，不写"为什么"也不讲故事。控制在 50 个字以内。**结尾不加句号。**
- **`--revanxAiCoding`**（可选）：**仅当用户明确声明"这次提交是 AI 写的 / 加上 AI 标记 / revanxAiCoding"时才追加。** 默认不加，不要自作主张。

### 补充信息（可选）

如果首行无法表达完整改动，与首行**空一行**后用 `- ` 列出分点：

```
<type>(<scope>): <description>

- 第一点
- 第二点
- 第三点
```

每条分点同样**简洁陈述**，不讲故事。如果只有一两处简单改动，**不需要**补充点，首行足够。

### 风格底线

- 不要写 "经过深入思考"、"为了解决..."、"由于..." 这类叙事
- 不要罗列文件路径
- 不要加 emoji（除非用户明确要求）
- 不要写英文（除非全是英文标识符如类名）
- 不要在 description 中重复 type（如 `feat: 新增功能` ❌ → `feat: 支持 X` ✅）

## 工作流

收到 commit 请求后按顺序执行：

### 1. 收集上下文

```bash
git status
git diff --staged   # 若已 staged
git diff            # 若未 staged
git log -5 --oneline   # 参考最近 commit 风格
```

- 若没有 staged 改动且有 unstaged 改动，**先和用户确认是否需要 `git add`**，不要默认全部 `git add .`

### 2. 判断是否需要原子化拆分

审视 diff 内容，判断是否应拆分为多个原子提交。满足以下**任一**条件时执行拆分：

- **diff 范围较大**（跨 5 个以上文件，或单文件改动超 200 行），且改动可按逻辑独立分组
- **diff 明显包含多个不交叉的改动**（例如：一次性同时做了 bug 修复 + 新功能 + 代码格式化）

**拆分流程：**

1. 按改动的逻辑归属分组（按功能、模块或目的），列出拆分方案
2. 将拆分方案展示给用户确认（用户可能有不同的拆分偏好）
3. 用户确认后，按组依次 `git add <files>` → 生成 commit message → `git commit`，每组独立走一遍后续步骤 3-5

**不拆分的情况：**

- 改动虽然跨多文件，但属于同一个逻辑变更（如一个 refactor 涉及的所有文件）
- 改动量小，拆分反而增加噪音
- 用户明确表示不想拆分

### 3. 生成 commit message

基于 diff 判断：
- type：改动性质（新增 → feat，修复 → fix，无行为变化的整理 → refactor，等等）
- scope：改动主要落在哪个子项目/模块（参考 VectorX.md 中的目录结构）
- description：用一句话陈述做了什么
- 是否需要补充分点：改动包含 2 个以上独立小变更时加

### 4. 与用户确认（推荐，非强制）

简单改动可直接提交。涉及关键模块、跨多文件、或有歧义时，先把 message 给用户看一眼再提交。

### 5. 执行提交

```bash
git commit -m "<生成的 message>"
```

多行 message 用多个 `-m` 或 here-doc：

```bash
git commit -m "feat(valos): 接入 JWT 鉴权" -m "- 新增 JwtAuthenticationFilter
- 替换原手写鉴权逻辑
- 白名单走 permitAll"
```

提交后 `git log -1` 展示结果给用户。

## 示例

**示例 1 — 简单 fix（无 scope，无补充）**
```
fix: 修复 AskUserQuestion data 为 undefined 时崩溃
```

**示例 2 — 带 scope 的 feat + 补充点**
```
feat(valos): SessionController 接入 JWT 鉴权

- 新增 JwtAuthenticationFilter 解析 token
- 替换原手写鉴权逻辑为 SecurityContext
- 白名单路径走 permitAll
```

**示例 3 — 用户明确说明是 AI 编码**
```
feat(revanx-vscode): 多仓任务下回填仓库分支信息 --revanxAiCoding

- 创建活动弹框支持多仓回填
- 新建任务弹框统一展示分支
```

**示例 4 — refactor 单行**
```
refactor(valos): 合并 /approve 与 /answer 为统一 /responses 端点
```

**示例 5 — chore**
```
chore(inside-web-app): 升级 Spring Boot 至 2.7.18
```

## 反例

```
feat: 经过深入思考后我们决定重构这个模块，因为之前的实现有问题，所以本次提交修改了 A.kt B.kt C.kt 三个文件，主要做了以下事情……
```

问题诊断：
- 讲故事（"经过深入思考"、"因为之前"）
- 罗列文件路径
- description 过长
- 缺少 scope
- 该用补充分点的地方写在了首行

正确写法：
```
refactor(valos): 重构 SessionController 鉴权链路

- 抽取 AuthFilter
- 移除冗余 try-catch
- 统一异常返回格式
```

## 何时拒绝/追问

- 工作区干净（`git status` 没改动）→ 告知用户没有可提交内容
- 改动跨越多个明显无关的子项目 → 建议拆分
- 用户给的描述完全无法对应 diff → 追问是否漏 add 文件
