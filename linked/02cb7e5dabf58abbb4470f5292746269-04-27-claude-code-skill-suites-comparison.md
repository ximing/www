---
title: Claude Code 六大 Skill 套件源码级对比：谁在解决什么问题，谁在制造新问题
date: 2026-04-27 22:14
series: ai-coding
tags: ["AICoding", "源码分析"]
description: 通读 OpenSpec、Spec Kit、Superpowers、GSD、ECC、GStack 六个仓库的源码，按研发生命周期逐一对比它们的命令、产物、核心机制、优缺点与适用场景，给出基于事实的选型建议。
published: true
---

不装套件直接使用 Claude Code 时，常见三类问题。会话变长后，窗口会积累过时信息，模型可能基于错误前提决策。需求描述与模型理解不一致时，产出越复杂，偏差越可能扩大，而且通常在产出完成后才发现。缺少类型检查、测试和浏览器验证时，模型无法获得执行结果的反馈。

这些 Skill 套件可按主要职责分为规范层和执行层。规范层将需求和设计保存为 Markdown 产物，由主会话实现；OpenSpec 和 Spec Kit 属于这一类。执行层涵盖规划到提交的流程，包含任务拆分、测试、审查或发布；Superpowers、GSD、ECC、GStack 属于这一类。两类存在交叉：Spec Kit 的 `/implement` 涉及执行，GStack 的 `/office-hours` 用于审查需求。

套件选择会影响产物目录、命名约定、Git 分支策略和 `CLAUDE.md` 内容。依赖这些结构后，移除套件还需要清理仓库中的产物。规范层套件的侵入性较低，但不覆盖实现阶段的上下文管理和反馈验证。

README 和社区转述中有过时或不准确的信息。本文依据仓库源码核对命令表、产物结构、编排逻辑和边界条件，并按套件和研发生命周期组织内容。

文中数字对应 GSD v1.50.0-canary.0、ECC v2.0.0。项目持续迭代，旧版本的数据可能与当前版本不同。

## OpenSpec：规格管理流程

