# pi-book v0.82.1 内容修复设计

## 背景

pi-book 当前工作稿已经完成一轮从 pi-mono v0.79.7 到 v0.82.1 的同步，并完成三视角评审后的 23 项修复。二次审查确认，当前章节的版本尾注模板、SUMMARY 标点、前言当前版本提示、appendix/ch32 的两处旧 `api-registry.ts` 路径口径等问题已经修复，不能再作为待办重复执行。ch27–29 的尾注虽然模板已统一，历史语义仍需在本轮闭环。

仍未收敛的问题主要位于 README、`outline.md`、结论章和若干跨章定义：旧 API 与错误统计仍有残留，agent loop、Skill 冲突、流式 `partial`、compaction 等概念存在相互冲突或过强结论，项目级写作合约也没有覆盖“每章至少一张 Mermaid”等新验收要求。

本次修复直接叠加在 `update-v0.79-sync` 分支现有未提交稿上。现有修改和未跟踪的治理产物都属于当前工作成果，必须保留；不得切换到干净分支重做，不得修改上级 pi-mono 源码，也不得提交。

## 目标

1. 使全书当前事实与 pi-mono v0.82.1、commit `b4f29368` 一致。
2. 保留 ch27–29，但统一标注为 pi-mono v0.66.1、commit `c779c14e` 的历史案例。
3. 消除跨章硬冲突，使同一概念在正文、结论、附录和项目合约中只有一种当前口径。
4. 修正会误导读者实现、运维或安全判断的 API、运行时语义和示例。
5. 满足修订后的项目级章节图示、代码块长度、版本说明和源码引用约束。

## 非目标

- 不重新设计全书目录，不删除章节。
- 不把历史产品重新描述为当前仓库组件。
- 不追踪 `b4f29368` 之后的 pi-mono 主线变化。
- 不修改 pi-mono 源码、测试、依赖或生成文件。
- 不借修复之机扩写无关主题。

## 方案取舍

本次采用“全书收敛”，不采用以下两个极端方案：

- **只修审查命中的句子**：改动最小，但同一错误口径仍可能留在结论、图表、附录或 README 中，无法证明全书一致。
- **重新规划篇章并重写**：可以彻底重组主线，但会放大当前工作树的冲突面，也超出“修复”授权。

全书收敛以概念和事实为修复单元：先建立唯一口径，再搜索所有重复表述并逐处校正。代价是涉及文件较多；收益是每个改动都能由跨章搜索和机械检查验收，同时保留现有章节与写作成果。

## 事实基线

| 内容类型 | 权威基线 | 使用规则 |
|---|---|---|
| 当前 pi-mono 架构与 API | `b4f29368`，v0.82.1 | ch01–26b、ch30–32、附录的当前事实 |
| 已移出包 | `c779c14e`，v0.66.1 | ch27 web-ui、ch28 mom、ch29 pods |
| 当前外部生态比较 | 执行时读取的官方文档 | 独立合约、标注核验日期；不作为内部事实修复的阻塞项 |
| 书稿结构要求 | `CLAUDE.md`、`specs/project.spec.md` | 两者冲突时先修正 project spec，再按统一规则验收 |

源码引用优先使用完整的 `packages/.../file.ts:line`。章节内连续讨论同一文件时允许缩写为 `file.ts:line`，但首次出现必须给出完整路径。

## 当前工作稿重盘点

审计快照日期为 2026-07-26。pi-book 子仓库当前有 33 个 tracked 修改文件（`+1300/-1231`）和 31 个未跟踪文件；后者包括 5 份 `update-v082-*` 合约、`REQ-SYNC`、live wiki 和本设计。执行时以文件清单为准，不依赖固定数量做边界判断。

