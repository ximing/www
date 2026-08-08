---
title: Claude Code 六大 Skill 套件源码级对比：谁在解决什么问题，谁在制造新问题
date: 2026-04-27 22:14
series: ai-coding
tags: ["AICoding", "源码分析"]
description: 通读 OpenSpec、Spec Kit、Superpowers、GSD、ECC、GStack 六个仓库的源码，按研发生命周期逐一对比它们的命令、产物、核心机制、优缺点与适用场景，给出基于事实的选型建议。
published: true
---

裸用 Claude Code 写代码，三个问题反复出现。第一，上下文腐烂：会话越长，窗口里堆积的过时信息越多，模型基于错误前提做决策的比例随轮次上升。第二，需求跑偏：你描述的意图和模型理解的意图不一致，产出越复杂偏差越大，而且往往要等产出落地才发现。第三，反馈回路缺失：没有类型检查、没有测试、没有浏览器，模型闭眼写代码，错了也不知道。

Skill 套件就是为对付这三个问题而生的。半年下来，这个方向已经分化成两类。规范层只管"写清楚要做什么"，把需求和设计物化为 Markdown 产物，实现交给主会话。OpenSpec 和 Spec Kit 属于这类。执行层接管从规划到提交的完整流程，内置任务拆分、测试、审查、发布。Superpowers、GSD、ECC、GStack 属于这类。两类之间没有绝对边界，Spec Kit 的 `/implement` 涉及执行，GStack 的 `/office-hours` 涉及需求反思，但核心倾向可以这么分。

选错套件的代价不只是多装一个工具。执行层套件会把自己的流程嵌进你的开发习惯：产物目录、命名约定、git 分支策略、CLAUDE.md 内容。一旦依赖这些结构，退出时需要清理的不只是配置文件，还有散落在仓库各处的产物。规范层套件的侵入性低，但也意味着它不管执行，你仍然需要在实现环节解决上下文腐烂和反馈缺失。

这六个套件各有各的说法，README 和社区转述里有不少过时或不准确的信息。我的做法是 clone 仓库读源码，逐个核实命令表、产物结构、编排逻辑和边界条件，而不是照着 README 写综述。下面按套件逐章展开，再按研发生命周期做横向对比，最后给选型建议。

文中数字对应的版本：GSD v1.50.0-canary.0、ECC v2.0.0。这些项目迭代很快，旧版本的数据与当前版本可能差距较大。

## OpenSpec：Spec-Driven 的极简实现

