spec: task
name: "更新v082-Agent Runtime 同步 (v0.79.7→v0.82.1)"
inherits: project
tags: [chapter, update, runtime, v082]
satisfies: [REQ-SYNC]
depends: [update-v082-ai]
estimate: 1d
---

## Intent

packages/agent 在本区间的主题是「循环引擎与 provider 目录彻底解耦」：v0.81.0 把
底层循环的 `streamFn` 改为必填的 `streamFunction` 以剥离对 pi-ai/compat 的依赖，
v0.81.1 又补回宿主可配置的默认兜底；`AgentHarness` 在 v0.82.0 经历工具模型
breaking 重构。ch08-10 需要按此更新循环签名、turn 间回调与厚壳（AgentHarness）
的描述。

## Decisions

- ch08：`runLoop` 底层签名订正为必填 `streamFunction: StreamFn`（`agent-loop.ts:161`）；
  公共入口以 `streamFn ?? getDefaultStreamFn()` 兜底（`agent-loop.ts:116/141`）；
  新增 `stream-fn.ts` 的 `setDefaultStreamFn`/`getDefaultStreamFn` 宿主级注入机制，
  并把「0.81.0 改必填以剥离 provider 目录依赖、0.81.1 恢复可配置 fallback」写为
  取舍演进案例
- ch08：turn 间回调升级为两个变体 — `prepareNextTurn(signal)` 与
  `prepareNextTurnWithContext(context, signal)`（`agent.ts:448-454` 优先后者），
  注明 abort signal 已下发到 turn 间回调
- ch09：补一句 `AgentToolResult.addedToolNames` 传播到 `ToolResultMessage`
  （v0.80.7）— 工具结果可动态引入新工具并从该转录点起可加载
- ch10：`AgentHarness` 工具模型改写 — `AgentHarnessTool<TContext>` 的
  `execute(..., context)`（`harness/types.ts:99-112`）+ `toolContext` 选项
  （`:955`）；`ExecutionEnv` 仍在（`:373`）但被包装为 `ExecutionToolContext`
  （`harness/tools/tool-context.ts:4`）；内置上下文感知工具
  `harness/tools/{read,write,edit,bash}.ts`
- ch10：会话层区分 `SessionStorage`（`harness/types.ts:498`，含
  `getPathToRootOrCompaction`）与 `SessionRepo`（`:528`）两层；补
  `InMemorySessionStorage`/`JsonlSessionStorage` 命名
- ch10：补 compaction/branch-summary 的 `retry?: RetryPolicy`（`harness/types.ts:934`）
  与 `retry_scheduled`/`retry_attempt_start`/`retry_finished` 事件，与既有
  `"retry"` 阶段叙述呼应
- ch10：`AgentHarness` 认证描述按 models-only 更新（v0.80.0 移除
  `getApiKeyAndHeaders`，`models: Models` 必填）；`Agent` 路径的 `getApiKey`
  叙述保持不变
- ch10：`sdk.ts:293` 行号更新为 `:294`；「生产仍用 new Agent、AgentHarness 仅
  测试/docs 使用」的结论保留
- ch08/ch09/ch10 章末「版本演化说明」延伸声明已对照 v0.82.1

## Boundaries

### Allowed Changes
- pi-book/src/ch08-agent-loop.md
- pi-book/src/ch09-tool-execution.md
- pi-book/src/ch10-agent-class.md

### Forbidden
- 不要修改 pi-mono 源码
- 不要把 `Agent` 类的 `streamFn` 选项描述成已移除（`agent.ts:101` 仍在）
- 不要删除「生产未采用 AgentHarness」的结论

## Out of Scope

- coding-agent 对 SDK/会话/压缩层的产品化封装（见 update-v082-coding-agent）
- pi-ai 侧 `Models` 运行时本身（见 update-v082-ai）

## Completion Criteria

<!-- lint-ack: error-path — 本任务为书稿事实核查与改写,场景均为核查型断言,无运行时失败路径 -->

场景: ch08 循环签名与默认注入机制
  测试: review_ch08_stream_function
  当 检查 ch08 的 `runLoop` 签名与 stream 函数来源描述
  那么 底层循环描述为必填 `streamFunction`
  并且 说明 `setDefaultStreamFn`/`getDefaultStreamFn` 宿主级兜底
  并且 0.81.0→0.81.1 的演进写为取舍案例

场景: ch08 turn 间回调两变体
  测试: review_ch08_prepare_next_turn
  当 检查 ch08 的 turn 间回调小节
  那么 同时描述 `prepareNextTurn(signal)` 与 `prepareNextTurnWithContext(context, signal)`
  并且 注明优先调用带 context 的变体

场景: ch09 动态工具引入补充
  测试: grep_ch09_added_tool_names
  当 在 ch09 中搜索 `addedToolNames`
  那么 存在对该字段传播机制的说明

场景: ch10 AgentHarness 工具模型改写
  测试: review_ch10_harness_tools
  当 检查 ch10 的厚壳工具描述
  那么 描述 `AgentHarnessTool` 带应用自定义 context 的 execute 签名
  并且 说明 `ExecutionEnv` 被包装为 `ExecutionToolContext`

场景: ch10 会话两层与重试策略
  测试: review_ch10_session_retry
  当 检查 ch10 的会话与压缩描述
  那么 区分 `SessionStorage` 与 `SessionRepo` 两层
  并且 提及 `RetryPolicy` 与三个 retry_* 事件

场景: ch10 认证与行号订正
  测试: grep_ch10_models_only
  当 检查 ch10 中 AgentHarness 认证描述与 sdk.ts 行号引用
  那么 AgentHarness 侧为 models-only 且不再提 `getApiKeyAndHeaders`
  并且 `sdk.ts` 引用行号为 294

场景: 三章版本演化说明延伸
  测试: grep_runtime_chapters_evolution
  当 检查 ch08/ch09/ch10 章末版本演化说明
  那么 每章尾注声明已对照 v0.82.1