| 状态 | 内容 |
|---|---|
| 已完成，不重复修改 | preface 当前版本提示；当前章节版本尾注模板；SUMMARY 副标题标点；ch01/ch02 旧 `api-registry.ts` 现状句；appendix:39 与 ch32:149 旧路径 |
| 仍待修复 | README/outline 版本、目录与旧 API；ch02 “130+ 个工具”；纯函数/纯计算表述；ch31 内建 Agent 工具；ch17 Skill 覆盖规则；ch25 `!!`；ch06 `partial`；ch12 重复压缩；ch32/appendix 二次开发入口 |
| 新增结构门禁 | 7 章缺 Mermaid；全语言扫描得到 7 个 fenced code block 超过 40 行（当前均为 TypeScript）；审计时 project spec 漏登记，现已作为治理前置项补齐 |
| 延后到独立合约 | LangChain、Vercel AI SDK 等外部生态比较 |

执行前再次运行相同扫描；如果某项已被其他会话修复，记录为通过，不重复改写。

## 修复主线

### 1. 事实与 API 收敛

受影响文件主要包括 README、ch01、ch02、ch04–10、ch13–19、ch25、ch31、ch32 和附录。

- pi-ai 的规范主线统一为 `Models`、`createModels`、`createProvider` 与 `@earendil-works/pi-ai/providers/*`。`@earendil-works/pi-ai/compat` 在 v0.82.1 仍是公开的临时迁移入口，`registerApiProvider` 只能明确标为 compat API，不能谎称带有源码中不存在的 `@deprecated` 标记；已删除的 `api-registry.ts` 才只能出现在历史上下文中（`packages/ai/package.json:18-20`；`packages/ai/src/compat.ts:1-10,126-148`；`packages/coding-agent/src/core/sdk.ts:1-3`）。
- ch02 使用 v0.82.1 实际 package scripts，删除“130+ 个工具”之类由文件数误推的统计。
- `Model`、`EventStream`、tool finalize、Extension prompt hook、Skill 加载优先级等示例必须与基线类型和控制流一致。
- ch31 不再声称 coding-agent 内建 Agent 工具。当前 sub-agent 只能作为 extension/组合模式说明，并采用可编译的当前 API 或明确标注为概念伪代码。
- 外部框架比较交给独立合约，限定能力范围、使用执行时的官方来源，不保留“没有循环引擎”一类未经核验的绝对判断。

### 2. 跨章语义统一

以下概念建立全书统一口径。证据行号固定在 `b4f29368`；“可能累积遗漏”等为基于控制流的设计推论，正文必须明确标为限制或推论，不能伪装成源码注释。

| 概念 | 统一表述 | `b4f29368` 证据 |
|---|---|---|
| agent loop | 无 module-level 持久状态；调用方提供独立 context 时可重复调用和组合。它会直接修改传入 context 并执行 LLM/tool I/O，因此不是纯函数，也不保证共享同一 context 时可重入 | `packages/agent/src/agent-loop.ts:155-220,281-370` |
| 分层依赖 | 依赖保持无环并从基础包流向产品包；产品层可直接依赖多个基础包，不声称“只知道下一层” | `packages/agent/package.json:31-36`；`packages/coding-agent/package.json:41-45`；`packages/server/package.json:42-44` |
| Agent / AgentHarness | 生产 coding-agent 使用薄壳 `Agent`；`AgentHarness` 是并行演进的厚壳路线，coding-agent 尚未采用 | `packages/coding-agent/src/core/sdk.ts:2,294`；`packages/agent/src/harness/agent-harness.ts:171-199`；`packages/agent/docs/models.md:790-794` |
| Skill 冲突 | Skill 按名称 first-wins；用户级默认目录先于项目级，冲突记录 diagnostic 而不覆盖。不得把其他资源的覆盖规则概括成统一规则 | `packages/coding-agent/src/core/skills.ts:394-425,430-432` |
| `!!command` | 执行并持久化 bash 结果，但 `convertToLlm` 从 LLM context 排除 | `packages/coding-agent/src/modes/interactive/interactive-mode.ts:2779-2790,5921-5987`；`packages/coding-agent/src/core/agent-session.ts:2801-2823,2844-2857`；`packages/coding-agent/src/core/messages.ts:26-40,148-194` |
| 工具并发 | sequential 完整串行；默认 parallel 模式先串行 prepare，再并发 execute/finalize，最终消息保持源顺序，结束事件按完成顺序发射 | `packages/agent/src/agent-loop.ts:411-425,433-553` |
| 会话分支 | 只移动 append-only transcript 的 leaf pointer；不修改旧 entry，也不回滚工作区文件 | `packages/coding-agent/src/core/session-manager.ts:1296-1303,1354-1365` |
| compaction | 活动 context 使用有损摘要，JSONL 原始 entry 保留；后续压缩用 `previousSummary` 迭代更新。“遗漏可能累积”是据此得出的设计限制，不是源码的显式保证 | `packages/coding-agent/src/core/session-manager.ts:410-469,1096-1118`；`packages/coding-agent/src/core/compaction/compaction.ts:621-658,718-788` |
| `partial` | 事件协议提供发射时的累计消息状态，但不保证对象身份或不可变性；Anthropic adapter 复用可变 accumulator，faux provider 发射浅拷贝。消费者若要保存历史快照或恢复，必须显式 clone/serialize；终态由 `done`/`error` 携带 | `packages/ai/src/types.ts:483-503`；`packages/ai/src/api/anthropic-messages.ts:567-703`；`packages/ai/src/providers/faux.ts:316-390` |

