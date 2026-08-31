---
title: Ralph 详解：把 AI 编程助手放进 while 循环的自主开发方法
date: 2026-02-10 23:00
series: ai-coding
tags: ["AICoding"]
description: 系统梳理 Ralph 这一自主 AI 编程循环技术：它是什么、解决什么约束、工作原理与设计原则、如何落地、适用与不适用的场景，以及三个实践案例分析，包括 aimo 项目从 Ralph 迁移到 superpowers 的完整记录。
published: true
---

2025 年 7 月，Geoffrey Huntley 发表 [Ralph Wiggum as a "software engineer"](https://ghuntley.com/ralph/)，介绍了用 bash 循环驱动 AI 编程工具持续执行任务的方法。名称来自《辛普森一家》中的 Ralph Wiggum。此后半年，社区出现了多个实现和变体；Anthropic 也在 Claude Code 官方插件市场发布了 [ralph-wiggum 插件](https://github.com/anthropics/claude-code/tree/main/plugins/ralph-wiggum)。我曾在 [aimo](https://github.com/ximing/aimo) 项目中大量使用 Ralph，随后将脚本移植到本仓库的 `scripts/ralph/`，后来又切换到 superpowers。本文根据原始资料和这些实践，说明 Ralph 的机制、适用条件与限制，并分析三个案例。

## Ralph 的运行方式

Ralph 的最简形式是一行 bash：

```bash
while :; do cat PROMPT.md | claude ; done
```

循环每轮启动一个新的 AI 编程工具进程，并输入同一份 `PROMPT.md`。Agent 读取磁盘上的计划文件，完成一项任务，提交代码并更新计划文件后退出。下一轮进程不保留对话历史，只能从磁盘文件恢复任务状态。任务列表清空后，循环结束。

Ralph 包含两个条件：

1. 每轮迭代使用新上下文。Agent 进程退出后不保留对话历史。
2. 任务清单、进度和经验记录在磁盘文件中，下一轮重新读取。

社区有两种实现：

- 外部 bash 循环是原始形式。进程在每轮重新创建，上下文会清零；前面的脚本包含其全部循环机制。
- 会话内循环是官方插件的形式。Claude Code 的 ralph-wiggum 插件通过 Stop hook 拦截 Agent 退出，并再次输入同一段 prompt。它无需额外脚本，提供完成信号和最大轮次保护；但会话上下文不会在轮次间清零，长会话会累积更多上下文。Huntley 在视频 "why the claude code plugin isn't it" 中认为插件版缺少每轮清空上下文这一特征。插件可用于体验或小任务，长任务仍需评估外部循环。

Huntley 的数据称，他使用 Ralph 用三个月构建了 CURSED：一门刻意设计且不在训练数据中的新编程语言。Ralph 写出了编译器，也能用该语言编写程序。另一项广泛传播的数据称，一份 $50k 的外包合同通过 Amp 运行 Ralph，以 $297 的 API 成本完成交付。这些数据来自 Huntley 本人，无法独立验证；后文的 YC 黑客松案例提供了社区复现的参考。

## 上下文窗口限制

Ralph 针对的是上下文窗口的限制。以 Claude 为例，名义上下文是 200k token；Huntley 的实测认为，输出质量在 147k 到 152k token 左右开始下降。[Ralph Playbook](https://github.com/ghuntley/how-to-ralph-wiggum) 将 40% 到 60% 的利用率定义为 "smart zone"，并认为超出这一范围后模型推理质量会持续降低。

连续运行的 coding agent 会持续累积读文件、运行命令和工具返回的结果。超过该范围后，可能遗漏先前决定、重复已实现的功能，或产生相互矛盾的修改。这也是长会话中常见的上下文退化现象。

Ralph 将工作拆为短会话：每轮完成一项任务，将跨轮信息写入磁盘，并在下一轮加载。Huntley 用 "deterministically bad in an undeterministic world" 概括这一做法：每轮加载相同文件并执行相同流程，因此失败模式更稳定、更容易观察。发现一种失败后，可以在 prompt 中增加对应约束；相较于调试行为变化较大的长会话，这种方式更便于定位问题。

## 磁盘文件如何保存状态

![Ralph 循环工作原理](../../assets/2026/02-10-ralph/loop.svg)

bash 循环只负责重复启动进程，任务信息和执行约束由磁盘文件与 prompt 提供。Ralph Playbook 将一次迭代归纳为以下步骤：

1. Orient：使用子代理阅读 `specs/` 中的需求规格。
2. Read plan：读取计划文件，例如 `IMPLEMENTATION_PLAN.md` 或 `prd.json`，确认当前进度。
3. Select：选择优先级最高的未完成事项。
4. Investigate：搜索代码库，确认该事项尚未实现。
5. Implement：实现任务；文件读写可交由并行子代理处理。
6. Validate：执行构建、类型检查和测试，失败则修复。
7. Update：标记计划文件中已完成的事项，并将新发现的问题和经验写回磁盘。
8. Commit 并退出：提交代码、结束进程，再由循环启动下一轮。

计划文件记录任务和完成状态；进度日志（`progress.txt`）与 `AGENTS.md` 记录构建、测试方式及已知问题；git 历史保存每轮 commit，可用于回滚。退出条件通常也由文本约定：prompt 要求 Agent 在所有任务完成后输出 `<promise>COMPLETE</promise>`，外层脚本通过 grep 匹配该文本后终止循环。

## 执行约束

原始文章和 Playbook 反复强调以下约束；它们也影响了我的实践结果。

1. 每轮只完成一项任务。Huntley 在原文中重复了三次。项目后期可以扩大粒度；出现偏差后应重新缩小。任务越小，单轮的上下文占用越低，结果更稳定。
2. 每轮加载相同的上下文。`PROMPT.md`、`AGENTS.md` 和 `specs/` 构成固定起点，使失败模式可以复现和调整。
3. 主上下文负责调度。搜索代码、读取文件等消耗上下文的工作可交给子代理，每个子代理独立运行并在完成后退出。构建和测试只应由一个子代理执行，避免大量子代理同时验证并相互干扰；Huntley 将这种问题称为 back pressure 拥塞。
4. 使用 backpressure 阻止不合格代码进入历史。类型检查、测试、lint 和安全扫描都可以承担这一作用。Huntley 指出，"the speed of the wheel turning" 与正确性存在取舍：Rust 类型系统严格，但编译较慢，且 LLM 一次写对 Rust 的概率较低，会降低迭代速度。动态语言项目运行 Ralph 时需要接入静态检查工具；原文举例为 Erlang 的 dialyzer 和 Python 的 pyrefly，否则不合格代码会累积。
5. 先完成 Specs。启动循环前，可与 LLM 讨论需求，并按主题写入 `specs/` 中的文件。CURSED 运行一个月后，Huntley 发现词法规格中一个关键字被定义两次；Ralph 依据错误规格执行，消耗了大量迭代。因此需要先检查规格，再检查工具行为。
6. 为 Planning 和 Building 使用不同的 prompt。planning 模式只对比 `specs` 与现有代码，输出按优先级排序的 TODO 列表，不修改代码；building 模式按计划实现。计划出现偏差、过期或堆积过多时，可以重新运行一轮 planning，成本为一轮迭代。
7. 根据失败记录补充约束。Huntley 将其描述为在出现错误的位置添加提示。常见约束包括：
   - 修改前搜索代码。代码搜索存在非确定性；Agent 搜索不到时可能将功能误判为未实现，造成重复实现。Huntley 将其称为 Ralph 的 Achilles' heel，prompt 应明确要求 "don't assume not implemented"。
   - 禁止 placeholder 实现。模型可能优先生成能编译的桩代码或最小实现。CURSED 的 prompt 包含编号 9999999999999999999999999999 的约束："DO NOT IMPLEMENT PLACEHOLDER OR SIMPLE IMPLEMENTATIONS. WE WANT FULL IMPLEMENTATIONS. DO IT OR I WILL YELL AT YOU"。
8. 无人值守运行需要沙箱。Claude Code 使用 `--dangerously-skip-permissions` 跳过权限确认后，隔离环境成为主要防线。可采用 Docker 或云主机隔离、只提供任务所需的最小凭据并限制网络出口。Playbook 提醒："It's not if it gets popped, it's when. And what is the blast radius?"。可用 Ctrl+C 终止循环，使用 `git reset --hard` 回滚。

## 实现 Ralph 循环

### 文件布局

典型的 Ralph 工作目录如下：

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

Playbook 提供的核心逻辑支持 plan/build 切换和最大轮次：

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

`-p` 使 Claude Code 以非交互模式从 stdin 读取 prompt；`--dangerously-skip-permissions` 跳过权限确认；`--output-format=stream-json` 输出便于监控的结构化日志。相同的循环可用于 amp、codex、opencode 等 CLI。

### 官方插件

不维护脚本时，可通过官方插件在一个会话中启动循环：

```bash
/ralph-loop "Build a REST API for todos. Requirements: CRUD operations, input validation, tests. Output <promise>COMPLETE</promise> when done." \
  --completion-promise "COMPLETE" --max-iterations 50
```

`--completion-promise` 只进行精确字符串匹配，不能表示“成功或受阻”两种状态。因此 `--max-iterations` 是必要保护。prompt 还应说明持续受阻时的处理方式，例如“15 轮后仍未完成就记录阻塞原因、已尝试的方案和建议的替代路径”。使用 `/cancel-ralph` 停止循环。

### Prompt 的完成条件

- 完成标准应可自动验证。例如，“做一个好的 todo API”无法直接验证；“全部 CRUD 端点可用、输入校验就位、测试通过且覆盖率大于 80%”可以验证。
- 大需求应拆为有顺序的阶段，每个阶段具备自己的验证方式。
- prompt 应定义修正流程：编写测试、实现、运行测试，失败后修复，通过后再提交。
- 应始终设置最大轮次。
- 多个来源建议保持 prompt 简短；YC 黑客松案例提供了相关数据。

## 适用条件和限制

Ralph 更适合以下情况：

- Greenfield 新项目。Huntley 的判断是 Ralph 可完成 greenfield 项目约 90% 的工作，其余部分由人完成。
- 任务有可自动验证的完成标准，至少具备类型检查、测试套件或 lint 之一。
- 代码移植与语言转换。源实现可作为规格，且验证方式通常已存在。
- Spec-to-code 任务，需求能够预先写成清晰规格。
- 需求可拆为独立小任务，且单个任务能放入一个上下文窗口。

以下情况的风险较高：

- 存量复杂代码库。Huntley 表示："There's no way in heck would I use Ralph in an existing code base"。其中有许多隐式约束且影响范围难以界定，自动门禁未覆盖的修改可能造成静默破坏。
- 完成标准不清晰、依赖人类设计判断的任务。
- 一次性操作和生产事故调试。这类任务需要针对性排查，循环可能扩大误操作影响。
- 缺少测试和类型系统的项目。应先补充自动门禁。

## 案例：repomirror 的夜间移植

YC Agents 黑客松中，repomirror 团队将 Claude Code 放入 while 循环运行一夜，最终产生约 1100 个 commit，并完成六个仓库的移植（[完整报告](https://github.com/repomirrorhq/repomirror/blob/main/repomirror.md)）。其中一个任务是将 Python 编写的 browser-use 移植到 TypeScript。prompt 只有几行：移植并维护仓库、每次修改文件后提交、使用 `.agent/` 目录存放 TODO。他们还基于文档的 `llms-full.txt` 进行了两个 spec-to-code 实验。总推理成本不足 $800，每个 Sonnet agent 每小时约 $10.50。

报告记录了以下现象和限制：

- Prompt 从 103 词扩展到 1500 词后，agent 变慢且输出质量下降；恢复为 103 词后恢复正常。该案例表明，增加指令会降低每轮执行的一致性。
- 完成 AI SDK 的 Python 移植后，agent 额外加入了原版没有的 Flask、FastAPI 集成和多种 schema 校验器支持。无人值守任务需要在 prompt 中明确仓库边界。
- 多数 agent 完成后继续编写额外测试或完善 TODO；一个 agent 检测到死循环后使用 `pkill` 终止自身进程。沙箱应限制进程权限。
- 两个移植仓库都曾报告完成，但 demo 实际无法运行，团队最终以交互方式人工完成收尾。Ralph 缩短了从 0 到 90% 的时间，最后 10% 仍由人工处理。


## 案例：aimo 从 Ralph 切换到 superpowers

[aimo](https://github.com/ximing/aimo) 是一个 memo 应用，也是我使用 Ralph 规模最大的一次实践。早期大部分功能由 Ralph 批量实现；项目进入维护迭代阶段后，我切换到 superpowers。这一变化对应任务性质的变化。

Ralph 阶段约持续三周，从 2026 年 2 月中旬到 3 月初，产出二十个 PRD 和十余个 PR，包括 memo 双向链接、Electron 客户端、Chrome 扩展、AI 会话存储，以及 LanceDB 到 MySQL 的存储迁移。前文 `progress.txt` 中的九个 story 也在此阶段完成。每个 PRD 使用独立分支，完成后归档到 `tasks/archive/`。期间曾试用官方 ralph-wiggum 插件，后来重新使用自建脚本；aimo 版本将循环模型固定为 MiniMax-M2.5。

切换的原因是项目阶段发生变化。Ralph 阶段最后完成的回收站，与 superpowers 阶段完成的通知中心规模相近，但工作内容不同。前三周接近 greenfield，功能可以连续实现；之后工作转为增加接口越权检查与 SQL 注入防护、将 N+1 查询改为批量查询、补充单元测试。这些任务属于存量代码库的加固迭代，需要在关键环节由人确认。该经历也验证了 Huntley 对存量复杂代码库的限制。

两种方法共享需求规格、任务拆分和逐步验证等做法，区别在外层驱动和人工介入点。Ralph 的 `prd.json` 将需求、验收标准和完成状态写成机器可读清单，由人在循环外进行事后审查；superpowers 将 design 和 plan 分为两份面向人的文档（`docs/superpowers/`，命名沿用 Ralph 阶段的 specs），并在设计和计划阶段由人确认后再执行。可复用的资产是 specs、计划文档和 `progress.txt` 中的 Codebase Patterns，而非循环脚本。详细对比见 [Ralph 与 Superpowers 的区别](/post/2026/02.02-superpowers/)：两者的需求都由人事前共创，差异在执行阶段——Ralph 把人放在循环外，通过调整 prompt 和重跑 planning 间接纠偏；Superpowers 把人放在流程内，在设计、计划和熔断升级节点确认。前者适用于 greenfield 阶段的连续功能实现，后者适用于存量维护中需要人类判断的任务。

## 局限和风险

- 禁止 placeholder 实现不能完全消除桩代码。可定期运行专门搜索 TODO、占位实现和最小实现的 planning 迭代，并将结果加入后续任务。
- 循环可能留下无法编译的代码库。Huntley 明确提到这一情况。此时需要人判断是使用 `git reset --hard` 重跑，还是通过新的 prompt 修复。
- 成本需要控制。黑客松案例中每个 agent 每小时约 $10.50；无人值守运行会产生实际账单，应预先设置 max-iterations 和沙箱预算告警。
- 仍需资深工程师。CURSED 的规格错误由 Huntley 发现并修正，prompt 调优也依赖诊断失败模式的能力。Huntley 表示，工具不需要工程师或可以 100% 交付的说法是在 "peddling horseshit"。
- 多个 Ralph 并行的模式尚不成熟。Huntley 将多代理通信类比为非确定性微服务间调用，并认为当前以单个循环和一个主上下文调度子代理的形式更可靠。

## 参考资料

- [Ralph Wiggum as a "software engineer" — Geoffrey Huntley](https://ghuntley.com/ralph/)
- [The Ralph Playbook — how-to-ralph-wiggum](https://github.com/ghuntley/how-to-ralph-wiggum)
- [Claude Code ralph-wiggum 官方插件](https://github.com/anthropics/claude-code/tree/main/plugins/ralph-wiggum)
- [We Put a Coding Agent in a While Loop and It Shipped 6 Repos Overnight — repomirror](https://github.com/repomirrorhq/repomirror/blob/main/repomirror.md)
