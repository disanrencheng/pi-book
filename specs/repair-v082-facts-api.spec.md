spec: task
name: "修复v082-事实与 API 收敛"
inherits: project
tags: [chapter, repair, facts, api, v082]
satisfies: [REQ-SYNC]
estimate: 1.5d
---

## Intent

在已完成的 v0.82.1 同步稿上清理仍残留于 README、outline、导航章、结论章和附录的
旧 API、错误统计与过时入口。当前事实统一以 pi-mono v0.82.1 commit `b4f29368`
为准，同时保留已完成的 23 项评审修复，不重写已经正确的技术章节。

## Decisions

- 当前事实基线固定为 pi-mono v0.82.1 commit `b4f29368`
- pi-ai 规范装配入口使用 `Models`、`createModels`、`createProvider` 与
  `@earendil-works/pi-ai/providers/*`；`@earendil-works/pi-ai/compat` 在 v0.82.1
  仍公开导出 `registerApiProvider`，只能标为临时迁移兼容入口，不能虚构
  `@deprecated` 标记；
  已删除的 `api-registry.ts` 只允许出现在明确标注的历史对照中
  （`packages/ai/package.json:18-20`；`packages/ai/src/compat.ts:1-10,126-148`；
  `packages/coding-agent/src/core/sdk.ts:1-3`）
- coding-agent 内建工具集合以
  `packages/coding-agent/src/core/tools/index.ts:83-84` 的 7 个名称为准；
  源文件数量不得推导为工具数量
- coding-agent 没有内建 `Agent` 工具；sub-agent 只能写成 extension/SDK 组合模式或
  明确标注的概念示例
- 当前二次开发入口统一为 provider/Models、SDK、RPC、server、storage/sqlite-node 与 extension；
  `evals` 只登记为 private 内部评测 harness
- README 的当前源码链接使用 pi 包清单声明的
  `https://github.com/earendil-works/pi`，并同时声明核心历史起点与当前对照版本
- 外部框架能力比较不在本合约核验，交给 `repair-v082-ecosystem-comparison`

## Boundaries

### Allowed Changes
- pi-book/README.md
- pi-book/outline.md
- pi-book/CLAUDE.md
- pi-book/specs/ch04-07-pi-ai.spec.md
- pi-book/src/SUMMARY.md
- pi-book/src/ch01-prologue.md
- pi-book/src/ch02-packages.md
- pi-book/src/ch03-reading-map.md
- pi-book/src/ch04-provider-registry.md
- pi-book/src/ch05-message-transform.md
- pi-book/src/ch06-event-stream.md
- pi-book/src/ch07-oauth.md
- pi-book/src/ch08-agent-loop.md
- pi-book/src/ch09-tool-execution.md
- pi-book/src/ch10-agent-class.md
- pi-book/src/ch13-config-layers.md
- pi-book/src/ch14-system-prompt.md
- pi-book/src/ch15-extensions.md
- pi-book/src/ch16-skills.md
- pi-book/src/ch17-resource-loader.md
- pi-book/src/ch18-model-registry.md
- pi-book/src/ch19-tool-principles.md
- pi-book/src/ch25-editor.md
- pi-book/src/ch31-contrarian-choices.md
- pi-book/src/ch32-boundaries.md
- pi-book/src/appendix.md
- pi-book/.agent-spec/reviews/repair-v082/facts-api.decisions.json
- pi-book/.agent-spec/reviews/repair-v082/facts-api.lifecycle.json
- pi-book/.agent-spec/reviews/repair-v082/facts-api.report.json
- pi-book/docs/superpowers/reviews/2026-07-26-pi-book-v082-content-repair-review.md

### Forbidden
- 不要修改 pi-mono 源码、测试、依赖或生成文件
- 不要删除章节或历史案例
- 不要把 `b4f29368` 之后的主线行为写成 v0.82.1 事实
- 不要把 compat `registerApiProvider` 描述成规范主线或虚构 `@deprecated` 标记
- 不要把已删除的 `api-registry.ts` 或内建 `Agent` 工具保留为当前能力
- 不要回退已完成的版本尾注、SUMMARY 标点或现行 `Models`/`ModelRuntime` 修复

## Out of Scope

- agent loop、Skill、`partial`、compaction 等跨章语义修复
- 安全边界、Mermaid、代码块长度与历史章节尾注
- LangChain、Vercel AI SDK 等外部生态能力比较

## Completion Criteria

<!-- lint-ack: error-path — 本任务为书稿事实核查与改写，场景均为静态核查型断言 -->

场景: 当前 pi-ai API 不再与历史注册表混用
  测试: grep_no_removed_api_as_current
  当 全书扫描 README、outline、CLAUDE、specs 与 src 中的 provider 入口
  那么 规范主线只使用 `Models`、`createModels`、`createProvider` 或 `providers/*`
  并且 `registerApiProvider` 只位于明确标注的临时 compat/迁移语境
  并且 `api-registry.ts` 只位于明确标注的历史对照

场景: 工具数量来自内建工具集合而不是源文件数
  测试: review_ch02_builtin_tool_count
  当 检查 ch02 的规模统计与工具描述
  那么 不再出现“130+ 个工具实现”
  并且 内建工具口径与 `ToolName` 的 7 个名称一致

场景: sub-agent 不冒充内建 Agent 工具
  测试: grep_no_builtin_agent_tool_claim
  当 检查 ch31、ch32 与附录中的 sub-agent 说明
  那么 不再声称 pi-coding-agent 内建 `Agent` 工具
  并且 组合示例标明 extension/SDK 实现或概念伪代码

场景: README 与 outline 反映当前仓库和目录
  测试: review_readme_outline_current
  当 检查 README、outline 与 SUMMARY
  那么 源码链接指向 `earendil-works/pi`
  并且 当前对照版本为 v0.82.1
  并且 目录包含 SDK 章节并把 ch27-29 标为历史案例

场景: 二次开发入口使用当前能力
  测试: review_current_extension_entrypoints
  当 检查 ch32 与附录的二次开发入口表
  那么 provider、SDK、RPC、server、storage/sqlite-node 与 extension 入口对应当前 API
  并且 `evals` 明确标注为 private

场景: 当前类型与扩展 hook 示例对应 v082 源码
  测试: review_current_types_and_hooks
  当 检查 ch04-10、ch13-19 与 ch25 中的 API 示例
  那么 `Model`、`AssistantMessageEventStream`、tool finalize 与 Extension prompt hook
  均指向 `b4f29368` 中存在的符号和签名
  并且 概念伪代码不会被标成可直接编译的当前 API

场景: 修复未污染历史和上级源码
  测试: review_facts_repair_boundaries
  当 检查本合约产生的变更
  那么 ch27-29 的 v0.66.1 历史正文没有被改写为当前事实
  并且 上级 pi-mono 源码无变更