上述口径必须同步检查首次定义章、后续引用章、ch30–32 和附录。已经正确的段落只作为验收锚点，不为制造 diff 而改写。

### 3. 安全与边界说明

- Skill 是纯文本，但被选择并读取后会进入对话并影响后续工具选择，仍属于 prompt injection 和间接工具执行的信任边界，不能描述为“零风险”（`packages/coding-agent/src/core/skills.ts:277-317`；`packages/coding-agent/src/core/agent-session.ts:1297-1315`）。
- AGENTS.md/CLAUDE.md 是注入 system prompt 的软约束，不等于文件系统权限。硬阻断依赖 `beforeToolCall`、sandbox 或宿主策略（`packages/coding-agent/src/core/resource-loader.ts:67-120`；`packages/coding-agent/src/core/system-prompt.ts:144-151`；`packages/coding-agent/src/core/agent-session.ts:468-517`）。
- 项目 AGENTS.md 位于当前工作目录及其祖先路径，不位于 `{cwd}/.pi/AGENTS.md`；`.pi/` 保存 settings 和项目资源（`packages/coding-agent/src/core/resource-loader.ts:67-120`）。
- read 的截断保护限制进入模型的内容，但当前实现先将整个文件读为 `Buffer` 再截断，不保证大文件读取本身的 I/O 或进程内存上限（`packages/coding-agent/src/core/tools/read.ts:238-288`）。
- edit 的队列只保证同一进程、同一规范化路径的工具调用串行化，不覆盖跨进程修改或外部 TOCTOU（`packages/coding-agent/src/core/tools/file-mutation-queue.ts:4-60`）。

### 4. 版本闭环与历史章节

- README、preface、ch01 和当前架构章节的版本尾注统一说明：核心设计起点为 v0.66.x，当前对照版本为 v0.82.1。ch27–29 是明确例外，只声明 v0.66.1 历史基线。
- ch30–32 和附录必须按 v0.82.1 重新收敛，不再把 mom、web-ui、pods 当作当前产品入口。
- ch27–29 保留在主目录中，章首和章尾都使用历史时态，并说明移出时间、当前继任情况以及保留原因。
- 当前二次开发入口改为 provider/Models、SDK、RPC、server、storage/sqlite-node 和 extension；`evals` 只作为 private 内部评测 harness 登记。历史章节只能作为设计案例引用。
- README 的源码仓库链接更新到当前官方仓库，并同步当前章节分篇和 SDK 章节。
- `outline.md`、`src/SUMMARY.md`、`CLAUDE.md`、`specs/project.spec.md` 的路径、章节和验收口径保持一致。

## 书稿规范修复

“每章至少一张 Mermaid”是本轮新增的 project-level 规则，不冒充既有要求；`specs/project.spec.md` 已在创建 repair 合约前显式登记，structure-history 只验证并执行正文补齐，不再拥有该治理前置项。