OpenSpec（[Fission-AI/OpenSpec](https://github.com/Fission-AI/OpenSpec)，43.1K stars）是一个 Node.js CLI 工具，约 8,000 行 TypeScript 和 Markdown。它的功能范围很窄：把需求和设计写成规格文档，按规格实现，实现完归档。没有测试编排、没有审查流程、没有部署。

### 命令与产物

核心命令六个：

- `/opsx:explore`：思考，不产出文件。用于理解问题空间，把推理过程留在对话里。
- `/opsx:propose`：一键生成四个产物：proposal（要做什么）、specs（功能规格）、design（技术设计）、tasks（任务列表）。这四个文件之间存在 DAG 依赖：proposal 先行，specs 和 design 依赖 proposal，tasks 依赖 specs 和 design。
- `/opsx:apply`：按 tasks 逐条实现。
- `/opsx:update`：修改已有规划产物并保持互相一致（只动产物，不动代码）。
- `/opsx:sync`：把 delta specs 合并回主 specs。archive 时会提示，也可以单独跑。
- `/opsx:archive`：归档。把这次变更的 specs 合并回主 specs，清理临时产物。

扩展命令六个（需切换 profile 启用）：`/opsx:new`（只建 change 脚手架，产物留待后续生成）、`/opsx:continue`（按依赖图创建下一个产物）、`/opsx:ff`（fast-forward，一次创建全部规划产物）、`/opsx:verify`（验证实现与 specs 的一致性）、`/opsx:bulk-archive`（批量归档，自动检测并处理 change 间的 spec 冲突）、`/opsx:onboard`（引导式教程，带你走一遍完整工作流）。另有旧版命令 `/openspec:proposal`、`/openspec:apply`、`/openspec:archive`，当前仍可用但不在新文档推荐中。

任务状态有三个：BLOCKED（依赖未满足）、READY（可执行）、DONE（已完成）。状态流转靠依赖图驱动，不靠线性阶段号。

### Delta Spec 机制

这是 OpenSpec 在规格管理上的核心设计。一次变更不需要重写整个 specs，只写差异：ADDED（新增哪些规格条目）、MODIFIED（改了哪些条目的哪些字段）、REMOVED（删除了哪些条目）。归档时 Delta Spec 自动合并回主 specs。好处是变更历史的粒度细，每次改动能追溯到"改了什么"。多个 change 并行时可能改到同一份 spec，`/opsx:bulk-archive` 会检测跨 change 的冲突，并按代码库的实际实现状态来裁决合并顺序。超出这个范围的语义冲突仍需要人判断。

### Schema 可定制

默认 schema 定义了 proposal、specs、design、tasks 四个产物的结构和字段。如果你的项目有领域特定的规格结构（比如 API 合约、数据库 schema），可以用 `openspec schema fork` 基于内置 schema 分叉出自己的工作流，放在项目的 `openspec/schemas/` 目录下随代码一起版本化。

### 跨宿主适配

同一份 Markdown 产物要能在不同 AI 工具里被调用。OpenSpec 的做法是同一命令名、不同拼写：Claude Code 用 `/opsx:propose`，Cursor 和 GitHub Copilot 用 `/opsx-propose`，Amazon Q 用 `@opsx-propose`，Codex 用 `$openspec-propose`。`openspec init` 会按你选的工具打印对应的调用形式。

### 设计原则与边界

OpenSpec 的核心原则是 "Enablers, not gates"。依赖关系只为 AI 提供上下文，不强制线性阶段。你可以跳过 explore 直接 propose，也可以只生成 proposal 不生成 tasks。这降低了流程摩擦，但没有质量门禁：specs 写得很空不会被阻断，tasks 漏掉验证标准也不会报错。

边界明确：没有 Review、没有 QA、没有安全检查、没有部署。它只管"写清楚要做什么"和"按任务实现"这两段。

仓库里没有 CLAUDE.md，AGENTS.md 是空文件。安装后不会往项目上下文注入常驻信息，侵入性低。代价是模型不会自动知道 OpenSpec 的存在，需要手动触发命令。

## Spec Kit：六阶段管道的治理工具

Spec Kit（[github/spec-kit](https://github.com/github/spec-kit)，91K stars）是 GitHub 官方出的 Python CLI 工具（`specify-cli`），约 25,000 行 Python 和 Markdown。它和 OpenSpec 定位相似，但体量大得多：OpenSpec 六个核心命令走完一个特性，Spec Kit 有六个阶段，每个阶段自带质量门禁。

### SDD 六阶段

核心命令七个，按默认顺序串联：

1. `/speckit.constitution`：生成项目级宪法文件 `constitution.md`。这个文件定义项目的约束边界：技术栈、架构决策、禁止事项。后续命令执行时会加载它作为治理约束。
2. `/speckit.specify`：定义要构建什么，即需求和用户故事。执行时如果 constitution.md 存在会被加载，约束规格的方向。
3. `/speckit.plan`：生成技术实现计划。
4. `/speckit.tasks`：生成任务列表，带依赖关系和并行标记。
5. `/speckit.implement`：按任务列表实现。
6. `/speckit.converge`：对照 spec/plan/tasks 评估代码库现状，把差距追加为新任务。

第七个核心命令 `/speckit.taskstoissues` 把任务列表转成 GitHub Issues。可选命令三个：`/speckit.clarify`（在 plan 之前澄清规格中欠定义的部分）、`/speckit.analyze`（跨产物一致性与覆盖分析，在 tasks 之后、implement 之前运行）、`/speckit.checklist`（生成自定义质量检查清单，验证需求的完整性、清晰度和一致性，官方文档把这个比作"给英文写单元测试"）。

constitution 先行是 Spec Kit 和 OpenSpec 的根本区别。OpenSpec 的 proposal 没有项目级约束的前置检查，你可以写一个和项目架构矛盾的 proposal 然后直接往下走。Spec Kit 的 constitution 在第一步就把边界写下来，后续阶段的 prompt 会把它作为约束加载。注意这个约束只作用于 prompt 层：模型被告知要遵守宪法，但越界时没有代码级校验会硬性阻断。

### 三层定制体系

Spec Kit 的定制分为三层：Extension（扩展默认 schema）、Preset（预设配置组合）、Override（覆盖默认行为）。优先级递减：Override 覆盖 Preset，Preset 覆盖 Extension，Extension 扩展默认。这个体系比 OpenSpec 的"分叉 schema"更结构化。学习成本更高：需要理解三层交互规则，Override 写错可能把 Preset 的合理配置覆盖掉。

### Bundle 系统与集成架构

Bundle 是 Spec Kit 的扩展包机制：可搜索、安装、更新、移除社区扩展包。每个 Bundle 是一组 schema 定制、命令扩展和上下文文件的集合。`IntegrationManifest` 用 SHA-256 追踪已安装文件的哈希值。卸载时对比哈希：用户编辑过的文件（哈希不匹配）不删除，避免误删用户修改。

每个核心脚本有三套实现：POSIX shell、PowerShell、Python。三套实现功能一致，适配不同开发环境。Spec Kit 支持 30+ AI 工具集成，每个集成是独立子包，放在 `src/specify_cli/integrations/<key>/` 下。

上下文文件由 `agent-context` 扩展管理，CLI 本身不管 CLAUDE.md。这意味着上下文注入是可选的，不是安装后自动发生的。

### 边界

开销是 Spec Kit 最大的问题。每个特性走完六阶段管道需要 30 分钟到 3 小时，这个时间包含模型生成产物、人审查产物、模型修改产物。对于修一个 CSS typo 的场景，这个开销不可接受。Spec Kit 自己没有给出跳过阶段或缩短管道的机制，你需要手动决定哪些特性走完整流程、哪些不走。

和 OpenSpec 一样，没有 QA、没有安全检查、没有部署。仓库里没有 CLAUDE.md，AGENTS.md 有 603 行，定义了集成架构和扩展点，但不包含运行时行为指令。

## Superpowers：TDD 内建的最短路径

Superpowers（[obra/superpowers](https://github.com/obra/superpowers)，168.7K stars）由 Jesse Vincent 和 Prime Radiant 团队维护。14 个 Skill，约 1,500 行胶水脚本，零第三方依赖。源码总量约 4,700 行 Markdown 和 shell。

先纠正一个说法：社区文章常把 Superpowers 描述为"brainstorm / write-plan / execute-plan 三个命令"。仓库里不存在 `commands/` 目录，这三个不是用户触发的命令，是自动触发的 Skill：`brainstorming`、`writing-plans`、`executing-plans`。作为 model-invoked 技能，模型判断任务阶段后自动激活，不需要人敲斜杠。

### TDD 三层嵌套

Superpowers 把测试驱动开发嵌进了三层：

1. 行为规则层：每个 Skill 的 SKILL.md 里声明该 Skill 的行为规则，行为规则本身是可测试的约束。比如 `implementing` Skill 声明"每个实现步骤必须报告 TDD Evidence"。
2. Plan 模板层：plan 模板把 RED-GREEN-REFACTOR 物化为 checkbox。模型写计划时必须包含"写测试、跑测试确认红、写实现、跑测试确认绿、重构"这些步骤，不是口头承诺 TDD 而是把循环写进计划文件。
3. SDD 执行层：`implementing` Skill 要求 implementer 每完成一个任务报告 TDD Evidence，包括测试文件路径、红绿状态、覆盖率变化。

三层嵌套的效果是：模型想跳过测试时，计划文件的 checkbox 和证据报告要求会让它至少走完 RED-GREEN。但这是 prompt 层约束，不是 CI 门禁。模型在长会话中仍可能绕过这些要求，Superpowers 没有运行时强制机制。

### SDD 子 Agent 编排

Superpowers 的 SDD（Spec-Driven Development）模式使用子 Agent 编排实现任务。核心设计点：

- 全部产物文件化。子 Agent 的产出不是留在对话上下文里，而是写入文件。这保护了 controller 的上下文窗口不被子 Agent 的中间过程占满。
- Plan-scoped workspace：每个子 Agent 只能访问当前 plan 声明的文件范围，防止越界修改。
- Ledger 抗 compaction：子 Agent 的关键决策写入 ledger 文件，即使上下文被压缩（compaction），决策记录仍然可以从文件中恢复。
- 5 轮熔断修复循环：如果子 Agent 连续 5 轮未能通过测试，流程熔断，把控制权交还给人。

### 内置审查流程

Superpowers 有两个审查相关的 Skill：`requesting-code-review` 和 `receiving-code-review`。前者发起审查请求，后者处理审查反馈。`finishing-a-development-branch` 处理分支收尾，包括合并前检查和清理。

审查流程是内置的但不是强制的。你可以在 plan 里跳过 code-review 步骤，Superpowers 不会阻断。

### 跨宿主与 Git Worktree

14 个 Skill 共 3,185 行 SKILL.md，支持 11 个宿主平台（Claude Code、Cursor、Codex、OpenCode 等）。Superpowers 还支持 Git worktree：每个任务在独立的 worktree 里执行，避免并行任务之间的文件冲突。

### 边界

没有安全审计、没有浏览器 QA、没有部署发布。流程里有人工确认点：README 描述的流程是，设计稿分段展示给你签字确认，你说 "go" 之后才启动子 Agent 执行。但这些确认和 TDD 约束一样是 prompt 层约定，模型在上下文压力下仍可能跳过确认直接执行。plan 阶段的架构错误会在执行阶段被放大，改已经写进去的代码比改 plan 代价高得多。

## GSD：长链路项目的工程化推进

GSD（[gsd-build/get-shit-done](https://github.com/gsd-build/get-shit-done)，57.6K stars，v1.50.0-canary.0）是六个套件中测试规模最大的：549 个测试文件约 132,441 行测试代码，加上约 171,865 行源代码，总量约 30 万行。支持 15 种运行时。

先纠正一个过时数据：社区文章常引用"29 Skills + 12 Agents + 20+ 命令"，这是早期版本。当前版本没有独立的 Skills 概念，实际是 67 个命令和 33 个 Agent。

### 命令路由：两阶段命名空间

67 个命令被组织在 6 个命名空间下，每个命名空间是一个 Meta-Skill 路由器：`/gsd-workflow`、`/gsd-project`、`/gsd-quality`、`/gsd-context`、`/gsd-manage`、`/gsd-ideate`。模型先从 6 个命名空间选择（约 120 tokens），再路由到具体子命令。这个两阶段路由取代了早期版本 86 个 skill 的扁平列举（约 2,150 tokens），每次调用节省约 2K tokens；调用频率越高，节省累积越多。

### Fresh Context 机制

GSD 上下文管理的核心设计是 Fresh Context：每个 Agent 获得独立的 200K 或 1M 上下文窗口，不共享。编排器只调度不执行，编排逻辑写在 workflow `.md` 文件里，由模型读取并按步骤派发。

上下文 Profile 分三档：

- `dev.md`：简洁，行动导向，适合实现阶段。
- `research.md`：冗长，探索性，适合研究阶段。
- `review.md`：关键细节导向，适合审查阶段。

Profile 的选择不是自动的，由 workflow 在派发 Agent 时指定。

Context Monitor Hook 持续监控上下文剩余空间：低于 35% 发 WARNING，低于 25% 发 CRITICAL。这提醒人手动干预（压缩上下文或拆分任务），但不自动执行任何动作。

使用 1M 上下文窗口的模型时，派发给 Agent 的上下文会自适应扩充：Executor Agent 获得前序 wave 的 SUMMARY.md 加当前阶段的 CONTEXT.md 和 RESEARCH.md；Verifier Agent 获得所有 PLAN.md、SUMMARY.md、CONTEXT.md、REQUIREMENTS.md。Verifier 的上下文比 Executor 更全面，因为验证需要看到全局约束。

### 阶段化工作流

GSD 的主流程是六个阶段：

`new-project` → `discuss-phase` → `plan-phase` → `execute-phase` → `verify-work` → `ship`

辅助阶段五个：`ui-phase`、`spec-phase`、`mvp-phase`、`validate-phase`、`secure-phase`。辅助阶段不强制串入主流程，可以按需插入。

并行研究 Agent 是 GSD 在规划阶段的特色。plan-phase 启动 4 路并行研究：STACK.md（技术栈评估）、FEATURES.md（特性清单）、ARCHITECTURE.md（架构方案）、PITFALLS.md（已知陷阱）。综合器按顺序合并四路结果。工具优先级：Context7 MCP > CLI 回退 > WebSearch > 训练数据。这个优先级保证能查到最新文档时优先查，而不是依赖可能过时的训练数据。

### 波执行模型

execute-phase 的任务执行采用波（wave）模型：

1. 自动依赖分析：根据任务间文件依赖关系构建依赖图。
2. 波分组：无依赖的任务分到同一波，有依赖的任务按依赖顺序分到不同波。
3. 波内并行：同一波内的任务由不同 Agent 并行执行。
4. 波间顺序：前一波全部完成后才启动下一波。

STATE.md 文件锁保证波间顺序的实现：用 `O_EXCL` 原子创建文件锁，10 秒过期，获取失败时抖动自旋等待。这个实现比简单的文件存在检查可靠：过期机制防止崩溃后锁残留，抖动自旋防止多 Agent 同时重试导致的惊群。

`--no-verify` 提交 + wave 后统一 pre-commit：每个任务提交时跳过 pre-commit（加快速度），整波完成后统一跑 pre-commit 检查。这个权衡牺牲了单任务的提交安全性，换取并行执行速度。

### Forensics 诊断

GSD 内置 6 种异常模式的诊断工具 Forensics：

1. Stuck Loop：Agent 在同一任务上反复失败。
2. Missing Artifact：预期产物未生成。
3. Partial-plan Drift：部分计划与实际进展偏离。
4. Abandoned Work：已开始但未完成的任务。
5. Crash：Agent 非正常终止。
6. Scope Drift：任务范围超出计划。

Forensics 只读调查，不修改任何文件。输出数据脱敏（移除用户名、路径等），证据归因到具体的 Agent 和任务。

### 产物体系

GSD 的产物全部放在 `.planning/` 目录下：PROJECT.md、REQUIREMENTS.md、ROADMAP.md、STATE.md、CONTEXT.md、RESEARCH.md、PLAN.md、SUMMARY.md、VERIFICATION.md、UAT.md、UI-SPEC.md。产物数量多，覆盖了从需求到验证的完整链路，但也意味着产物维护成本高：如果项目方向调整，需要同步更新多个文件。

### 14 个 Hook

GSD 注册了 14 个 Hook，覆盖 statusLine、PostToolUse、SessionStart、PreToolUse、PostToolUseFailure 等事件。三个防护 Hook 值得单独说：

- `gsd-prompt-guard.js`：扫描 `.planning/` 目录写入的内容，检测注入模式（比如在 REQUIREMENTS.md 里嵌入指令让模型执行非计划操作）。
- `gsd-workflow-guard.js`：检测在 GSD 上下文之外的编辑，防止人手动改文件绕过流程。
- `gsd-read-guard.js`：阻止编辑未读文件，强制模型先读后改。

这些 Hook 是 advisory 的，不是阻断性的。它们会发出警告，但不会阻止操作继续。这意味着在模型坚持错误行为时，Hook 的保护有限。

### 边界

学习曲线陡峭：67 个命令，安装器 11,376 行。纯 Markdown 编排没有运行时保证：如果模型读错了 workflow `.md` 里的步骤描述，执行路径就偏离了，没有类型系统或解释器来捕捉。Hook 为 advisory 非阻断，前面说过。安全方面只有 `/gsd-secure-phase`（验证 plan 威胁模型中的缓解措施是否落到代码里，产出 SECURITY.md），不接外部漏洞数据源。没有浏览器 QA。

## ECC：企业级 Agent Harness

ECC（[affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code)，167.7K stars，v2.0.0）是六个套件中文件数最多的：3,407 个文件，约 159,000 行可执行代码加约 446,000 行 Markdown，文档体量约为可执行代码的 3 倍。

纠正一个过时数据：社区文章常引用"48 Agents + 183 Skills + 79 命令"，这是旧版本。当前版本是 67 Agents、281 Skills、94 命令。

### AgentShield 纠偏

README 宣称 AgentShield 引擎有 102 条规则和 1282 条测试。但 AgentShield 引擎是一个独立 npm 包 `ecc-agentshield`，不在这个仓库里。本仓库只有两个薄包装：`commands/security-scan.md` 和 `skills/security-scan/SKILL.md`，它们调用 `npx ecc-agentshield scan`。102 条规则和 1282 条测试无法从本仓库验证。

本仓库自身的安全实现是另一套：每个 agent 内嵌 Prompt Defense Baseline（一组对抗注入的 prompt 约束），Hook 层有三个阻断脚本：`config-protection.js`（保护配置文件不被修改）、`block-no-verify.js`（阻止 `--no-verify` 提交）、`governance-capture.js`（捕获治理事件）。加上 `tdd-workflow` skill 的不可信输入处理（拒绝 `curl | sh`，要求人工审批）。这些是可验证的、存在于本仓库的安全措施。

### 多模型路由纠偏

ECC 的 `multi-plan` 和 `multi-execute` 命令在设计文档里描述了一个多模型协作方案：Code Sovereignty（外部模型零文件写权限，只输出 Unified Diff，Claude 作为唯一写盘者）、信任规则（后端听 Codex、前端听 Gemini）、SESSION_ID 接力。设计本身有亮点，特别是 Code Sovereignty：让多个模型产出建议但不给它们写权限，避免了多模型同时修改同一文件导致的冲突。

但当前版本开箱无法运行。`multi-plan` 和 `multi-execute` 依赖未随仓库分发的 `ccg-workflow` 运行时和 `~/.claude/.ccg/` 角色文件。仓库里没有这些文件，也没有安装脚本。这是一个常见的开源项目问题：README 描述了完整功能，但核心运行时缺失。

### Hook 生命周期

Hook 是 ECC 实现最扎实的部分。7 类事件：SessionStart、SessionEnd、PreToolUse、PostToolUse、PostToolUseFailure、PreCompact、Stop。50 个 JS 脚本，分发器模式：每个事件触发时，分发器遍历该事件注册的所有脚本并执行。

两个 Hook 值得展开：

- `stop-format-typecheck.js`：会话结束时批量执行 format 和 typecheck，270 秒总预算。如果 typecheck 超时，输出剩余错误让用户手动处理。这个 Hook 的价值在于每次会话结束都做一遍静态检查，防止提交未验证的代码。
- `session-start.js`：会话开始时加载上次会话摘要和 instincts（置信度 0.7 阈值、最多 6 条、8000 字符上限）。Instincts 是从历史会话中提取的高频行为模式，用来加速新会话的冷启动。

缺陷：`hooks.json` 的引导代码在 30+ 个文件里复制粘贴，整个配置文件达 37KB。维护成本高，修改 Hook 配置需要同步多个文件。

### 跨平台实际深度

ECC 宣称支持 7+ 平台，但实际深度差异很大：

- Claude Code：完整功能，所有 Agent、Skill、Hook 可用。
- Cursor：格式转换加委托复用，大部分 Skill 可用但 Hook 适配不完整。
- OpenCode：子集移植，13 agents / 31 commands / 37 skills。
- Codex：基础适配。
- Gemini：有限适配。
- Qwen / Trae / Zed：近乎占位，只有入口文件。

这个梯度是正常的：Claude Code 的 Hook 系统和上下文机制是独有的，其他平台没有等价物。但在选型时要知道：如果你主要用 Cursor，ECC 的核心优势（Hook 生命周期、50 个 JS 脚本）在你的环境里大部分不可用。

### TDD Workflow Skill

ECC 的 TDD Workflow Skill 有几个实现细节：

- Runner 探测五级解析链：npm、pnpm、yarn、bun，按优先级逐一尝试，适配不同包管理器。
- Plan 接力加注入防御：把计划文件当不可信数据，拒绝 `curl | sh` 这类命令，要求人工审批执行。
- 证据链：plan task → test target → RED 证据 → GREEN 证据，git checkpoint 必须可从 HEAD 到达。这保证每个实现步骤都有对应的测试证据，且证据不被后续提交覆盖。

### 边界

规模庞大，Setup 复杂。281 Skills 没有索引或分类，靠前缀约定组织（比如 `tdd-*`、`security-*`、`review-*`）。前缀约定在数量少时有效，到了 281 个 Skill 的规模，谁也记不住所有前缀。v1 和 v2 并存：仓库里同时存在 v1 和 v2 的命令和 Skill，旧命令的产物格式和新命令不兼容，迁移路径没有文档。

## GStack：角色分离的虚拟工程团队

GStack（[garrytan/gstack](https://github.com/garrytan/gstack)，84.3K stars）由 Garry Tan（Y Combinator CEO）维护。README 的说法是 23 个专家角色技能加 8 个 Power Tools，全部是 slash 命令，另有 5 个独立 CLI 工具。源码量约 220,000 行（不含测试），测试占约 45%，是六个套件中有效代码量最大的。

### 角色分离机制

GStack 的核心设计是把工程流程分配给不同角色，每个角色有独立的审查视角和阻断权。6 加 1 个核心角色：

- CEO（`/plan-ceo-review`）：挑战项目范围。4 种模式：扩展、选择性扩展、保持范围、缩减。CEO 的职责是防止项目范围膨胀或萎缩到不合理的程度。
- Eng Manager（`/plan-eng-review`）：锁定架构、数据流、边界情况、测试策略。Eng Manager 关注的是技术方案的完整性，不是需求合理性。
- Senior Designer（`/plan-design-review`）：0-10 评分加 AI Slop 检测。Slop 检测是针对模型生成文本的套路化表达（比如滥用"Additionally"、"It's worth noting"）的扫描，这个检测在其他套件里没有等价物。
- DX Lead（`/plan-devex-review`）：20-45 个强制问题，3 种模式。DX Lead 从开发者体验角度审查，关注的是使用这个系统的开发者的感受。
- QA Lead（`/qa`）：真机浏览器测试，后面单独讲。
- CSO（`/cso`）：安全审查，后面单独讲。
- Release Engineer（`/ship`）：交付发布，后面单独讲。

`/autoplan` 一键串联 CEO → design → eng 三权审查。三权审查的结果需要合并，GStack 没有自动合并机制：如果 CEO 说扩展范围、Eng Manager 说缩减范围，你需要自己决定听谁的。

### Chromium Daemon 加 QA 闭环

GStack 的 `/qa` 是六个套件中唯一在真实浏览器里跑测试的。流程：读 git diff → 识别受影响路由 → 开真机浏览器测试 → 修 Bug → 原子提交 → 回归测试。`/qa-only` 只报告不改代码。

底层是基于 Playwright 的持久浏览器 Daemon：浏览器实例跨命令复用，README 给出的数字是每条 browse 命令约 100ms。CDP 域技能可以直接操作 Chrome DevTools Protocol，per-site 记忆存储每个站点的操作偏好，raw CDP 逃逸舱允许直接发送 CDP 命令（方法需在 allowlist 里逐条登记）。

`/pair-agent` 让任何 AI Agent 共享浏览器实例，这在需要多个 Agent 协作测试时有用。`/benchmark` 基线页面加载和 Core Web Vitals，用于性能回归检测。

这个 QA 闭环是 GStack 对 Web 项目的核心价值。其他五个套件都没有浏览器访问能力，它们的测试策略局限于单元测试和静态检查。对于前端项目，没有浏览器测试意味着样式和交互的正确性完全靠人看。

### /office-hours

`/office-hours` 是 GStack 的需求反思机制，YC 风格产品诊断。源码里定义了六个问题，按项目阶段选用（无产品阶段问 Q1-Q3，有用户问 Q2/Q4/Q5，有付费客户问 Q4-Q6）：

1. Demand Reality：有什么证据说明有人真的需要它？不算"感兴趣"、不算 waitlist 报名，要的是"明天消失会有人真的难受"。
2. Status Quo：用户现在怎么解决这个问题，哪怕方式很笨？这个变通方案花了他们多少时间和钱。
3. Desperate Specificity：说出一个最需要它的具体的人。职位是什么，什么让他升职，什么让他被解雇，什么让他睡不着。
4. Narrowest Wedge：这周就能交付、有人愿意真金白银付钱的最小版本是什么？不需要登录、不需要集成的版本长什么样。
5. Observation & Surprise：你有没有坐在用户旁边看过他们怎么用，不插手？他们做的什么事出乎你的意料。
6. Future-Fit：三年后的世界会有本质的不同，你的产品会变得更重要还是更不重要？

每个问题带追问规则：第一遍回答通常是包装过的版本，要追到第二三遍。源码里给每个问题列了红旗清单，比如"我们拿到了 500 个 waitlist 报名"、"医疗行业的企业客户"这类回答会被直接打回。输出是一份重新框定后的设计文档。这和 Spec Kit 的 constitution 机制不同：constitution 划定约束边界，office-hours 挑战需求本身。

### 安全

`/cso` 实现两层安全：OWASP Top 10 检查加 STRIDE 威胁建模。STRIDE（Spoofing、Tampering、Repudiation、Information Disclosure、Denial of Service、Elevation of Privilege）逐类分析，输出威胁和对策。

Prompt Injection 防御是多层的：随浏览器打包的 22MB ML 分类器在本地扫描每个页面和工具输出；Claude Haiku 对整个对话形态做二次检查；系统 prompt 里埋随机 canary token，用来捕捉跨文本、工具参数、URL、文件写的外泄尝试；裁决层要求两个分类器一致才阻断，避免 Stack Overflow 这类自带指令文本的页面被误杀。可选开启 721MB 的 DeBERTa-v3 ensemble 做三取二表决。脱敏引擎 3 级分类（HIGH / MEDIUM / LOW），scan-at-sink 模式：数据在离开系统时扫描，而不是在进入时。

`/careful` 对破坏性命令（rm、drop、truncate 等）发警告。`/freeze` 锁定指定文件禁止编辑。`/guard` 组合 careful 和 freeze。

### 交付流水线

GStack 的交付是最完整的：

- `/ship`：sync → test → audit coverage → push → PR。全自动化。
- `/land-and-deploy`：合并 PR → 等 CI → 部署 → 验证生产健康。
- `/canary`：部署后监控循环，持续检查关键指标。
- `/document-release`：更新所有项目文档（CHANGELOG、README、API docs）。

从 PR 到生产监控，链路完整。其他套件最多做到自动 PR。

### 上下文管理

`/context-save` 和 `/context-restore` 显式保存和恢复上下文。你可以手动保存当前会话状态，下次恢复时跳过重复研究。GBrain 集成提供持久化存储：PGLite 本地、Supabase 云端、Remote MCP 三种后端。跨会话决策记忆用 `decisions.jsonl` 事件溯源存储：每次决策追加一条记录，不做 update-in-place，保证历史可追溯。

配合 Conductor（并行跑多个 Claude Code 会话的工具）使用时，Garry Tan 自称常态跑 10-15 个并行 sprint，这是他认为的当前实际上限。有一个实际坑写在 README 里：Conductor 会从每个 workspace 的进程环境中剥离 `ANTHROPIC_API_KEY` 和 `OPENAI_API_KEY`，需要改用 `GSTACK_` 前缀的环境变量注入。

### 其他

`/codex` 调用 OpenAI Codex 获取第二意见。`/retro` 做团队周回顾，支持跨项目全局模式。`/design-shotgun` 生成 4-6 个 AI mockup 变体，`/design-html` 把 mockup 转成生产 HTML/CSS。

SKILL.md 从 `.tmpl` 模板生成，总量上限 160K token。两层测试：gate（安全确定性测试，必须通过）和 periodic（质量基准测试，允许波动）。Diff-based 测试选择：只跑和本次变更相关的测试。Hermetic E2E 隔离环境：每个 E2E 测试在独立环境中运行，不依赖外部服务。

### 边界

仅支持 Claude Code，非跨平台。对嵌入式和固件项目不适用：Chromium Daemon 和浏览器 QA 在没有 GUI 的环境里无法运行。

## 研发生命周期横向对比

下面按 9 个阶段逐一对比关键差异。

### 需求与战略

OpenSpec 最轻量：`/opsx:propose` 一次性生成 proposal 和 specs，没有前置约束检查。Spec Kit 的 constitution 先行，所有规格必须符合宪法，宪法提供了项目级的一致性保证，但也增加了启动成本：每个项目第一次跑时需要花时间写 constitution。GStack 的 `/office-hours` 走的是另一个方向：不划约束边界，而是反向挑战需求本身，用六个诊断问题追问"这个项目该不该做"（问题清单见 GStack 一章）。GSD 的 `discuss-phase` 全流程引导需求讨论，产出 REQUIREMENTS.md。ECC 取决于选用哪个 Agent：ECC 有 67 个 Agent，不同 Agent 对需求阶段的处理不同，没有统一的需求流程。

| 套件 | 需求产出 | 前置约束 | 反向挑战 |
|------|---------|---------|---------|
| OpenSpec | proposal + specs | 无 | 无 |
| Spec Kit | constitution + specs | constitution 先行 | 无 |
| Superpowers | plan（含行为规则） | 行为规则层 | 无 |
| GSD | REQUIREMENTS.md | 无（discuss-phase 自由讨论） | 无 |
| ECC | 取决于 Agent | 取决于 Agent | 无 |
| GStack | 重新框定后的设计文档 | 无 | /office-hours 6 个强制问题 |

GStack 的反向挑战在这个阶段是独有功能。OpenSpec 和 Spec Kit 都假设"需求已经想清楚了"，GStack 假设"需求很可能没想清楚"。

### 规划与设计

GSD 的 4 路并行研究 Agent 在规划深度上最高：STACK.md、FEATURES.md、ARCHITECTURE.md、PITFALLS.md 四个维度同时研究，综合器合并。GStack 的三权分立（CEO / Designer / Eng Manager）从不同视角审查方案，审查深度高但研究深度不如 GSD 的 4 路并行。Superpowers 的 plan 自带 TDD checkbox，把测试策略写进计划，这是其他套件不做的。Spec Kit 的 `/speckit.clarify` 和 `/speckit.analyze` 提供需求澄清和代码库分析的质量门禁。

| 套件 | 研究深度 | 审查机制 | TDD 内建 |
|------|---------|---------|---------|
| OpenSpec | 单路探索 | 无 | 无 |
| Spec Kit | clarify + analyze 门禁 | constitution 合规检查 | 无 |
| Superpowers | brainstorming 自动触发 | 无 | plan 模板 RED-GREEN-REFACTOR checkbox |
| GSD | 4 路并行研究 Agent | plan-phase 审查 | 无 |
| ECC | 取决于 Agent | 取决于 Agent | TDD Workflow Skill |
| GStack | 单路 | 三权分立 | 无 |

### 任务拆解

GSD 的任务拆解粒度最细：每个原子任务包含验证命令和完成标准。Spec Kit 的任务带依赖关系和并行标记，可以转 GitHub Issues。OpenSpec 的 tasks 也有依赖关系，但没有并行标记和 Issue 转换。Superpowers 的任务拆解由 plan 模板驱动，粒度取决于模型。ECC 和 GStack 的任务拆解取决于选用的命令或角色。

GSD 和 Spec Kit 在任务拆解上最成熟：依赖关系、并行标记、完成标准都结构化了。其他四个要么粒度粗，要么结构化程度低。

### 编码实现

这里有一个设计选择的分歧。GStack 不包装实现：没有 `/implement` 命令，实现由主会话在清晰规划后自然完成。Garry Tan 的判断是：实现是模型最擅长的环节，套件在这一层介入越多，越可能干扰模型的能力。其他五个套件都有不同形式的实现编排。

GSD 的波执行并行加文件锁在并行实现上最成熟。ECC 的多模型路由（multi-plan / multi-execute）设计有亮点，但前面说过当前开箱无法运行。Superpowers 的子 Agent 并行依赖文件化产物和 ledger，开销比 GSD 的 Fresh Context 低，但上下文隔离也不如 GSD 彻底。

### 审查

GStack 的审查最偏执：`/plan-ceo-review`、`/plan-design-review`、`/plan-eng-review` 三个角色各起一个审查，再加 `/codex` 获取第二意见，还集成了 Greptile 做 PR 分类。这是"多个独立视角交叉审查"的思路。

ECC 的 AgentShield 在企业级安全审查上是独有的（但引擎在外部包，前面说过）。GSD 的审查走人工 UAT：verify-work 阶段产出 VERIFICATION.md 和 UAT.md，但最终验证仍靠人。Superpowers 内置 code-review Skill，是唯一把审查作为标准流程步骤的轻量套件。

OpenSpec 和 Spec Kit 没有内建审查。OpenSpec 的 `/opsx:verify` 验证实现与 specs 的一致性，但这是规格合规检查，不是代码审查。

### QA 测试

GStack 的 `/qa` 在真机浏览器里跑测试，这在六个套件中独一档。对于 Web 项目，这个能力填补了其他所有套件的空白。Superpowers 的 TDD 内建保证每个任务有测试，但测试在终端里跑，没有浏览器。ECC 有 `/e2e` 和 `/test-coverage` 命令，但 E2E 的具体实现取决于项目的测试框架。GSD 的混合验证在 verify-work 阶段运行，验证内容取决于 VERIFICATION.md 里声明的验证项。

OpenSpec 和 Spec Kit 没有 QA。

### 安全

ECC 的 AgentShield 是唯一达到企业级的安全方案（但引擎在外部包，本仓库的验证范围有限）。GStack 的 `/cso` 实现 OWASP Top 10 加 STRIDE 威胁建模，Prompt Injection 多层防御，脱敏引擎，这些在本仓库可验证。GSD 的 `/gsd-secure-phase` 由 security-auditor Agent 验证 plan 威胁模型中的缓解措施是否落到代码里，产出 SECURITY.md，范围比前两者窄，不接外部漏洞数据源。其余三个套件没有内建安全。

| 套件 | 安全能力 | 可验证范围 |
|------|---------|-----------|
| OpenSpec | 无 | 无 |
| Spec Kit | 无 | 无 |
| Superpowers | 无 | 无 |
| GSD | secure-phase 威胁模型缓解验证，产出 SECURITY.md | 本仓库可验证 |
| ECC | AgentShield + Prompt Defense + Hook 阻断 | 本仓库：Prompt Defense + Hook；AgentShield 在外部包 |
| GStack | /cso OWASP + STRIDE + 多层注入防御 + 脱敏 | 本仓库全部可验证 |

### 交付发布

GStack 最完整：`/ship`（PR）→ `/land-and-deploy`（合并 + CI + 部署 + 验证）→ `/canary`（监控循环）→ `/document-release`（文档更新）。GSD 的 `/gsd-ship` 自动创建 PR。OpenSpec 的 `/opsx:archive` 归档沉淀规格变更。Spec Kit 的 `/speckit.converge` 收敛验证但不涉及部署。Superpowers 的 `finishing-a-development-branch` 处理分支收尾但不部署。ECC 没有专用交付命令。

从 PR 到生产监控的完整链路，只有 GStack 覆盖。

### 上下文管理

GSD 的 Fresh Context 是架构级解决方案：每个 Agent 独立窗口，编排器只调度不执行，三档 Profile 按阶段切换，Context Monitor Hook 监控使用率。这个设计在长链路项目中的防腐效果明显：前序阶段的上下文不会污染后续阶段。

GStack 的显式 save/restore 加 GBrain 持久化存储提供了跨会话的记忆，适合需要多天推进的项目。ECC 的 Hook 加 MCP 向量搜索（claude-mem）在会话间搜索历史上下文，但向量搜索的召回精度取决于 embedding 质量和查询措辞。Superpowers 用 git worktree 隔离每个任务的上下文，开销低但隔离也不彻底：worktree 共享 git 历史，只是工作目录隔离。

OpenSpec 和 Spec Kit 按需加载上下文：只在执行命令时注入相关产物，不常驻 CLAUDE.md。Superpowers 和 ECC 的上下文常驻 CLAUDE.md：安装后持续占用窗口空间。

四种策略各有取舍：

| 策略 | 代表 | 优点 | 代价 |
|------|------|------|------|
| 常驻 CLAUDE.md | Superpowers / ECC | 模型始终知道流程约定 | 持续占用上下文 |
| 按需加载 | OpenSpec / Spec Kit | 不占用常驻空间 | 模型可能忘记约定 |
| 阶段化 Fresh Context | GSD | 阶段间上下文隔离彻底 | 编排开销大，多 Agent 协调复杂 |
| 显式 save/restore | GStack | 跨会话记忆，人可控 | 需要手动触发保存/恢复 |

## 核心架构机制对比

### 编排模型

六个套件全部使用 Markdown 提示编排：流程步骤写在 Markdown 文件里，由模型读取并按步骤执行。但同为 Markdown 编排，内部实现差异很大。

GSD 有 SDK：299 个 TypeScript 文件，包含类型定义、工具函数和运行时支持。这是六个套件中唯一有真正运行时的。GStack 有 5 个独立 CLI 工具和 Playwright 浏览器 Daemon，编排逻辑在 CLI 里而非纯 Markdown。ECC 有 50 个 JS Hook 脚本，分发器模式，编排发生在 Hook 事件触发链里。

OpenSpec 和 Spec Kit 的编排纯靠 Markdown：命令文件定义步骤，模型按步骤走，没有运行时兜底。Superpowers 的编排也是纯 Markdown，但靠文件化产物和 ledger 做轻量状态管理。

纯 Markdown 编排的共同问题是：如果模型读错了步骤描述（比如把"先写测试"理解成"先写实现"），执行路径就偏离了，没有解释器或类型系统来捕捉。GSD 的 SDK 和 GStack 的 CLI 在这一点上有优势：至少类型定义和运行时能保证命令存在、参数合法。

### 上下文管理策略

前面横向对比已经覆盖，这里补充一个维度：上下文防腐的机制光谱。

最左端是 OpenSpec 和 Spec Kit：上下文防腐靠产物文件的独立性，不同特性的产物互不引用，不会互相污染。中间是 Superpowers：git worktree 物理隔离，不同任务在不同目录下执行。右侧是 GSD：Fresh Context 架构级隔离，不同 Agent 不同窗口。最右端是 GStack：显式 save/restore 加持久存储，防腐靠人的判断决定什么时候保存、什么时候恢复。

### Agent 生成

Superpowers 和 GSD 用 Task subagent：任务粒度的临时 Agent，任务完成后销毁。GStack 用独立 SKILL.md 角色：每个角色有独立的提示和行为定义，跨任务复用。ECC 有 67 个纯 Markdown Agent，每个 Agent 是一个 Markdown 文件定义行为，没有运行时。OpenSpec 和 Spec Kit 没有独立的 Agent 概念，命令就是主会话的不同模式。

ECC 的 67 个 Agent 在数量上最多，但全部是 Markdown 定义，没有运行时保证。GStack 的角色虽然也是 SKILL.md 定义，但配合 CLI 工具和浏览器 Daemon，有部分运行时支撑。GSD 的 33 个 Agent 配合 SDK 和 Fresh Context，运行时支撑最完整。

### 安装与分发

| 套件 | 安装方式 | 安装器大小 | 侵入性 |
|------|---------|-----------|--------|
| OpenSpec | npm CLI | 小 | 低：无 CLAUDE.md，无 AGENTS.md 内容 |
| Spec Kit | uv CLI（Python） | 中 | 低：CLI 不管 CLAUDE.md，上下文可选 |
| Superpowers | Claude Code 插件市场 / npx skills add | 小 | 中：常驻 CLAUDE.md |
| GSD | git clone + install.js | 11,376 行安装器 | 高：.planning/ 目录 + 14 Hook + CLAUDE.md |
| ECC | npm 包 + 插件 | 大 | 高：37KB hooks.json + CLAUDE.md + 全局配置 |
| GStack | bun install | 中 | 中：CLI 工具 + 可选 GBrain 后端 |

GSD 的 11,376 行安装器是一个需要重视的成本。安装过程涉及创建目录结构、注册 Hook、写入配置文件、初始化 git hooks。如果安装中断或出错，清理不完整可能导致项目状态不一致。

### 测试体系

GSD 的 549 个测试文件约 132,441 行测试代码，是唯一有规模级测试的套件。ECC 有 197 个测试文件，还有 docs-vs-code 一致性测试（验证文档描述和代码实现一致）。GStack 有两层测试：gate（安全确定性）和 periodic（质量基准），加上 Diff-based 测试选择和 Hermetic E2E 隔离。Superpowers 和 OpenSpec 没有测试。Spec Kit 的测试主要在 CLI 层，不在 Skill 产物层。

## 选型指南

按场景给建议，不讲"哪个最好"。

### Solo 开发者快速原型

Superpowers 加 GStack 的 `/qa` 和 `/cso`。Superpowers 提供轻量流程（TDD 内建、审查内建），GStack 的浏览器 QA 和安全审查按需调用，不走完整角色流程。这样组合的侵入性低：Superpowers 约 4,700 行 Markdown，GStack 按需调用不装全套。

前提：你的项目是 Web 项目（GStack 的 `/qa` 依赖浏览器）。非 Web 项目把 GStack 换成 ECC 的 `/test-coverage`。

### 团队新项目启动

Spec Kit 加 GSD 加 GStack 的 `/review` 和 `/qa`。Spec Kit 的 constitution 在项目启动时划清约束边界，GSD 接管执行和验证，GStack 的审查和 QA 按需调用。这个组合覆盖从需求到验证的完整链路，但安装成本高：Spec Kit 的 Python CLI、GSD 的 11,376 行安装器、GStack 的 bun install 都需要配置。

前提：团队愿意接受 Spec Kit 的 30min-3hrs per feature 开销，且项目规模大到能摊薄这个固定成本。

### 企业级现有项目维护

OpenSpec 加 ECC 加 GStack 的 `/canary`。OpenSpec 在已有项目上启动成本低，不需要 constitution，直接 `/opsx:propose` 走起。ECC 的 Hook 生命周期（50 个 JS 脚本、7 类事件）和 AgentShield 提供企业级安全。GStack 的 `/canary` 提供部署后监控。

前提：ECC 的 37KB 配置文件和 Hook 复制粘贴问题在多人维护时需要专人管理。GStack 的 `/canary` 需要部署基础设施支持。

### 全栈复杂重构

GSD 加 ECC 的 multi-plan / multi-execute。GSD 的 Fresh Context 和波执行在长链路重构中防腐效果明显，4 路并行研究 Agent 对大范围重构的前期研究有帮助。ECC 的多模型路由在架构决策时提供多视角。

前提：multi-plan / multi-execute 当前开箱无法运行，需要自行安装 ccg-workflow 运行时和角色文件。GSD 的学习曲线（67 命令）需要投入时间。

### SaaS 产品持续交付

GStack 加 Spec Kit constitution。GStack 的交付链路最完整（PR → CI → canary → 监控），Spec Kit 的 constitution 保证迭代过程中约束边界不变。这个组合适合需要频繁发布且每次发布都要过安全审查和性能监控的场景。

前提：仅限 Web 项目。GStack 的交付链路假设了 CI/CD 和部署基础设施。

### 选型决策要点

- 需要安全合规：ECC 或 GStack。ECC 的 AgentShield 企业级但引擎在外部包；GStack 的 `/cso` 在本仓库可验证。
- 需要浏览器测试：GStack，没有其他选择。
- 需要长链路上下文防腐：GSD 的 Fresh Context 架构级隔离效果最好。
- 需要团队标准化：Spec Kit 的 constitution 先行保证全团队规格一致性。
- 需要最小开销：OpenSpec 或 Superpowers。OpenSpec 无常驻上下文，Superpowers 约 4,700 行 Markdown 零依赖。

## 结语

最重要的判断是"哪个套件的抽象边界和我的问题边界最匹配"。GStack 为 Web SaaS 项目设计的角色分离和浏览器 QA，用在 CLI 工具开发上大部分能力浪费。GSD 为长链路项目设计的 Fresh Context 和波执行，用在改两行配置的场景上开销不可接受。OpenSpec 的极简流程在需求明确的小项目上刚好够用，在需要安全审查和部署监控的企业项目上不够。

功能最多不等于最适合，最简单不等于最差。选错的代价是：调试框架本身比调试代码更痛苦。GSD 的 67 命令和 14 Hook 如果和你项目的约定冲突，排查冲突来源比排查业务 bug 更难，因为框架的错误信息不指向你写的代码，指向框架内部的编排逻辑。

我目前的使用方式是混合：个人项目用 Superpowers 的 TDD 约束加 OpenSpec 的规格管理，Web 项目加上 GStack 的 `/qa` 和 `/cso`，不用任何套件的完整流程。这个选择牺牲了流程完整性，换取了低侵入性和灵活组合。如果你的项目只够用一套，选抽象边界最小、刚好覆盖你最痛问题的那个。