OpenSpec（[Fission-AI/OpenSpec](https://github.com/Fission-AI/OpenSpec)，43.1K stars）是一个 Node.js CLI 工具，约 8,000 行 TypeScript 和 Markdown。它将需求和设计写成规格文档，按规格实现并在完成后归档；不提供测试编排、审查流程或部署。

### 命令与产物

常用命令包括：

- `/opsx:explore`：思考，不产出文件。用于理解问题空间，把推理过程留在对话里。
- `/opsx:propose`：生成四个产物：proposal（要做什么）、specs（功能规格）、design（技术设计）、tasks（任务列表）。这四个文件之间存在 DAG 依赖：proposal 先行，specs 和 design 依赖 proposal，tasks 依赖 specs 和 design。
- `/opsx:apply`：按 tasks 逐条实现。
- `/opsx:update`：修改已有规划产物并保持互相一致（只动产物，不动代码）。
- `/opsx:sync`：把 delta specs 合并回主 specs。archive 时会提示，也可以单独跑。
- `/opsx:archive`：归档。把这次变更的 specs 合并回主 specs，清理临时产物。

需切换 profile 启用的扩展命令包括：`/opsx:new`（只建 change 脚手架，产物留待后续生成）、`/opsx:continue`（按依赖图创建下一个产物）、`/opsx:ff`（fast-forward，一次创建全部规划产物）、`/opsx:verify`（验证实现与 specs 的一致性）、`/opsx:bulk-archive`（批量归档，自动检测并处理 change 间的 spec 冲突）、`/opsx:onboard`（引导式教程，说明完整工作流）。另有旧版命令 `/openspec:proposal`、`/openspec:apply`、`/openspec:archive`，当前仍可用但不在新文档推荐中。

任务状态有三个：BLOCKED（依赖未满足）、READY（可执行）、DONE（已完成）。状态流转靠依赖图驱动，不靠线性阶段号。

### Delta Spec 机制

一次变更只记录差异，不重写整个 specs：ADDED（新增规格条目）、MODIFIED（修改条目的字段）、REMOVED（删除条目）。归档时，Delta Spec 自动合并回主 specs。因此每次变更可以追溯到具体修改内容。多个 change 并行修改同一份 spec 时，`/opsx:bulk-archive` 会检测跨 change 冲突，并按代码库的实际实现状态决定合并顺序；语义冲突仍需人工判断。

### Schema 可定制

默认 schema 定义了 proposal、specs、design、tasks 四个产物的结构和字段。如果你的项目有领域特定的规格结构（比如 API 合约、数据库 schema），可以用 `openspec schema fork` 基于内置 schema 分叉出自己的工作流，放在项目的 `openspec/schemas/` 目录下随代码一起版本化。

### 跨宿主适配

同一份 Markdown 产物要能在不同 AI 工具里被调用。OpenSpec 的做法是同一命令名、不同拼写：Claude Code 用 `/opsx:propose`，Cursor 和 GitHub Copilot 用 `/opsx-propose`，Amazon Q 用 `@opsx-propose`，Codex 用 `$openspec-propose`。`openspec init` 会按你选的工具打印对应的调用形式。

### 设计原则与边界

OpenSpec 将其原则表述为 "Enablers, not gates"。依赖关系为 AI 提供上下文，不强制线性阶段。可以跳过 explore 直接 propose，也可以只生成 proposal 而不生成 tasks。它不提供质量门禁，空泛的 specs 或缺少验证标准的 tasks 不会被阻断。

OpenSpec 不包含 Review、QA、安全检查和部署，只覆盖需求规格与按任务实现。

仓库中没有 CLAUDE.md，AGENTS.md 为空。安装后不向项目上下文注入常驻信息；需要手动触发命令。

## Spec Kit：六阶段规格流程

Spec Kit（[github/spec-kit](https://github.com/github/spec-kit)，91K stars）是 GitHub 的 Python CLI 工具（`specify-cli`），约 25,000 行 Python 和 Markdown。它与 OpenSpec 的定位相似：OpenSpec 用六个命令完成一个特性，Spec Kit 设有六个阶段和对应的质量门禁。

### SDD 六阶段

核心命令按默认顺序包括：

1. `/speckit.constitution`：生成项目级宪法文件 `constitution.md`。这个文件定义项目的约束边界：技术栈、架构决策、禁止事项。后续命令执行时会加载它作为治理约束。
2. `/speckit.specify`：定义要构建什么，即需求和用户故事。执行时如果 constitution.md 存在会被加载，约束规格的方向。
3. `/speckit.plan`：生成技术实现计划。
4. `/speckit.tasks`：生成任务列表，带依赖关系和并行标记。
5. `/speckit.implement`：按任务列表实现。
6. `/speckit.converge`：对照 spec/plan/tasks 评估代码库现状，把差距追加为新任务。

`/speckit.taskstoissues` 将任务列表转为 GitHub Issues。可选命令包括：`/speckit.clarify`（在 plan 之前澄清规格中欠定义的部分）、`/speckit.analyze`（跨产物一致性与覆盖分析，在 tasks 之后、implement 之前运行）、`/speckit.checklist`（生成自定义质量检查清单，验证需求的完整性、清晰度和一致性）。官方文档将 checklist 描述为“给英文写单元测试”。

Spec Kit 在流程开始时使用 constitution，OpenSpec 的 proposal 则没有项目级约束的前置检查。Spec Kit 将边界写入 constitution，后续阶段的 prompt 加载该文件作为约束。限制是约束仅作用于 prompt 层，越界时没有代码级校验阻断。

### 三层定制体系

Spec Kit 的定制分为三层：Extension（扩展默认 schema）、Preset（预设配置组合）、Override（覆盖默认行为）。优先级依次为 Override、Preset、Extension 和默认配置。该体系包含明确的层级规则；使用时需要理解三层交互，错误的 Override 可能覆盖 Preset 配置。

### Bundle 系统与集成架构

Bundle 是 Spec Kit 的扩展包机制，支持搜索、安装、更新和移除社区扩展包。每个 Bundle 包含 schema 定制、命令扩展和上下文文件。`IntegrationManifest` 用 SHA-256 记录已安装文件的哈希值；卸载时会保留哈希不匹配的用户修改文件。

每个核心脚本有三套实现：POSIX shell、PowerShell、Python。三套实现功能一致，适配不同开发环境。Spec Kit 支持 30+ AI 工具集成，每个集成是独立子包，放在 `src/specify_cli/integrations/<key>/` 下。

上下文文件由 `agent-context` 扩展管理，CLI 不管理 CLAUDE.md。上下文注入是可选的，安装后不会自动发生。

### 边界

每个特性完成 Spec Kit 的六阶段流程需要 30 分钟到 3 小时，包括模型生成产物、人工审查和模型修改。修复 CSS typo 等小改动通常不适合这一流程。Spec Kit 不提供跳过阶段或缩短流程的机制，需要人工决定哪些特性使用完整流程。

和 OpenSpec 一样，没有 QA、没有安全检查、没有部署。仓库里没有 CLAUDE.md，AGENTS.md 有 603 行，定义了集成架构和扩展点，但不包含运行时行为指令。

## Superpowers：内置 TDD 的执行流程

Superpowers（[obra/superpowers](https://github.com/obra/superpowers)，168.7K stars）由 Jesse Vincent 和 Prime Radiant 团队维护。14 个 Skill，约 1,500 行胶水脚本，零第三方依赖。源码总量约 4,700 行 Markdown 和 shell。

社区文章常将 Superpowers 描述为“brainstorm / write-plan / execute-plan 三个命令”。仓库中没有 `commands/` 目录；对应的是 `brainstorming`、`writing-plans`、`executing-plans` 三个自动触发的 Skill。作为 model-invoked 技能，它们由模型根据任务阶段激活，无需手动输入斜杠命令。

### TDD 三层嵌套

Superpowers 在三个层面定义测试驱动开发：

1. 行为规则层：每个 Skill 的 SKILL.md 里声明该 Skill 的行为规则，行为规则本身是可测试的约束。比如 `implementing` Skill 声明"每个实现步骤必须报告 TDD Evidence"。
2. Plan 模板层：plan 模板将 RED-GREEN-REFACTOR 写为 checkbox。模型写计划时必须包含“写测试、跑测试确认红、写实现、跑测试确认绿、重构”等步骤，将 TDD 循环写入计划文件。
3. SDD 执行层：`implementing` Skill 要求 implementer 每完成一个任务报告 TDD Evidence，包括测试文件路径、红绿状态、覆盖率变化。

计划文件中的 checkbox 和证据报告要求模型完成 RED-GREEN 流程。这些要求属于 prompt 层约束，不是 CI 门禁；长会话中模型仍可能绕过，Superpowers 不提供运行时强制机制。

### SDD 子 Agent 编排

Superpowers 的 SDD（Spec-Driven Development）模式通过子 Agent 编排任务：

- 子 Agent 将全部产出写入文件，controller 的上下文窗口不保留这些中间过程。
- Plan-scoped workspace：每个子 Agent 只能访问当前 plan 声明的文件范围，防止越界修改。
- Ledger 记录：子 Agent 将关键决策写入 ledger 文件。上下文被压缩（compaction）后，仍可从文件恢复决策记录。
- 5 轮修复限制：子 Agent 连续 5 轮未能通过测试时，流程停止重试并将控制权交还给人。

### 内置审查流程

Superpowers 有两个审查相关的 Skill：`requesting-code-review` 和 `receiving-code-review`。前者发起审查请求，后者处理审查反馈。`finishing-a-development-branch` 处理分支收尾，包括合并前检查和清理。

审查流程可用但不强制。plan 可以跳过 code-review 步骤，Superpowers 不会阻断。

### 跨宿主与 Git Worktree

14 个 Skill 共 3,185 行 SKILL.md，支持 11 个宿主平台（Claude Code、Cursor、Codex、OpenCode 等）。Superpowers 还支持 Git worktree：每个任务在独立的 worktree 里执行，避免并行任务之间的文件冲突。

### 边界

Superpowers 不包含安全审计、浏览器 QA 和部署发布。流程包含人工确认点：README 描述为分段展示设计稿，用户输入 "go" 后才启动子 Agent 执行。这些确认与 TDD 约束同属 prompt 层约定，模型仍可能跳过确认直接执行。plan 阶段的架构错误会传递到执行阶段，修改已实现代码的成本高于修改 plan。

## GSD：长周期项目工作流

GSD（[gsd-build/get-shit-done](https://github.com/gsd-build/get-shit-done)，57.6K stars，v1.50.0-canary.0）包含 549 个测试文件，约 132,441 行测试代码和约 171,865 行源代码，总量约 30 万行。支持 15 种运行时。

社区文章常引用“29 Skills + 12 Agents + 20+ 命令”，这是早期版本的数据。当前版本没有独立的 Skills 概念，包含 67 个命令和 33 个 Agent。

### 命令路由：两阶段命名空间

67 个命令组织在 6 个命名空间下，每个命名空间是一个 Meta-Skill 路由器：`/gsd-workflow`、`/gsd-project`、`/gsd-quality`、`/gsd-context`、`/gsd-manage`、`/gsd-ideate`。模型先从 6 个命名空间选择（约 120 tokens），再路由到具体子命令。两阶段路由替代了早期版本对 86 个 skill 的扁平列举（约 2,150 tokens），每次调用约少用 2K tokens。

### Fresh Context 机制

GSD 使用 Fresh Context 管理上下文：每个 Agent 获得独立的 200K 或 1M 上下文窗口，不共享。编排器只调度，不执行；编排逻辑写在 workflow `.md` 文件中，由模型读取并按步骤派发。

上下文 Profile 分三档：

- `dev.md`：简洁，行动导向，适合实现阶段。
- `research.md`：冗长，探索性，适合研究阶段。
- `review.md`：关键细节导向，适合审查阶段。

Profile 的选择不是自动的，由 workflow 在派发 Agent 时指定。

Context Monitor Hook 持续监控上下文剩余空间：低于 35% 发 WARNING，低于 25% 发 CRITICAL。这提醒人手动干预（压缩上下文或拆分任务），但不自动执行任何动作。

使用 1M 上下文窗口的模型时，派发给 Agent 的上下文会自适应扩充：Executor Agent 获得前序 wave 的 SUMMARY.md 加当前阶段的 CONTEXT.md 和 RESEARCH.md；Verifier Agent 获得所有 PLAN.md、SUMMARY.md、CONTEXT.md、REQUIREMENTS.md。Verifier 的上下文比 Executor 更全面，因为验证需要看到全局约束。

### 阶段化工作流

GSD 的主流程包括六个阶段：

`new-project` → `discuss-phase` → `plan-phase` → `execute-phase` → `verify-work` → `ship`

辅助阶段五个：`ui-phase`、`spec-phase`、`mvp-phase`、`validate-phase`、`secure-phase`。辅助阶段不强制串入主流程，可以按需插入。

plan-phase 启动 4 路并行研究：STACK.md（技术栈评估）、FEATURES.md（特性清单）、ARCHITECTURE.md（架构方案）、PITFALLS.md（已知陷阱）。综合器按顺序合并四路结果。工具优先级为 Context7 MCP、CLI 回退、WebSearch、训练数据；可获取最新文档时优先使用文档。

### 按依赖关系分波执行

execute-phase 的任务执行采用波（wave）模型：

1. 自动依赖分析：根据任务间文件依赖关系构建依赖图。
2. 波分组：无依赖的任务分到同一波，有依赖的任务按依赖顺序分到不同波。
3. 波内并行：同一波内的任务由不同 Agent 并行执行。
4. 波间顺序：前一波全部完成后才启动下一波。

`STATE.md` 文件锁用于控制波间顺序：通过 `O_EXCL` 原子创建文件锁，锁在 10 秒后过期；获取失败时使用带抖动的自旋等待。过期机制可处理崩溃后残留的锁，抖动可降低多 Agent 同时重试的概率。

`--no-verify` 提交 + wave 后统一 pre-commit：每个任务提交时跳过 pre-commit（加快速度），整波完成后统一跑 pre-commit 检查。限制是单个任务提交时未运行 pre-commit 检查。

### Forensics 诊断

GSD 的 Forensics 诊断工具覆盖 6 种异常模式：

1. Stuck Loop：Agent 在同一任务上反复失败。
2. Missing Artifact：预期产物未生成。
3. Partial-plan Drift：部分计划与实际进展偏离。
4. Abandoned Work：已开始但未完成的任务。
5. Crash：Agent 非正常终止。
6. Scope Drift：任务范围超出计划。

Forensics 只读调查，不修改任何文件。输出数据脱敏（移除用户名、路径等），证据归因到具体的 Agent 和任务。

### 产物体系

GSD 的产物都放在 `.planning/` 目录下：PROJECT.md、REQUIREMENTS.md、ROADMAP.md、STATE.md、CONTEXT.md、RESEARCH.md、PLAN.md、SUMMARY.md、VERIFICATION.md、UAT.md、UI-SPEC.md。这些产物覆盖需求到验证的流程；项目方向调整时，需要同步更新多个文件。

### 14 个 Hook

GSD 注册了 14 个 Hook，覆盖 statusLine、PostToolUse、SessionStart、PreToolUse、PostToolUseFailure 等事件。三个防护 Hook 包括：

- `gsd-prompt-guard.js`：扫描 `.planning/` 目录写入的内容，检测注入模式（比如在 REQUIREMENTS.md 里嵌入指令让模型执行非计划操作）。
- `gsd-workflow-guard.js`：检测在 GSD 上下文之外的编辑，防止人手动改文件绕过流程。
- `gsd-read-guard.js`：阻止编辑未读文件，强制模型先读后改。

这些 Hook 属于 advisory，不阻断操作。它们会发出警告，但模型继续执行错误操作时无法阻止。

### 边界

GSD 包含 67 个命令，安装器有 11,376 行。纯 Markdown 编排没有运行时保证：模型误读 workflow `.md` 的步骤描述时，执行路径可能偏离，类型系统或解释器无法捕捉此类错误。Hook 为 advisory，且不接外部漏洞数据源。安全能力仅限 `/gsd-secure-phase`，该命令验证 plan 威胁模型中的缓解措施是否落到代码中并产出 SECURITY.md。GSD 不包含浏览器 QA。

## ECC：Agent Harness 套件

ECC（[affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code)，167.7K stars，v2.0.0）是六个套件中文件数最多的：3,407 个文件，约 159,000 行可执行代码加约 446,000 行 Markdown，文档体量约为可执行代码的 3 倍。

社区文章常引用“48 Agents + 183 Skills + 79 命令”，这是旧版本数据。当前版本包含 67 Agents、281 Skills、94 命令。

### AgentShield 的可验证范围

README 宣称 AgentShield 引擎有 102 条规则和 1282 条测试。AgentShield 引擎是独立 npm 包 `ecc-agentshield`，不在这个仓库中。本仓库仅有两个调用该包的文件：`commands/security-scan.md` 和 `skills/security-scan/SKILL.md`，它们调用 `npx ecc-agentshield scan`。102 条规则和 1282 条测试无法从本仓库验证。

本仓库自身的安全实现是另一套：每个 agent 内嵌 Prompt Defense Baseline（一组对抗注入的 prompt 约束），Hook 层有三个阻断脚本：`config-protection.js`（保护配置文件不被修改）、`block-no-verify.js`（阻止 `--no-verify` 提交）、`governance-capture.js`（捕获治理事件）。加上 `tdd-workflow` skill 的不可信输入处理（拒绝 `curl | sh`，要求人工审批）。这些是可验证的、存在于本仓库的安全措施。

### 多模型路由的运行依赖

ECC 的 `multi-plan` 和 `multi-execute` 命令在设计文档里描述了一个多模型协作方案：Code Sovereignty（外部模型零文件写权限，只输出 Unified Diff，Claude 作为唯一写盘者）、信任规则（后端听 Codex、前端听 Gemini）、SESSION_ID 接力。Code Sovereignty 让多个模型输出建议但不授予写权限，可避免多模型同时修改同一文件。

但当前版本开箱无法运行。`multi-plan` 和 `multi-execute` 依赖未随仓库分发的 `ccg-workflow` 运行时和 `~/.claude/.ccg/` 角色文件。仓库里没有这些文件，也没有安装脚本。README 描述的功能依赖未随仓库分发的核心运行时。

### Hook 生命周期

ECC 的 Hook 覆盖 7 类事件：SessionStart、SessionEnd、PreToolUse、PostToolUse、PostToolUseFailure、PreCompact、Stop。50 个 JS 脚本采用分发器模式：事件触发时，分发器遍历并执行该事件注册的脚本。

其中两个 Hook 的行为如下：

- `stop-format-typecheck.js`：会话结束时批量执行 format 和 typecheck，270 秒总预算。如果 typecheck 超时，输出剩余错误让用户手动处理。该 Hook 在每次会话结束时执行静态检查，以发现未验证的代码。
- `session-start.js`：会话开始时加载上次会话摘要和 instincts（置信度 0.7 阈值、最多 6 条、8000 字符上限）。Instincts 是从历史会话提取的高频行为模式，在新会话开始时加载。

缺陷：`hooks.json` 的引导代码在 30+ 个文件里复制粘贴，整个配置文件达 37KB。维护成本高，修改 Hook 配置需要同步多个文件。

### 跨平台支持范围

ECC 宣称支持 7+ 平台，各平台的支持范围不同：

- Claude Code：完整功能，所有 Agent、Skill、Hook 可用。
- Cursor：格式转换加委托复用，大部分 Skill 可用但 Hook 适配不完整。
- OpenCode：子集移植，13 agents / 31 commands / 37 skills。
- Codex：基础适配。
- Gemini：有限适配。
- Qwen / Trae / Zed：仅提供入口文件。

Claude Code 的 Hook 系统和上下文机制在其他平台没有等价实现。主要使用 Cursor 时，ECC 的 Hook 生命周期和 50 个 JS 脚本大多不可用。

### TDD Workflow Skill

ECC 的 TDD Workflow Skill 有几个实现细节：

- Runner 探测五级解析链：npm、pnpm、yarn、bun，按优先级逐一尝试，适配不同包管理器。
- Plan 接力加注入防御：把计划文件当不可信数据，拒绝 `curl | sh` 这类命令，要求人工审批执行。
- 证据链：plan task → test target → RED 证据 → GREEN 证据，git checkpoint 必须可从 HEAD 到达。这保证每个实现步骤都有对应的测试证据，且证据不被后续提交覆盖。

### 边界

ECC 的 Setup 需要处理 281 Skills。它们没有索引或分类，依赖 `tdd-*`、`security-*`、`review-*` 等前缀组织；前缀不能提供完整的发现机制。仓库同时存在 v1 和 v2 的命令与 Skill，旧命令的产物格式与新命令不兼容，且未提供迁移路径文档。

## GStack：按角色划分流程

GStack（[garrytan/gstack](https://github.com/garrytan/gstack)，84.3K stars）由 Garry Tan（Y Combinator CEO）维护。README 说明它包含 23 个专家角色技能、8 个 Power Tools、slash 命令及 5 个独立 CLI 工具。源码量约 220,000 行（不含测试），测试占约 45%。

### 角色与审查职责

GStack 将工程流程分配给不同角色，每个角色有独立的审查视角和阻断权。角色包括：

- CEO（`/plan-ceo-review`）：审查项目范围。4 种模式：扩展、选择性扩展、保持范围、缩减。该角色检查范围是否过度扩张或缩减。
- Eng Manager（`/plan-eng-review`）：审查架构、数据流、边界情况和测试策略，关注技术方案的完整性。
- Senior Designer（`/plan-design-review`）：0-10 评分加 AI Slop 检测。Slop 检测扫描模型生成文本中的套路化表达，例如滥用 "Additionally"、"It's worth noting"。
- DX Lead（`/plan-devex-review`）：20-45 个强制问题，3 种模式。DX Lead 从开发者体验角度审查，关注系统使用者的体验。
- QA Lead（`/qa`）：真机浏览器测试，后面单独讲。
- CSO（`/cso`）：安全审查，后面单独讲。
- Release Engineer（`/ship`）：交付发布，后面单独讲。

`/autoplan` 依次执行 CEO、design、eng 三项审查。GStack 不自动合并审查结果；CEO 建议扩展范围而 Eng Manager 建议缩减范围时，需要人工决定。

### Chromium Daemon 与浏览器 QA

GStack 的 `/qa` 在真实浏览器中运行测试。流程为：读取 git diff、识别受影响路由、在真实浏览器测试、修复 Bug、原子提交、回归测试。`/qa-only` 只报告，不修改代码。

底层是基于 Playwright 的持久浏览器 Daemon，浏览器实例可跨命令复用；README 给出的每条 browse 命令耗时约为 100ms。CDP 域技能可直接操作 Chrome DevTools Protocol，per-site 记忆存储各站点的操作偏好，raw CDP 功能允许直接发送已逐条登记在 allowlist 中的 CDP 命令。

`/pair-agent` 让任何 AI Agent 共享浏览器实例，这在需要多个 Agent 协作测试时有用。`/benchmark` 基线页面加载和 Core Web Vitals，用于性能回归检测。

GStack 的浏览器 QA 面向 Web 项目。其他五个套件不提供浏览器访问能力，测试主要依赖单元测试和静态检查。前端项目缺少浏览器测试时，样式和交互需要人工验证。

### /office-hours

`/office-hours` 用于反思需求，采用 YC 风格产品诊断。源码定义六个问题，并按项目阶段选用：无产品阶段使用 Q1-Q3，有用户时使用 Q2/Q4/Q5，有付费客户时使用 Q4-Q6。

1. Demand Reality：有什么证据说明用户需要它？“感兴趣”和 waitlist 报名不作为充分证据。
2. Status Quo：用户当前如何解决这个问题？现有变通方案消耗多少时间和金钱？
3. Desperate Specificity：描述一个最需要该产品的具体用户，包括职位、晋升或解雇条件及其主要顾虑。
4. Narrowest Wedge：本周可交付、且用户愿意付费的最小版本是什么？不需要登录和集成的版本包含什么？
5. Observation & Surprise：是否在不干预的情况下观察过用户使用过程？哪些行为与预期不符？
6. Future-Fit：三年后的外部环境发生变化时，产品的重要性会如何变化？

每个问题包含追问规则，通常要求继续追问第二、三次回答。源码为每个问题列出红旗清单，例如“我们拿到了 500 个 waitlist 报名”“医疗行业的企业客户”会被拒绝。输出为重新框定后的设计文档。Spec Kit 的 constitution 划定约束边界，`/office-hours` 则检查需求本身。

### 安全

`/cso` 实现两层安全：OWASP Top 10 检查加 STRIDE 威胁建模。STRIDE（Spoofing、Tampering、Repudiation、Information Disclosure、Denial of Service、Elevation of Privilege）逐类分析，输出威胁和对策。

Prompt Injection 防御包括多层机制：随浏览器打包的 22MB ML 分类器在本地扫描页面和工具输出；Claude Haiku 对整个对话形态进行二次检查；系统 prompt 中的随机 canary token 用于识别跨文本、工具参数、URL 和文件写入的外泄尝试。裁决层要求两个分类器一致才阻断，以降低 Stack Overflow 等包含指令文本页面的误报。可选开启 721MB 的 DeBERTa-v3 ensemble，采用三取二表决。脱敏引擎分为 HIGH、MEDIUM、LOW 三个级别，数据在离开系统时扫描。

`/careful` 对破坏性命令（rm、drop、truncate 等）发警告。`/freeze` 锁定指定文件禁止编辑。`/guard` 组合 careful 和 freeze。

### 交付流水线

GStack 的交付流程包括：

- `/ship`：sync → test → audit coverage → push → PR。全自动化。
- `/land-and-deploy`：合并 PR → 等 CI → 部署 → 验证生产健康。
- `/canary`：部署后监控循环，持续检查关键指标。
- `/document-release`：更新所有项目文档（CHANGELOG、README、API docs）。

GStack 覆盖从 PR 到生产监控的流程。其他套件不提供同等范围的发布与监控流程。

### 上下文管理

`/context-save` 和 `/context-restore` 显式保存和恢复上下文，可在恢复会话时跳过重复研究。GBrain 集成提供 PGLite 本地、Supabase 云端和 Remote MCP 三种持久化存储后端。跨会话决策使用 `decisions.jsonl` 事件溯源存储：每次决策追加记录，不执行 update-in-place，以保留历史。

配合 Conductor（并行运行多个 Claude Code 会话的工具）使用时，Garry Tan 表示通常运行 10-15 个并行 sprint，并将此视为当前实际限制。README 还说明 Conductor 会从各 workspace 的进程环境中移除 `ANTHROPIC_API_KEY` 和 `OPENAI_API_KEY`，需改用 `GSTACK_` 前缀的环境变量注入。

### 其他

`/codex` 调用 OpenAI Codex 获取第二意见。`/retro` 做团队周回顾，支持跨项目全局模式。`/design-shotgun` 生成 4-6 个 AI mockup 变体，`/design-html` 把 mockup 转成生产 HTML/CSS。

SKILL.md 从 `.tmpl` 模板生成，总量上限 160K token。两层测试：gate（安全确定性测试，必须通过）和 periodic（质量基准测试，允许波动）。Diff-based 测试选择：只跑和本次变更相关的测试。Hermetic E2E 隔离环境：每个 E2E 测试在独立环境中运行，不依赖外部服务。

### 边界

仅支持 Claude Code，非跨平台。对嵌入式和固件项目不适用：Chromium Daemon 和浏览器 QA 在没有 GUI 的环境里无法运行。

## 研发生命周期横向对比

### 需求与战略

OpenSpec 的 `/opsx:propose` 一次生成 proposal 和 specs，不进行前置约束检查。Spec Kit 先写 constitution，后续规格以此为约束；每个项目首次使用时需要编写该文件。GStack 的 `/office-hours` 通过六个诊断问题检查需求本身（问题清单见 GStack 一章）。GSD 的 `discuss-phase` 引导需求讨论并产出 REQUIREMENTS.md。ECC 有 67 个 Agent，不同 Agent 对需求阶段的处理不同，未定义统一的需求流程。

| 套件 | 需求产出 | 前置约束 | 反向挑战 |
|------|---------|---------|---------|
| OpenSpec | proposal + specs | 无 | 无 |
| Spec Kit | constitution + specs | constitution 先行 | 无 |
| Superpowers | plan（含行为规则） | 行为规则层 | 无 |
| GSD | REQUIREMENTS.md | 无（discuss-phase 自由讨论） | 无 |
| ECC | 取决于 Agent | 取决于 Agent | 无 |
| GStack | 重新框定后的设计文档 | 无 | /office-hours 6 个强制问题 |

在需求阶段，GStack 通过 `/office-hours` 检查需求本身；OpenSpec 和 Spec Kit 则从已有需求开始生成规格。

### 规划与设计

GSD 并行研究 STACK.md、FEATURES.md、ARCHITECTURE.md、PITFALLS.md 四个维度，再由综合器合并。GStack 的 CEO、Designer、Eng Manager 从不同视角审查方案。Superpowers 的 plan 包含 TDD checkbox，将测试策略写入计划。Spec Kit 的 `/speckit.clarify` 和 `/speckit.analyze` 提供需求澄清和代码库分析的质量门禁。

| 套件 | 研究方式 | 审查机制 | TDD 内建 |
|------|---------|---------|---------|
| OpenSpec | 单路探索 | 无 | 无 |
| Spec Kit | clarify + analyze 门禁 | constitution 合规检查 | 无 |
| Superpowers | brainstorming 自动触发 | 无 | plan 模板 RED-GREEN-REFACTOR checkbox |
| GSD | 4 路并行研究 Agent | plan-phase 审查 | 无 |
| ECC | 取决于 Agent | 取决于 Agent | TDD Workflow Skill |
| GStack | 单路 | 三权分立 | 无 |

### 任务拆解

GSD 的原子任务包含验证命令和完成标准。Spec Kit 的任务带依赖关系和并行标记，可以转为 GitHub Issues。OpenSpec 的 tasks 也有依赖关系，但没有并行标记和 Issue 转换。Superpowers 的任务拆解由 plan 模板驱动，粒度取决于模型。ECC 和 GStack 的任务拆解取决于选用的命令或角色。

GSD 和 Spec Kit 将依赖关系、并行标记和完成标准写入任务结构。其他四个套件的任务粒度和结构化程度取决于命令或模型。

### 编码实现

GStack 不提供 `/implement` 命令，实现由主会话在规划后完成。Garry Tan 认为实现环节由模型完成即可，套件在这一层增加流程可能干扰模型执行。其他五个套件提供不同形式的实现编排。

GSD 使用分波并行与文件锁。ECC 的多模型路由（multi-plan / multi-execute）需要额外运行时，当前不能直接运行。Superpowers 的子 Agent 并行依赖文件化产物和 ledger，开销低于 GSD 的 Fresh Context，但上下文隔离范围也较小。

### 审查

GStack 使用 `/plan-ceo-review`、`/plan-design-review`、`/plan-eng-review` 三个角色审查，可通过 `/codex` 获取第二意见，并集成 Greptile 进行 PR 分类。

ECC 使用 AgentShield 进行安全审查，但引擎位于外部包。GSD 在 verify-work 阶段产出 VERIFICATION.md 和 UAT.md，最终验证仍由人工完成。Superpowers 内置 code-review Skill，将审查作为标准流程步骤。

OpenSpec 和 Spec Kit 没有内建审查。OpenSpec 的 `/opsx:verify` 验证实现与 specs 的一致性，但这是规格合规检查，不是代码审查。

### QA 测试

GStack 的 `/qa` 在真实浏览器中运行测试。Superpowers 的 TDD 要求每个任务附带测试，但测试在终端中运行。ECC 提供 `/e2e` 和 `/test-coverage` 命令，E2E 的具体实现取决于项目测试框架。GSD 在 verify-work 阶段运行混合验证，验证内容由 VERIFICATION.md 声明。

OpenSpec 和 Spec Kit 没有 QA。

### 安全

ECC 使用 AgentShield，但引擎位于外部包，本仓库的验证范围有限。GStack 的 `/cso` 包含 OWASP Top 10、STRIDE 威胁建模、多层 Prompt Injection 防御和脱敏引擎，均可在本仓库验证。GSD 的 `/gsd-secure-phase` 由 security-auditor Agent 验证 plan 威胁模型中的缓解措施是否写入代码，并产出 SECURITY.md；它不接外部漏洞数据源。其余三个套件没有内建安全能力。

| 套件 | 安全能力 | 可验证范围 |
|------|---------|-----------|
| OpenSpec | 无 | 无 |
| Spec Kit | 无 | 无 |
| Superpowers | 无 | 无 |
| GSD | secure-phase 威胁模型缓解验证，产出 SECURITY.md | 本仓库可验证 |
| ECC | AgentShield + Prompt Defense + Hook 阻断 | 本仓库：Prompt Defense + Hook；AgentShield 在外部包 |
| GStack | /cso OWASP + STRIDE + 多层注入防御 + 脱敏 | 本仓库全部可验证 |

### 交付发布

GStack 的流程依次为 `/ship`（PR）、`/land-and-deploy`（合并、CI、部署、验证）、`/canary`（监控循环）、`/document-release`（文档更新）。GSD 的 `/gsd-ship` 自动创建 PR。OpenSpec 的 `/opsx:archive` 归档规格变更。Spec Kit 的 `/speckit.converge` 执行收敛验证但不涉及部署。Superpowers 的 `finishing-a-development-branch` 处理分支收尾但不部署。ECC 没有专用交付命令。

GStack 覆盖从 PR 到生产监控的流程。

### 上下文管理

GSD 的 Fresh Context 为每个 Agent 分配独立窗口，编排器只调度不执行，三档 Profile 按阶段切换，Context Monitor Hook 监控使用率。前序阶段的上下文不直接进入后续 Agent 的窗口。

GStack 的显式 save/restore 与 GBrain 持久化存储可保存跨会话信息，适用于需要多天推进的项目。ECC 通过 Hook 和 MCP 向量搜索（claude-mem）检索历史上下文，召回精度取决于 embedding 质量和查询措辞。Superpowers 用 Git worktree 隔离各任务的工作目录，但 worktree 仍共享 Git 历史。

OpenSpec 和 Spec Kit 按需加载上下文：只在执行命令时注入相关产物，不常驻 CLAUDE.md。Superpowers 和 ECC 的上下文常驻 CLAUDE.md：安装后持续占用窗口空间。

上下文管理策略的适用条件与代价如下：

| 策略 | 代表 | 优点 | 代价 |
|------|------|------|------|
| 常驻 CLAUDE.md | Superpowers / ECC | 模型始终知道流程约定 | 持续占用上下文 |
| 按需加载 | OpenSpec / Spec Kit | 不占用常驻空间 | 模型可能忘记约定 |
| 阶段化 Fresh Context | GSD | 阶段间上下文隔离彻底 | 编排开销大，多 Agent 协调复杂 |
| 显式 save/restore | GStack | 跨会话记忆，人可控 | 需要手动触发保存/恢复 |

## 实现机制对比

### 编排模型

六个套件都使用 Markdown 提示编排：流程步骤写在 Markdown 文件中，由模型读取并执行。各套件的运行时支持不同。

GSD 提供 SDK：299 个 TypeScript 文件，包含类型定义、工具函数和运行时支持。GStack 有 5 个独立 CLI 工具和 Playwright 浏览器 Daemon，部分编排逻辑位于 CLI 而非 Markdown。ECC 有 50 个 JS Hook 脚本，编排发生在 Hook 事件触发链中。

OpenSpec 和 Spec Kit 仅通过 Markdown 编排：命令文件定义步骤，由模型执行，不提供运行时校验。Superpowers 也通过 Markdown 编排，并使用文件化产物和 ledger 管理状态。

纯 Markdown 编排要求模型正确理解步骤描述。例如将“先写测试”理解为“先写实现”时，执行路径会偏离，解释器或类型系统无法捕捉这类错误。GSD 的 SDK 和 GStack 的 CLI 可通过类型定义和运行时校验命令是否存在、参数是否合法。

### 上下文管理策略

OpenSpec 和 Spec Kit 通过独立产物文件隔离不同特性。Superpowers 用 Git worktree 在不同目录执行不同任务。GSD 的 Fresh Context 为不同 Agent 分配不同窗口。GStack 通过显式 save/restore 和持久化存储，由用户决定何时保存和恢复。

### Agent 定义方式

Superpowers 和 GSD 用 Task subagent：任务粒度的临时 Agent，任务完成后销毁。GStack 用独立 SKILL.md 角色：每个角色有独立的提示和行为定义，跨任务复用。ECC 有 67 个纯 Markdown Agent，每个 Agent 是一个 Markdown 文件定义行为，没有运行时。OpenSpec 和 Spec Kit 没有独立的 Agent 概念，命令就是主会话的不同模式。

ECC 的 67 个 Agent 均由 Markdown 定义，不提供运行时保证。GStack 的角色同样由 SKILL.md 定义，但配合 CLI 工具和浏览器 Daemon。GSD 的 33 个 Agent 配合 SDK 和 Fresh Context 使用。

### 安装与分发

| 套件 | 安装方式 | 安装器大小 | 侵入性 |
|------|---------|-----------|--------|
| OpenSpec | npm CLI | 小 | 低：无 CLAUDE.md，无 AGENTS.md 内容 |
| Spec Kit | uv CLI（Python） | 中 | 低：CLI 不管 CLAUDE.md，上下文可选 |
| Superpowers | Claude Code 插件市场 / npx skills add | 小 | 中：常驻 CLAUDE.md |
| GSD | git clone + install.js | 11,376 行安装器 | 高：.planning/ 目录 + 14 Hook + CLAUDE.md |
| ECC | npm 包 + 插件 | 大 | 高：37KB hooks.json + CLAUDE.md + 全局配置 |
| GStack | bun install | 中 | 中：CLI 工具 + 可选 GBrain 后端 |

GSD 的安装器有 11,376 行。安装过程涉及创建目录结构、注册 Hook、写入配置文件和初始化 Git hooks。安装中断或失败时，未清理的配置可能导致项目状态不一致。

### 测试体系

GSD 有 549 个测试文件，约 132,441 行测试代码。ECC 有 197 个测试文件，并包含 docs-vs-code 一致性测试，用于验证文档描述和代码实现一致。GStack 有 gate（安全确定性）和 periodic（质量基准）两层测试，以及 Diff-based 测试选择和 Hermetic E2E 隔离。Superpowers 和 OpenSpec 的仓库中没有测试。Spec Kit 的测试主要在 CLI 层，不在 Skill 产物层。

## 按场景选型

### 个人项目原型开发

可组合 Superpowers 与 GStack 的 `/qa`、`/cso`。Superpowers 提供 TDD 和审查流程，GStack 的浏览器 QA 与安全审查可按需调用，无需使用完整角色流程。Superpowers 约 4,700 行 Markdown，GStack 可按需调用。

前提：你的项目是 Web 项目（GStack 的 `/qa` 依赖浏览器）。非 Web 项目把 GStack 换成 ECC 的 `/test-coverage`。

### 团队新项目

可组合 Spec Kit、GSD 与 GStack 的 `/review`、`/qa`。Spec Kit 的 constitution 在项目启动时定义约束边界，GSD 执行与验证，GStack 的审查和 QA 按需调用。该组合覆盖需求到验证，但需要配置 Spec Kit 的 Python CLI、GSD 的 11,376 行安装器和 GStack 的 bun install。

前提：团队愿意接受 Spec Kit 的 30min-3hrs per feature 开销，且项目规模大到能摊薄这个固定成本。

### 现有项目维护

可组合 OpenSpec、ECC 与 GStack 的 `/canary`。OpenSpec 不要求 constitution，可直接使用 `/opsx:propose`。ECC 的 Hook 生命周期（50 个 JS 脚本、7 类事件）和 AgentShield 提供安全能力。GStack 的 `/canary` 用于部署后监控。

前提：ECC 的 37KB 配置文件和 Hook 复制粘贴问题在多人维护时需要专人管理。GStack 的 `/canary` 需要部署基础设施支持。

### 全栈项目重构

可组合 GSD 与 ECC 的 multi-plan / multi-execute。GSD 的 Fresh Context 和分波执行隔离长链路重构的上下文，4 路并行研究 Agent 可用于大范围重构的前期研究。ECC 的多模型路由可为架构决策提供多个视角。

前提：multi-plan / multi-execute 当前开箱无法运行，需要自行安装 ccg-workflow 运行时和角色文件。GSD 的学习曲线（67 命令）需要投入时间。

### SaaS 产品发布

可组合 GStack 与 Spec Kit constitution。GStack 覆盖 PR、CI、canary 和监控，Spec Kit 的 constitution 在迭代中作为约束边界。该组合适用于频繁发布，且每次发布均需安全审查和性能监控的场景。

前提：仅限 Web 项目。GStack 的交付链路假设了 CI/CD 和部署基础设施。

### 选择条件

- 需要安全合规时，可考虑 ECC 或 GStack。ECC 的 AgentShield 引擎位于外部包；GStack 的 `/cso` 可在本仓库验证。
- 需要浏览器测试时，可使用 GStack。
- 需要长链路上下文隔离时，可使用 GSD 的 Fresh Context。
- 需要统一团队规格时，Spec Kit 的 constitution 可定义约束。
- 需要较低开销时，可考虑 OpenSpec 或 Superpowers。OpenSpec 不常驻上下文，Superpowers 约 4,700 行 Markdown 且零依赖。

## 选择的边界

套件覆盖的流程应对应项目当前的问题。GStack 的角色分离和浏览器 QA 面向 Web SaaS 项目，用于 CLI 工具开发时部分能力不会使用。GSD 的 Fresh Context 和分波执行面向长链路项目，用于两行配置修改时流程开销较高。OpenSpec 适用于需求明确的小项目，但不覆盖安全审查和部署监控。

GSD 的 67 个命令和 14 个 Hook 若与项目约定冲突，排查时还需要分析框架内部编排逻辑及业务代码。

我在个人项目中使用 Superpowers 的 TDD 约束和 OpenSpec 的规格管理；Web 项目额外使用 GStack 的 `/qa` 和 `/cso`，不采用任一套件的完整流程。这种组合减少了常驻流程和配置，但不覆盖端到端流程。若只使用一套，应选择覆盖当前主要问题、附加流程较少的套件。