1. 将当前扫描出的 7 个超长 fenced code block 拆成关键片段与文字说明：ch08（93 行）、ch09（52 行）、ch10（51 行）、ch13（64/46 行两块）、ch17（41 行）、ch31（58 行）。当前 7 个违规块恰好全部是 TypeScript；门禁扫描所有语言的 fenced block，不能通过换语言标签或删除必要语义规避限制。
2. 为当前缺图的 ch21、22、23、25、28、29、32 各补一张承载核心关系的 Mermaid 图。
3. ch01、ch03、ch24、ch26b 若没有独立“取舍分析”标题，至少确保关键决策明确列出得到与放弃；不强制机械重排已有叙事。
4. ch30–32 的关键结论补现行源码锚点或明确回指已有技术章节。
5. 保留附录现有两条跨章 E2E trace，并更新其中的当前 API 与路径。

代码块与 Mermaid 清单在执行前后都重新扫描；上述文件名是当前快照，不是跳过新发现问题的白名单。

## 合约分解

正文修复不得直接从本设计开工。先创建并审查以下任务合约，全部 `satisfies: [REQ-SYNC]`：

| 合约 | 职责 | 依赖 | 是否阻塞内部修复 |
|---|---|---|---|
| `repair-v082-facts-api.spec.md` | README/outline、旧 API、类型/hook 示例、统计、当前二次开发入口 | 无 | 是 |
| `repair-v082-semantics-security.spec.md` | 九条语义口径与五条安全边界 | facts-api | 是 |
| `repair-v082-structure-history.spec.md` | ch27–29 历史闭环、Mermaid、全部代码块、版本/定位、wiki | semantics-security | 是 |
| `repair-v082-ecosystem-comparison.spec.md` | 用带日期的官方来源重做外部框架比较 | structure-history | 否，可延期 |

前三份构成内部修复关键路径。外部比较引入联网核验与独立事实源，不拖住内部事实、语义和结构修复；若执行，必须在 structure-history 之后开始，避免并发改写 outline、ch01 和 ch30–32。四份 repair 合约是既有五份按包分组的 `update-v082-*` 合约之上的纠错层，不替代同一 release 区间的 CHANGELOG/包级同步证据。

每份合约在交付执行前必须满足：

- `agent-spec parse` 解析出非零场景；
- `agent-spec lint --min-score 0.7` 通过；
- review 型 selector 明确交给 fact-checker、tech-reviewer、structure-editor，不把 `verify` 的 `skip` 当作通过；
- 执行中若发现边界不足，先更新合约并重新 lint，再改正文。

### 验证闭环与脏工作树

pi-book 当前工作树包含同轮既有修改，不能用整个 worktree 作为单份合约的边界证据。以下 agent-spec 命令统一从上级 pi-mono 仓库根目录运行：spec 路径使用 `pi-book/specs/...`，`--code pi-book`，每个实际触碰路径使用带 `pi-book/` 前缀的 `--change <path>`。这样 caller 请求写入 `pi-book/.agent-spec/`，边界也与合约中的 allowed paths 一致。不得用 `--change-scope worktree` 吞入其他会话的修改。

每份实际执行的合约按以下协议关闭：

1. 先运行 `mkdir -p pi-book/.agent-spec/reviews/repair-v082 pi-book/docs/superpowers/reviews`，确保后续 stdout 重定向目标存在。
2. 运行 `agent-spec lifecycle pi-book/specs/<spec> --code pi-book --ai-mode caller --review-mode strict --format json`，附带显式 `--change pi-book/<path>` 清单，把 stdout 原样保存为 `pi-book/.agent-spec/reviews/repair-v082/<contract>.lifecycle.json`，并把未被机械检查覆盖的场景导出为 caller 请求。此时因 caller 尚未 resolve，命令可以非零退出；但 lifecycle JSON 中 boundary 必须为 PASS，failed 必须为 0。
3. fact-checker、tech-reviewer、structure-editor 分别给出 verdict；综合决定写入 `pi-book/.agent-spec/reviews/repair-v082/<contract>.decisions.json`，三份原始 verdict 与 Critical 计数写入 `pi-book/docs/superpowers/reviews/2026-07-26-pi-book-v082-content-repair-review.md`。
4. 运行 `agent-spec resolve-ai pi-book/specs/<spec> --code pi-book --decisions pi-book/.agent-spec/reviews/repair-v082/<contract>.decisions.json --format json`，把 stdout 原样保存到 `pi-book/.agent-spec/reviews/repair-v082/<contract>.report.json`。
5. 独立 post-resolve gate 同时读取 lifecycle 与 final report：lifecycle 的 boundary = PASS 且 failed = 0；final report 的 failed、skipped、uncertain 均为 0；三视角报告 Critical = 0。该 gate 位于合约场景之外，避免让合约用尚未生成的自身报告证明自身。

