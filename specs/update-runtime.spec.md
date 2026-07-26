spec: task
name: "更新-Agent Runtime 变更同步 (ch08-10)"
inherits: project
tags: [chapter, update, runtime]
estimate: 1d
depends: [update-ai]
---

## Intent

同步 packages/agent 从 v0.66.1 到 v0.79.7 的设计级变更到第 8-10 章。重点是循环
退出条件的 terminate 语义、turn 级停止/热切换回调、per-tool 执行模式，以及一个
全新的高层壳 AgentHarness 子系统——它把会话/压缩/skills/执行环境内聚进 agent 包，
与现有 Agent 类两条路线并存（生产仍用 Agent，须如实标注为演进中的并行抽象）。

## Decisions

- ch08: 内层循环退出条件从 `toolCalls.length > 0` 改为 `!terminate`；补 terminate 工具提示（v0.69.0）；AgentLoopConfig 新增 shouldStopAfterTurn（v0.72.0）与 prepareNextTurn（turn 间切换 model/thinkingLevel）
- ch09: AgentTool 新增 executionMode（sequential|parallel）；混批含任一 sequential 工具则整批降级串行；executeToolCalls 返回 { messages, terminate }；澄清生命周期事件按完成顺序急切发射、tool-result 工件按源顺序持久化（v0.68.1）
- ch10: Agent 类本身分析仍成立，但需声明存在第二条路径 AgentHarness（packages/agent/src/harness/，含 SessionRepo/session tree/分支、内置 compaction、skills、ExecutionEnv）；标注生产 coding-agent 仍用 new Agent，AgentHarness 当前仅在 agent 包自身测试/docs 使用
- 新增主题: AgentHarness 的类型化钩子链（on(type,handler)）、注册工具 vs 激活工具二层模型、Result<T,E> 错误模型与 AgentHarnessPhase
- 订正 ch10:356 声明合并示例模块名为 @earendil-works/pi-agent-core

## Boundaries

### Allowed Changes
- pi-book/src/ch08-agent-loop.md
- pi-book/src/ch09-tool-execution.md
- pi-book/src/ch10-agent-class.md

### Forbidden
- 不要修改 pi-mono 源码
- 不要把 AgentHarness 写成已取代 Agent（生产仍用 Agent）

## Out of Scope

- coding-agent 层的会话树/压缩产品化内容（见 update-coding-agent）

## Completion Criteria

场景: ch08 退出条件含 terminate
  测试: review_ch08_terminate
  假设 ch08-agent-loop.md 已更新
  当 检查内层循环退出条件
  那么 说明退出由工具结果的 terminate 提示驱动
  并且 不再仅以 "toolCalls.length > 0" 作为继续条件

场景: ch08 补 turn 级回调
  测试: review_ch08_turn_hooks
  当 检查 AgentLoopConfig 字段清单
  那么 列出 shouldStopAfterTurn 与 prepareNextTurn

场景: ch09 含 per-tool 执行模式与降级规则
  测试: review_ch09_execution_mode
  假设 ch09-tool-execution.md 已更新
  当 检查工具执行策略
  那么 说明 executionMode 的 per-tool 覆盖
  并且 说明混批含一个 sequential 工具则整批串行

场景: ch09 澄清事件与工件的两种顺序
  测试: review_ch09_event_ordering
  当 检查并行模式的事件发射描述
  那么 区分生命周期事件按完成顺序、tool-result 按源顺序
  并且 不再统一描述为"都按源顺序"

场景: ch10 声明 AgentHarness 并行路径
  测试: review_ch10_harness
  假设 ch10-agent-class.md 已更新
  当 检查章节
  那么 介绍 AgentHarness 子系统及其内聚的会话/压缩/skills/执行环境
  并且 明确标注生产 coding-agent 仍使用 Agent 类

场景: ch10 声明合并示例模块名正确
  测试: grep_ch10_module_name
  当 在 ch10 检查 declare module 示例
  那么 使用 "@earendil-works/pi-agent-core"
  并且 不再使用 "@mariozechner/agent"
