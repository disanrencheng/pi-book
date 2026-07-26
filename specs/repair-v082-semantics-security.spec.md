spec: task
name: "修复v082-跨章语义与安全边界"
inherits: project
tags: [chapter, repair, semantics, security, v082]
satisfies: [REQ-SYNC]
depends: [repair-v082-facts-api]
estimate: 2d
---

## Intent

在事实与 API 入口收敛后，统一全书对运行时语义和安全边界的解释。每条口径都绑定
pi-mono v0.82.1 commit `b4f29368` 的源码锚点；已经正确的段落只作为验收依据，
错误、冲突或绝对化表述才做最小修订。

## Decisions

- agent loop 无 module-level 持久状态；调用方提供独立 context 时可重复调用和组合，
  但它会直接修改传入 context 并执行 LLM/tool I/O，不称为纯函数、纯计算或
  对共享 context 可重入（`packages/agent/src/agent-loop.ts:155-220,281-370`）
- 包依赖保持无环且由基础包流向产品包；产品层可以直接依赖多个基础包，
  不声称每层“只知道下一层”（`packages/agent/package.json:31-36`；
  `packages/coding-agent/package.json:41-45`；`packages/server/package.json:42-44`）
- 生产 coding-agent 使用 `Agent`（`packages/coding-agent/src/core/sdk.ts:294`）；
  `AgentHarness` 是并行演进、当前仅见于 agent 测试/docs 的厚壳
  （`packages/agent/src/harness/agent-harness.ts:171-199`；
  `packages/agent/docs/models.md:790-794`）
- Skill 名称冲突使用 first-wins，用户级默认目录先于项目级；
  其他资源的覆盖策略不得套用到 Skill
  （`packages/coding-agent/src/core/skills.ts:394-425,430-432`）
- `!!command` 执行并持久化 bash 结果，但从 LLM context 排除
  （`packages/coding-agent/src/modes/interactive/interactive-mode.ts:2779-2790,5921-5987`；
  `packages/coding-agent/src/core/agent-session.ts:2801-2823,2844-2857`；
  `packages/coding-agent/src/core/messages.ts:26-40,148-194`）
- sequential 工具批次完整串行；默认 parallel 批次串行 prepare、并发 execute/finalize，
  `tool_execution_end` 按完成顺序发射，结果消息保持源顺序
  （`packages/agent/src/types.ts:37-40,258-263`；
  `packages/agent/src/agent-loop.ts:411-553`）
- 会话分支只移动 append-only transcript 的 leaf pointer，不回滚工作区文件
  （`packages/coding-agent/src/core/session-manager.ts:1296-1303,1354-1365`）
- compaction 对活动 context 有损、原始 JSONL entry 保留；后续压缩迭代更新
  `previousSummary`；“遗漏可能累积”必须标为基于控制流的设计限制，而非源码保证
  （`packages/coding-agent/src/core/session-manager.ts:410-469,1096-1118`；
  `packages/coding-agent/src/core/compaction/compaction.ts:621-658,718-788`）
- provider stream 的 `partial` 表示发射时的累计消息状态，但协议不保证对象身份或
  不可变性；Anthropic adapter 复用可变 accumulator，faux provider 发射浅拷贝。
  保存历史快照或恢复能力需要消费者显式 clone/serialize，终态由 `done`/`error` 携带
  （`packages/ai/src/types.ts:483-503`；
  `packages/ai/src/api/anthropic-messages.ts:567-703`；
  `packages/ai/src/providers/faux.ts:316-390`）
- Skill 被选择并读取后会进入对话，属于 prompt injection 与间接工具执行的信任边界，
  不能描述为零风险（`packages/coding-agent/src/core/skills.ts:277-317`；
  `packages/coding-agent/src/core/agent-session.ts:1297-1315`）
- AGENTS.md/CLAUDE.md 注入 system prompt 后仍是软约束，不等于文件系统权限；
  硬阻断依赖 hook、sandbox 或宿主策略
  （`packages/coding-agent/src/core/system-prompt.ts:144-151`；
  `packages/coding-agent/src/core/agent-session.ts:468-517`）
- 项目 AGENTS.md/CLAUDE.md 从 cwd 及其祖先链发现，不位于 `{cwd}/.pi/AGENTS.md`
  （`packages/coding-agent/src/core/resource-loader.ts:67-120`）
- read 先把完整文件读入 `Buffer` 再截断，因此截断保护模型 context，
  不保证读取 I/O 或进程内存上限
  （`packages/coding-agent/src/core/tools/read.ts:238-288`）
- file mutation queue 只串行化同一进程、同一规范化路径的调用，
  不覆盖跨进程修改或外部 TOCTOU
  （`packages/coding-agent/src/core/tools/file-mutation-queue.ts:4-60`）

## Boundaries