`pi-book/.agent-spec/pending-ai-requests.json` 是共享文件；必须逐份执行 lifecycle → review → resolve-ai，当前合约 resolve 并清理 pending 请求后才能开始下一份，不能并行生成 caller 请求。

structure-history 阶段创建并运行 `node pi-book/scripts/check-chapter-structure.mjs`，机械核查全部 `pi-book/src/ch*.md` 的定位锚点、Mermaid、版本演化说明和所有 fenced code block 的 40 行上限。事实与语义场景仍需结合固定 commit 源码审查，不能用 grep 代替语义判断。

## 修改边界

所有正文修改限定在 `pi-book/**`。编辑前按文件读取完整内容，并基于现有工作稿做最小补丁。任何看似有意的功能、章节或案例均保留；若发现必须删除才能修复的新冲突，应停止并请求用户确认。

本轮不创建提交。`git add`、`git commit`、分支切换、stash 和清理命令均不执行。

## 验收

### 事实验收

- 所有当前源码断言可在 `b4f29368` 中定位。
- ch27–29 的源码断言可在 `c779c14e` 中定位，且不冒充当前能力。
- 外部比较若本轮执行，必须带核验日期和官方来源；若延期，不阻塞前三份内部修复合约关闭。
- 审查报告中的事实、技术和结构问题逐项关闭。

### 一致性验收

- 全书不再把 agent loop 称为纯函数或纯计算。
- Skill、`!!`、工具并发、分支、compaction、AgentHarness 不再出现互相冲突的定义。
- README、前言、结论章、附录和 project spec 的版本及目录口径一致。

### 机械验收

- project spec 与四份 repair 合约的 `agent-spec parse`、`agent-spec lint --min-score 0.7` 通过，且 task 合约场景数非零。
- project spec 与前三份内部 repair 合约均完成 caller/resolve-ai 闭环，最终报告的 failed、skipped、uncertain 均为 0；ecosystem-comparison 若执行也满足同一门禁，若延期则明确保持 open，不阻塞内部闭环。
- review 型场景由 fact-checker、tech-reviewer、structure-editor 三视角给出明确 verdict，Critical 为 0；`agent-spec verify` 的 `skip` 不计为通过。
- 每章至少一张 Mermaid；所有语言的 fenced code block 均不超过 40 行。
- 每章保留定位锚点与版本演化说明。
- `mdbook build` 成功。
- `git -C pi-book diff --check` 无输出。
- 最终 `git status` 只包含原有工作稿与本轮明确修改的 pi-book 文件，不涉及上级 pi-mono 源码。
- `.agent-spec/wiki/learnings/sync-v0797-v0821.md` 记录本轮二次修复的证据、合约和残余风险。

## 执行顺序

1. 先修正 project spec，重盘点当前工作稿，并为统一口径固定 `b4f29368` 证据。
2. 创建四份 repair 合约，逐份通过 parse/lint；人工接受合约后才进入正文。
3. 按 facts-api → semantics-security → structure-history 执行内部关键路径。
4. ecosystem-comparison 在 structure-history 之后独立执行或延期，不阻塞内部修复，也不与内部合约并发写同一文件。
5. 运行章节结构脚本、`mdbook build`、三视角评审和 diff/status 检查。
6. 把修复结果、证据变化和未决项写入 live wiki。

这个顺序让规范与合约先成为稳定验收基线，再处理正文，最后统一做结构和知识沉淀，避免重复修改已完成项或把未证实口径写回全书。
