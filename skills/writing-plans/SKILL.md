---
name: writing-plans
description: "Use this skill to create implementation plans from design specs, brainstorming outputs, or any solution description. Produces a structured plan document with task decomposition, dependency topology, declarative task descriptions, and acceptance criteria. Each task is independently committable. Use this skill whenever the user asks to 'write a plan', 'create an execution plan', 'break down tasks', 'make an implementation plan', or when transitioning from brainstorming/design to implementation. Also use when the user says things like '制定执行计划', '拆分任务', '写计划', or references docs/superpowers/plans/. The brainstorming skill invokes this as its terminal step — if you just finished brainstorming, this is your next skill."
---

# Writing Implementation Plans

Turn a design or solution description into a structured, executable implementation plan. The plan decomposes work into independently committable tasks with clear boundaries, ordered by dependency topology.

## When This Skill Runs

This skill is the bridge between **design** and **implementation**. It takes a solution that's been thought through (via brainstorming, a spec document, or a clear verbal description) and produces a plan that can be executed task-by-task, each task resulting in one clean commit.

## Inputs

The skill accepts any of these as input:

1. **Spec document** — a `docs/superpowers/specs/*.md` file (most structured)
2. **Brainstorming output** — the design approved in a brainstorming session (available in conversation context)
3. **Verbal description** — the user describes what they want to build (least structured; you'll need to ask clarifying questions)

If the input is underspecified, ask targeted questions to fill gaps before writing the plan. Don't guess at architectural decisions — those should have been resolved during design.

## Process

### 1. Understand the solution

Read the spec or design thoroughly. Identify:
- The core abstractions and their relationships
- Natural boundaries where work can be split
- Which parts depend on which (what must exist before something else can be built)
- What the final integrated system looks like

If working from a spec file, read it completely before starting. If working from conversation context, summarize your understanding and confirm with the user before proceeding.

### 2. Decompose into tasks

Split the work into tasks where **each task can be independently committed and the codebase compiles after each commit**. This is the fundamental constraint — it shapes everything about how you split work.

**Guiding principles for task boundaries:**

- **Bottom-up by dependency**: start from the leaf dependencies (things with no prerequisites) and work up. Each task should only depend on tasks before it in the topology.
- **One concept per task**: a task implements one coherent unit — a layer, a module, a protocol component. If you find yourself writing "and also..." in a task description, consider splitting.
- **Compilable after each commit**: every task must leave the codebase in a compilable state. This means you can't split a type definition from its only consumer if the consumer won't compile without it.
- **Testable in isolation**: each task should include its own tests. Don't defer all testing to the end — a task without verification is a task you can't trust.

**Typical decomposition patterns:**

- Layer-by-layer (L1 → L2 → L3 → ... for layered architectures)
- Module-by-module (for independent modules that can be built in parallel)
- Core → Extensions (build the minimal core first, then add capabilities)
- Infrastructure → Business logic (utilities and types first, then logic that uses them)

### 3. Establish task topology

Draw the dependency graph explicitly. Use ASCII art to make the relationships visual:

```
T1 (foundation)
    ↓
T2 ← T3        (T2 and T3 both depend on T1; T3 also feeds T4)
    ↓      ↓
T4 ←────┘
    ↓
T5 (integration)
```

State the dependency rules clearly below the graph. Identify which tasks can run in parallel.

### 4. Write each task description

Each task uses this **declarative 5-element structure**:

#### Goal (目标)
One sentence: what does this task accomplish? What capability exists after this task that didn't exist before?

#### Scope (范围)
Concrete list of files to create or modify, types to define, APIs to implement. Be specific — file paths, class names, function signatures. This is the "what", not the "how".

#### Constraints (约束)
Hard rules this task must respect. These are non-negotiable requirements — compatibility constraints, API contracts, threading rules, version limitations. Only list constraints that the implementer might otherwise violate. Don't restate obvious things.

#### Approach (核心思路)
A brief description of the core implementation idea — just enough to point the implementer in the right direction without dictating every line of code. Cover the key algorithm, data structure choice, or architectural pattern. This section should be short (3-8 lines typically). If you need more, the task is probably too big.

#### Acceptance Criteria (验收标准)
Objective, verifiable criteria. Three levels:

1. **Compilation**: `./gradlew :module:compileKotlin` or equivalent passes with zero errors
2. **Unit tests**: specific test cases listed by name, covering the core behaviors and edge cases introduced in this task
3. **Integration/compatibility tests** (where appropriate): for tasks where correctness depends on cross-boundary compatibility (e.g., binary protocol compatibility, API contracts), include golden tests or end-to-end verification with concrete test data

Write tests that would actually catch bugs. "Tests pass" is not a criterion — describe *which* behaviors are tested.

#### Commit Message
A conventional commit message (scope optional) that concisely describes the task's deliverable. The commit message should work standalone — someone reading `git log` should understand what this commit added.

```
feat(module): brief description of what was added (T<N>)

- key deliverable 1
- key deliverable 2
- notable test coverage
```

### 5. Write final integration acceptance criteria

After all individual tasks, add a section that defines what "done" looks like for the entire plan. This includes:

- **Full build**: the complete compilation command and expected result
- **Full test suite**: all tests pass (with expected count ranges)
- **Coverage matrix**: a table showing which layers/modules are covered by which test files, with minimum case counts
- **Cross-cutting concerns**: protocol compatibility, thread safety, code quality standards, documentation requirements
- **File manifest**: the complete list of files created, organized by purpose (source / test / resources)

## Language

Write the plan in the same language as the input. If the spec document is in Chinese, write the plan in Chinese. If the user describes the solution in English, write in English. When in doubt, match the user's language. Task structure headings (Goal/Scope/Constraints/etc.) should also follow this rule — use 目标/范围/约束/核心思路/验收标准 for Chinese plans.

## Output Format

Save the plan to `docs/superpowers/plans/YYYY-MM-DD-<topic>.md` (or `<topic>-plan.md` if the topic alone is ambiguous).

The document structure:

```markdown
# <Title>

> Date: YYYY-MM-DD
> Spec: `specs/<spec-file>.md` (if applicable)
> Code location: `path/to/code/`
> Test location: `path/to/tests/`

---

## Task Topology

(ASCII dependency graph + dependency rules)

---

## T1: <Task Title>

### Goal
### Scope
### Constraints
### Approach
### Acceptance Criteria
### Commit

---

## T2: <Task Title>
...

---

## Final Integration Acceptance Criteria

### Build
### Test Coverage
### Cross-cutting Concerns
### File Manifest
```

## Quality Checklist

Before presenting the plan, verify:

- [ ] Every task compiles independently (no forward references to unwritten code)
- [ ] Dependency topology has no cycles
- [ ] No task is so large that it would be hard to review in a single PR (rough guideline: < 500 lines of production code per task)
- [ ] Acceptance criteria are objective (another person could verify pass/fail without ambiguity)
- [ ] Golden tests / compatibility tests exist for any cross-boundary contract
- [ ] The final integration criteria would catch a regression in any individual task
- [ ] Commit messages are meaningful in isolation (readable in `git log`)

## After the Plan

Once the plan is written and saved:

1. Commit the plan document
2. Tell the user the plan is ready and where it's saved
3. Ask if they want to review before proceeding to implementation
4. When ready to implement, suggest using `executing-plans` or `subagent-driven-development` skill
