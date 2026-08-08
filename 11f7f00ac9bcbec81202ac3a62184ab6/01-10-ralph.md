---
title: Ralph 详解：把 AI 编程助手放进 while 循环的自主开发方法
date: 2026-01-10 23:00
series: ai-coding
tags: ["AICoding"]
description: 系统梳理 Ralph 这一自主 AI 编程循环技术：它是什么、解决什么约束、工作原理与设计原则、如何落地、适用与不适用的场景，以及三个实践案例分析，包括 aimo 项目从 Ralph 迁移到 superpowers 的完整记录。
published: true
---

2025 年 7 月，Geoffrey Huntley 发表了 [Ralph Wiggum as a "software engineer"](https://ghuntley.com/ralph/)，提出一种用 bash 循环驱动 AI 编程工具连续工作的方法，名字取自《辛普森一家》里那个迟钝但从不放弃的 Ralph Wiggum。此后半年里，社区出现了多个实现和变体，Anthropic 也在 Claude Code 官方插件市场上架了 [ralph-wiggum 插件](https://github.com/anthropics/claude-code/tree/main/plugins/ralph-wiggum)。我自己在 [aimo](https://github.com/ximing/aimo) 项目上大规模用过 Ralph，之后把脚本移植到本仓库（`scripts/ralph/`），再往后工作流又切到了 superpowers，这些经历都会在下文展开。这篇文章最早只有几段概念简介，这次我把原始资料通读了一遍，重写成完整版：是什么、为什么、怎么用、什么场景用，外加两个实例分析。

## Ralph 是什么

Ralph 的最纯形式是一行 bash：

```bash
while :; do cat PROMPT.md | claude ; done
```

一个 while 循环，每轮启动一个全新的 AI 编程工具进程，把同一份 PROMPT.md 喂进去。Agent 读磁盘上的计划文件，挑一件事做，做完提交代码、更新计划文件，然后退出。循环再启动下一轮，上下文完全清零，靠磁盘文件恢复状态。如此往复，直到任务列表清空。

这个方法有两个关键点，缺一个都不算 Ralph：

1. 每轮迭代是全新上下文。Agent 进程每轮退出，不保留任何对话历史。
2. 磁盘文件是唯一记忆。任务清单、进度状态、经验教训全部落盘，每轮被重新读取。

社区目前有两种实现形态，差别就在这两点执行得彻不彻底：

- 外部 bash 循环（原版）：进程级隔离，每轮上下文真正清零。前面那行脚本就是全部机制。
- 会话内循环（官方插件）：Claude Code 的 ralph-wiggum 插件用 Stop hook 拦截 Agent 的退出动作，把同一段 prompt 再喂回去，循环发生在同一个会话里。它零配置、带完成信号和最大轮次保护，但上下文在轮次间不清零，会话越长上下文越臃肿。Huntley 本人明确认为插件版丢掉了 Ralph 的核心（他为此专门录了一期视频 "why the claude code plugin isn't it"）。插件适合体验和小任务，长任务仍应使用外部循环。

效果方面，Huntley 给出的数据是：他用 Ralph 花三个月构建了 CURSED，一门刻意设计、不存在于任何训练数据里的新编程语言，Ralph 不仅写出了编译器，还能用这门语言写程序；另一个流传较广的数字是一份 $50k 的外包合同用 Amp 跑 Ralph 以 $297 的 API 成本交付。这些数字来自 Huntley 本人，无法独立验证，但方向上与后续社区的复现一致（见后文 YC 黑客松案例）。

## 为什么需要 Ralph：上下文窗口是硬约束

Ralph 要解决的问题是上下文窗口的物理限制。以 Claude 为例，名义上下文是 200k token，Huntley 实测输出质量在 147k 到 152k 左右就开始下降（[Ralph Playbook](https://github.com/ghuntley/how-to-ralph-wiggum) 按 40% 到 60% 利用率划了一个 "smart zone"，超出这个区间模型的推理质量持续劣化）。

一个连续工作的 coding agent 必然越过这个区间：读文件、跑命令、工具返回的结果全部累积进上下文。越过之后的表现是遗忘之前的决定、重复已实现的功能、给出自相矛盾的修改。这就是长会话里的 AI 助手给人「越聊越笨」感觉的原因。

Ralph 的应对策略是把会话切短：每轮只做一件事，上下文占用始终留在高效区间；跨轮需要的信息全部写入磁盘文件，下一轮重新加载。Huntley 对此有一句概括：Ralph 是 "deterministically bad in an undeterministic world"。每轮加载相同的文件、走相同的流程，所以失败模式是稳定可观测的，发现一类失败就可以在 prompt 里加一条约束消除它，这比调试一个行为飘忽的长会话代理可行得多。

## 工作原理：磁盘文件承担记忆

![Ralph 循环工作原理](../../assets/ralph/loop.svg)

外层的 bash 循环本身没有任何智能，全部智能在磁盘文件和 prompt 里。单次迭代的生命周期（按 Ralph Playbook 的归纳）是：

1. Orient：用子代理研读 `specs/` 下的需求规格。
2. Read plan：读计划文件（`IMPLEMENTATION_PLAN.md` 或 `prd.json`），了解当前进度。
3. Select：选出优先级最高的一件未完成事项。
4. Investigate：先搜索代码库，确认这件事确实还没被实现。
5. Implement：实现它，文件读写尽量交给并行子代理。
6. Validate：运行构建、类型检查、测试，不通过就修复。
7. Update：把计划文件里对应事项标记完成，把新发现的问题和经验教训写回磁盘。
8. Commit 退出：提交代码，进程结束，循环立刻开始下一轮。

跨迭代记忆由三类文件承担：计划文件记录「做什么、做完没有」，进度日志（`progress.txt`）和 `AGENTS.md` 记录「踩过什么坑、这个仓库怎么构建测试」，git 历史提供每轮一个 commit 的可回滚快照。循环的退出条件是文本约定：prompt 里要求 Agent 在全部任务完成时输出 `<promise>COMPLETE</promise>`，外层脚本 grep 到这串文本就终止循环。

## 关键设计原则

原始文章和 Playbook 提到的原则不少，下面八条被反复强调，且在我自己的实践中确实影响成败。

1. 每轮只做一件事。Huntley 在原文里把这条重复了三遍。项目后期可以放宽，一旦跑偏立刻收回。任务粒度越小，单轮上下文占用越低，质量越稳定。

2. 每轮确定性加载相同的上下文。每轮栈底都是同样的 PROMPT.md、AGENTS.md 和 specs/，Agent 从相同的已知起点出发。这是失败模式可复现、可调优的前提。

3. 主上下文只当调度器。搜索代码、读文件这类占上下文的工作全部派给子代理，每个子代理的上下文独立、用完即弃。只有一个例外：构建和测试只允许单个子代理执行，否则几百个子代理同时跑验证会互相踩踏（Huntley 称之为 back pressure 拥塞）。

4. 用 backpressure 拒绝坏代码。类型检查、测试、lint、安全扫描，任何能自动拒绝不合格产出的机制都算。Huntley 的原话是 "the speed of the wheel turning" 与正确性需要权衡：Rust 类型系统最严，但编译慢、LLM 一次写对 Rust 的概率低，迭代节奏会被拖慢。用动态语言跑 Ralph 必须接静态检查工具（原文举了 Erlang 的 dialyzer 和 Python 的 pyrefly），否则坏代码会一路累积。

5. Specs 先行。开循环之前，先花很长时间和 LLM 对话需求，让它把规格按主题一个文件一篇写进 `specs/`。Huntley 的教训：CURSED 项目跑到一个月时他才发现词法规格里一个关键字被定义了两次，Ralph 忠实地按错误规格干活，浪费了大量迭代。Ralph 做错了方向，先怀疑规格，再怀疑工具。

6. Planning 和 Building 两种 prompt 分开。同一套循环配两份 prompt：planning 模式只做差距分析（对比 specs 与现有代码），产出按优先级排序的 TODO 列表，不写代码；building 模式按计划实现。计划是易耗品，跑偏、过时或者完成后堆积太多时，删掉重跑一轮 planning 即可，成本只有一轮迭代。

7. 调优靠观察失败、加约束。Huntley 把这比作在游乐场立告示牌（signs）：Ralph 从滑梯上摔下来，就在旁边立一块「滑下来，不要跳」的牌子。两条出现频率最高的约束：

   - 改动前先搜索（"don't assume not implemented"）。代码搜索有非确定性，Agent 搜不到就断定功能不存在，于是重复实现。Huntley 称这是 Ralph 的 Achilles' heel，prompt 里必须明确要求「不要假设功能没实现」。
   - 禁止 placeholder 实现。模型的奖励函数偏向「能编译」，会偷偷写桩代码和最小实现。CURSED 的 prompt 里有一条编号 9999999999999999999999999999 的约束："DO NOT IMPLEMENT PLACEHOLDER OR SIMPLE IMPLEMENTATIONS. WE WANT FULL IMPLEMENTATIONS. DO IT OR I WILL YELL AT YOU"。

8. 自主运行的前提是沙箱。循环要无人值守就得跳过权限确认（Claude Code 是 `--dangerously-skip-permissions`），权限层失效之后沙箱就是唯一防线：Docker 或云主机隔离、只给任务必需的最小凭据、限制网络出口。Playbook 的态度是 "It's not if it gets popped, it's when. And what is the blast radius?"。逃生手段是 Ctrl+C 终止循环和 `git reset --hard` 回滚。

## 怎么落地

### 文件布局

一个典型的 Ralph 工作目录：

```
project-root/
├── loop.sh                     # 外层循环脚本
├── PROMPT_plan.md              # planning 模式指令
├── PROMPT_build.md             # building 模式指令
├── AGENTS.md                   # 操作手册：怎么构建、测试、常见坑
├── IMPLEMENTATION_PLAN.md      # 优先级排序的任务清单（Ralph 维护）
├── specs/                      # 需求规格，一个主题一个文件
└── src/
    └── lib/                    # 共享工具库，约束 Ralph 复用既有模式
```

### 循环脚本

Playbook 给出的增强版核心逻辑（支持 plan/build 模式切换和最大轮次）：

```bash
#!/bin/bash
# Usage: ./loop.sh [plan] [max_iterations]
if [ "$1" = "plan" ]; then
    PROMPT_FILE="PROMPT_plan.md"; MAX_ITERATIONS=${2:-0}
else
    PROMPT_FILE="PROMPT_build.md"; MAX_ITERATIONS=${1:-0}
fi

ITERATION=0
while true; do
    if [ $MAX_ITERATIONS -gt 0 ] && [ $ITERATION -ge $MAX_ITERATIONS ]; then
        break
    fi
    cat "$PROMPT_FILE" | claude -p \
        --dangerously-skip-permissions \
        --output-format=stream-json \
        --verbose
    ITERATION=$((ITERATION + 1))
    echo "======================== LOOP $ITERATION ========================"
done
```

关键参数：`-p` 让 Claude Code 以非交互模式从 stdin 读 prompt；`--dangerously-skip-permissions` 跳过全部权限确认；`--output-format=stream-json` 输出结构化日志便于监控。同样的循环也适用于 amp、codex、opencode 等 CLI。

### 官方插件（轻量替代）

不想维护脚本可以用官方插件，一条命令在会话内开循环：

```bash
/ralph-loop "Build a REST API for todos. Requirements: CRUD operations, input validation, tests. Output <promise>COMPLETE</promise> when done." \
  --completion-promise "COMPLETE" --max-iterations 50
```

两个注意点：`--completion-promise` 是精确字符串匹配，无法表达「成功或受阻」双状态，所以 `--max-iterations` 才是主要的保险；prompt 里应写明卡死时的行为，比如「15 轮后仍未完成就记录阻塞原因、已尝试的方案和建议的替代路径」。停止循环用 `/cancel-ralph`。

### Prompt 编写要点

综合官方插件 README 和社区实践：

- 完成标准必须可自动验证。「做一个好的 todo API」不行，「全部 CRUD 端点可用、输入校验就位、测试通过且覆盖率大于 80%」才行。
- 大需求拆成有先后顺序的阶段，每个阶段自带验证手段。
- 在 prompt 里写清自我修正流程：写测试、实现、跑测试、失败就修、全绿才提交。
- 始终设置最大轮次。
- 保持简短。这条在多个来源里都被提到，黑客松案例里有具体数据。

## 适用场景与边界

适合：

- Greenfield 新项目。Huntley 的判断是 Ralph 能完成 greenfield 项目约 90% 的工作，剩下的靠人。
- 有可自动验证的完成标准的任务，类型检查、测试套件、lint 至少要有其一。
- 代码移植与语言转换，源实现就是规格，验证手段现成。
- Spec-to-code，需求能先写成清晰规格的任务。
- 能拆成一系列独立小任务、单任务在一个上下文窗口内装得下的需求。

不适合：

- 存量复杂代码库。Huntley 的原话："There's no way in heck would I use Ralph in an existing code base"。存量库里约束多隐式、影响面难界定，自动门禁覆盖不到的地方 Ralph 会造成静默破坏。
- 完成标准模糊、需要人类设计判断的任务。
- 一次性操作和生产事故调试，这类任务要的是针对性排查，循环只会放大误操作。
- 没有测试也没有类型系统的项目，先补门禁再谈 Ralph。

## 实例分析

### 案例一：repomirror，YC 黑客松一夜移植六个仓库

YC Agents 黑客松上，repomirror 团队把 Claude Code 放进 while 循环跑了一夜，早上的结果是约 1100 个 commit 和六个移植完成的仓库（[完整报告](https://github.com/repomirrorhq/repomirror/blob/main/repomirror.md)）。典型任务是把 Python 写的 browser-use 移植成 TypeScript，prompt 只有几行：你的工作是移植并维护这个仓库、每次文件修改后提交、用 `.agent/` 目录当草稿本存 TODO。他们还从文档的 llms-full.txt 出发做了两个 spec-to-code 实验。总推理成本不到 $800，折算下来每个 Sonnet agent 每小时约 $10.50。

这份报告的价值在于它如实记录了意外现象和失败边界：

- Prompt 越短越好。他们曾让 Claude 帮忙「优化」prompt，膨胀到 1500 词后 agent 立刻变慢变笨；改回 103 词后恢复正常。指令越多，每轮确定性越差。
- Agent 会超纲。完成 AI SDK 的 Python 移植后，agent 自发加了原版没有的 Flask、FastAPI 集成和多种 schema 校验器支持。无人值守时这是惊喜也可能是失控，取决于仓库边界有没有写进 prompt。
- Agent 会自救也会认怂。多数 agent 完成后转入写额外测试、完善 TODO 的状态；有一个 agent 发现自己陷入死循环后用 `pkill` 终止了自己的进程。这同时说明沙箱里进程权限要收紧。
- 90% 到 100% 靠人。两个移植仓库都有宣称完成但实际跑不通的 demo，团队最后回到交互模式人工收尾。Ralph 压缩了从 0 到 90% 的时间，最后 10% 仍需人工完成。

### 案例二：本仓库的 scripts/ralph 实践

本仓库的 `scripts/ralph/` 是从 aimo 项目移植过来的一套实现（两个仓库的脚本只有一行模型参数的差异），目录里的 `progress.txt` 保留了它在 aimo 上的一次真实运行记录，可以作为存量约束下改造的样本。

我的版本和原版有几个结构差异。用结构化的 `prd.json` 替代纯 Markdown 计划：每个 user story 带 `passes` 标志，prompt（`scripts/ralph/CLAUDE.md`）要求 Agent 每轮「选优先级最高且 `passes: false` 的 story」，选任务从自然语言推理变成确定性查询。`ralph.sh` 支持 `--tool amp|claude` 双工具和最大轮次参数，并根据 `prd.json` 里的 `branchName` 检测需求切换，自动把上一轮的 prd 和 progress 归档到 `archive/日期-分支名/`。完成信号同样是 `<promise>COMPLETE</promise>`。

经验沉淀上有一处改进。`progress.txt` 只追加不覆盖，纯追加会让经验被日志淹没，所以 prompt 里加了一条规则：可复用的模式必须提炼到文件顶部的 `## Codebase Patterns` 区，每轮开工先读这个区。实际跑下来这个区积累的内容是有效的，比如「LanceDB 的 `update()` 接受 `{ where, values }` 对象而非位置参数」「排序要在 `.toArray()` 后用 JS 做，没有 `.orderBy()`」「表名用 snake_case、主键用全名加 Id」。这些条目让后续迭代没有再犯同样的 API 误用。

运行记录：2026 年 2 月 20 日一天内，这套循环完成了一个 AI 会话功能的九个 story：从 LanceDB 建表和迁移脚本（US-001），到会话 CRUD 后端 API（US-002），前端状态层从 localStorage 切换到 API（US-003），再到侧边栏会话列表、浮动输入框等 UI 重构（US-004 到 US-009）。每轮通过 build 和 lint 门禁才允许 commit。

回顾这个案例，有三点值得说明。第一，这发生在存量代码库里，与 Huntley「只用于 greenfield」的告诫相悖，能成立的条件是：功能本身是边界清晰的新模块、TypeScript 加 lint 提供了每轮必跑的硬门禁、story 粒度足够小；缺少任何一个条件我都不会在存量库里跑。第二，结构化 `prd.json` 降低了每轮选任务的方差，代价是 JSON 的 token 效率低于 Markdown，Playbook 也建议计划文件用 Markdown；任务量不大时这个代价可以接受，任务上百条时应该换回 Markdown。第三，质量门禁是整个系统里唯一不能省的部分，九个 story 里多次出现第一轮实现 lint 不过、Agent 自修复后才提交的情况，没有门禁这些半成品会直接进入历史。

### 案例三：aimo 项目，从 Ralph 切换到 superpowers(3月10日 补充)_

[aimo](https://github.com/ximing/aimo) 是我的一个 memo 应用，也是我最大规模的一次 Ralph 实践：早期的大部分功能是用 Ralph 批量生产出来的，后来项目进入维护迭代阶段，我把它换成了 superpowers。这个切换过程本身就很能说明 Ralph 的适用边界。

Ralph 阶段大约三周（2026 年 2 月中到 3 月初），产出二十个 PRD、十余个 PR：memo 双向链接、Electron 客户端、Chrome 扩展、AI 会话存储、LanceDB 到 MySQL 的存储迁移都在其中，案例二那份 `progress.txt` 记录的九个 story 就是这个阶段跑的。每个 PRD 一个独立分支，跑完归档到 `tasks/archive/`。中间试过官方 ralph-wiggum 插件，后来还是用回自己的脚本，aimo 版本里把循环模型固定成了 MiniMax-M2.5。

换掉它的原因，我的判断是项目阶段变了，而不是任务变小了。Ralph 时代最后做的回收站，和 superpowers 时代做的通知中心，规模其实差不多；不一样的是工作性质。前三周接近 greenfield，功能是一个接一个生产出来的；那之后 aimo 的工作变成了另一类活：给接口补越权检查和 SQL 注入防护、把 N+1 查询改成批量查询、补单元测试。这正是 Huntley 划的那条边界：批量造功能的阶段 Ralph 在甜区，进入存量库的加固迭代后，无人值守循环的风险收益比变差，需要人在关键环节确认的工作流。等于我自己把「不适用存量复杂代码库」这条验证了一遍。

切换本身很平滑，因为两套方法共享的部分比差异多。需求先写成规格、任务拆小、每步可验证，这些原则换工具不变，被替换的只是最外层的驱动方式。Ralph 的 `prd.json` 把需求、验收标准和完成状态压成一份机器可读的清单，人在循环之外做事后审查；superpowers 把 design 和 plan 拆成两份写给人看的文档（`docs/superpowers/`，命名沿用自 Ralph 阶段的 specs），在设计和计划两个节点留人确认，确认之后才交给子代理执行。循环脚本是最不值钱的资产，真正能带走的是 specs、计划文档和 `progress.txt` 里积累的 Codebase Patterns。两套方法的详细对比我另写了一篇 [Ralph 与 Superpowers 的区别](/post/2026/02.02-superpowers/)，核心结论是两者的分歧在纠偏时机：Ralph 靠事后快改容错（计划可抛弃，重跑一轮 planning 即可），Superpowers 靠执行前确认容错（人在设计和计划两个环节确认后才执行）。greenfield 批量造功能时 Ralph 在甜区，存量维护需人判断时 Superpowers 更适合。aimo 进入维护阶段之后，恰好属于后者。

## 局限与风险

- Placeholder 倾向无法根除。模型奖励函数偏向「能编译」，禁止桩代码的约束会被间歇性无视。缓解办法是定期跑一轮专门搜索 TODO、占位实现和最小实现的 planning 迭代，把它们变成后续任务。
- 早上面对编译不过的代码库是常态。Huntley 明说这会发生，此时要做判断：`git reset --hard` 重跑，还是写一组新 prompt 去救。这个判断目前只能人做。
- 成本不低。按黑客松的数据每个 agent 每小时约 $10.50，无人值守跑一夜是实打实的账单，max-iterations 和沙箱预算告警都要提前设好。
- 仍然需要资深工程师。CURSED 的规格错误是 Huntley 自己发现并修正的，prompt 调优依赖对失败模式的诊断能力。Huntley 的原话是：声称不需要工程师、工具能 100% 交付的说法是 "peddling horseshit"。
- 多 Ralph 并行还不成熟。Huntley 的观点是多代理通信相当于让非确定性的微服务互相调用，当前阶段单体循环加一个主上下文调度子代理是更可靠的形态。

## 参考资料

- [Ralph Wiggum as a "software engineer" — Geoffrey Huntley](https://ghuntley.com/ralph/)
- [The Ralph Playbook — how-to-ralph-wiggum](https://github.com/ghuntley/how-to-ralph-wiggum)
- [Claude Code ralph-wiggum 官方插件](https://github.com/anthropics/claude-code/tree/main/plugins/ralph-wiggum)
- [We Put a Coding Agent in a While Loop and It Shipped 6 Repos Overnight — repomirror](https://github.com/repomirrorhq/repomirror/blob/main/repomirror.md)