### Allowed Changes
- pi-book/outline.md
- pi-book/src/ch01-prologue.md
- pi-book/src/ch02-packages.md
- pi-book/src/ch03-reading-map.md
- pi-book/src/ch06-event-stream.md
- pi-book/src/ch08-agent-loop.md
- pi-book/src/ch09-tool-execution.md
- pi-book/src/ch10-agent-class.md
- pi-book/src/ch11-session-tree.md
- pi-book/src/ch12-compaction.md
- pi-book/src/ch13-config-layers.md
- pi-book/src/ch14-system-prompt.md
- pi-book/src/ch16-skills.md
- pi-book/src/ch17-resource-loader.md
- pi-book/src/ch20-edit-tool.md
- pi-book/src/ch21-read-tool.md
- pi-book/src/ch22-bash-tool.md
- pi-book/src/ch25-editor.md
- pi-book/src/ch30-minimal-core.md
- pi-book/src/ch31-contrarian-choices.md
- pi-book/src/ch32-boundaries.md
- pi-book/src/appendix.md
- pi-book/.agent-spec/reviews/repair-v082/semantics-security.decisions.json
- pi-book/.agent-spec/reviews/repair-v082/semantics-security.lifecycle.json
- pi-book/.agent-spec/reviews/repair-v082/semantics-security.report.json
- pi-book/docs/superpowers/reviews/2026-07-26-pi-book-v082-content-repair-review.md

### Forbidden
- 不要修改 pi-mono 源码
- 不要把源码推论写成源码中的显式保证
- 不要把 prompt 指令、Project Trust 或 `beforeToolCall` 单独描述成完整 sandbox
- 不要把所有资源概括为相同的覆盖规则
- 不要删除已经正确的 AgentHarness、工具并发或会话树分析

## Out of Scope

- README/旧 API/统计修复
- 外部框架比较
- Mermaid、代码块长度和 ch27-29 历史尾注

## Completion Criteria

<!-- lint-ack: error-path — 本任务为书稿事实核查与跨章一致性修复，场景均为静态核查型断言 -->

场景: agent loop 与分层依赖口径统一
  测试: grep_loop_and_layering_semantics
  当 检查 outline、ch01-03、ch08、ch10、ch30-31
  那么 agent loop 描述为无 module-level 持久状态且对独立 context 可重复调用和组合
  并且 不再称为纯函数或纯计算
  并且 不声称共享同一可变 context 时可重入
  并且 分层依赖描述为无环而非“只知道下一层”

场景: Agent 与 AgentHarness 采用状态准确
  测试: review_agent_harness_adoption
  当 检查 ch10、ch30-32 与附录
  那么 生产 coding-agent 使用薄壳 `Agent`
  并且 `AgentHarness` 标为并行演进且当前只见于 agent 测试/docs

场景: Skill 冲突规则与其他资源分开
  测试: review_skill_collision_first_wins
  当 检查 ch16、ch17 与相关图表
  那么 Skill 名称冲突明确为 first-wins
  并且 ch17 不再声称 npm/project Skill 后加载覆盖用户级 Skill
  并且 其他资源保留各自经过源码核实的覆盖规则

场景: bash 双叹号语义一致
  测试: grep_double_bang_semantics
  当 检查 ch22、ch25、ch26 与附录
  那么 `!!command` 描述为执行并持久化但从 LLM context 排除
  并且 不再描述为重复上一条 bash 命令

场景: 工具、分支与压缩边界准确
  测试: review_tool_branch_compaction_semantics
  当 检查 ch09、ch11、ch12 与结论章
  那么 parallel 与 sequential 工具批次的阶段顺序准确
  并且 parallel 模式的结束事件按完成顺序而结果消息按源顺序
  并且 会话分支明确不回滚工作区文件
  并且 compaction 明确区分活动 context 与原始 JSONL
  并且 多次压缩不再保证摘要质量不会递减
  并且 previousSummary 迭代带来的累计遗漏风险明确标为设计限制或推论

场景: partial 不再冒充独立快照
  测试: review_partial_mutable_accumulator
  当 检查 ch06 与其跨章引用
  那么 `partial` 描述为 provider 流在发射时的累计消息状态
  并且 明确协议不保证对象身份或不可变性
  并且 Anthropic 同对象复用与 faux 浅拷贝被区分为具体实现
  并且 不再声称 EventStream 自带持久化或崩溃恢复

场景: prompt 指令与硬权限边界分离
  测试: review_prompt_security_boundary
  当 检查 ch13-17、ch22、ch30-32 与附录
  那么 Skill 不再描述为零风险
  并且 AGENTS.md/CLAUDE.md 明确是注入 prompt 的软约束
  并且 项目 context 文件来自 cwd 及祖先链而不是 `{cwd}/.pi/AGENTS.md`
  并且 硬阻断明确依赖 hook、sandbox 或宿主策略

场景: read 与 edit 的保护范围不过度承诺
  测试: review_read_edit_limitations
  当 检查 ch20、ch21 与结论章
  那么 read 截断只保证限制进入模型的内容
  并且 file mutation queue 只保证进程内同路径串行化
  并且 不声称覆盖跨进程竞态或外部 TOCTOU

场景: 新增源码锚点均经过 release commit 实测
  测试: review_semantics_source_anchors
  当 盘点本合约新增或修改的每处 `packages/...:line` 引用
  那么 每处引用都已在 commit `b4f29368` 的对应文件中打开核验
  并且 源码未显式保证的结论明确标为推论或限制
