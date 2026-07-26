spec: task
name: "更新v082-coding-agent 产品层同步 (v0.79.7→v0.82.1)"
inherits: project
tags: [chapter, update, coding-agent, v082]
satisfies: [REQ-SYNC]
depends: [update-v082-runtime]
estimate: 2.5d
---

## Intent

coding-agent 在本区间最大的变化是 `ModelRuntime` 取代 `ModelRegistry`/`AuthStorage`
成为模型与认证的规范门面（v0.80.8 Breaking），ch18 与 ch26b 的示例代码已失效。
其余为多点增量：bash 会话环境变量、constrained sampling、可恢复压缩、项目本地
配置层、资源加载边界（#7106）、RPC 新命令、扩展系统新能力。本任务按章修订并
吸收这些增量。

## Decisions

- ch18（coding-agent 侧）：`ModelRuntime`（`src/core/model-runtime.ts`）为规范的
  异步 model/auth 门面；`ModelRegistry` 降级为面向扩展的同步 compat 投影，其
  `refresh()` 返回 `Promise<void>`；`new ModelRegistry(authStorage, modelsJsonPath)`
  构造叙述废止；补文件型动态目录 `models-store.json`（`src/core/models-store.ts:28`）、
  pi.dev 目录 `If-None-Match`/`304` 重校验（`remote-catalog-provider.ts:70-112`）、
  `pi update --models` 强制刷新、`max`/`xhigh` thinking 层级一句带过
- ch26b：最小示例重写为 `ModelRuntime.create()` + `modelRuntime` 传入
  `createAgentSession`（`src/core/sdk.ts:38-45`）；注明 `AuthStorage` 不再从包根导出、
  凭证读取用 `readStoredCredential()`（`index.ts:26`）；新导出面（`InlineExtension`、
  生命周期事件类型、`JsonlSessionStorage`/`InMemorySessionStorage`）列举；新增
  「第二消费者」小节：packages/evals 的 pi-harness 用 `createHarness` 驱动真实
  `createAgentSession`（`packages/evals/src/pi-harness.ts`）
- ch22：新增 bash 会话环境变量小节 — `PI_SESSION_ID`/`PI_SESSION_FILE`/
  `PI_PROVIDER`/`PI_MODEL`/`PI_REASONING_LEVEL` 注入子进程
  （`src/core/tools/bash.ts:166-180`），并举 evals harness 依赖
  `PI_PROVIDER`/`PI_MODEL` 为用例
- ch19：新增「受约束采样」工具契约维度 — `constrainedSampling`
  （`src/core/extensions/types.ts:457`）prefer/require 严格 JSON Schema 或
  Lark/regex 语法，配 `supportsStrictTools`/`supportsGrammarTools` 能力位
- ch12：补可恢复压缩 — 重试策略与 `summarization_retry_*` 生命周期事件（v0.81.1）；
  压缩事件带 `reason`/`willRetry`（v0.79.10）与估算压缩后 token 数（v0.79.8）；
  usage 会计扩展到工具/压缩/分支摘要（v0.81.0）；压缩请求用全新 routing session id
  且禁用 prompt caching（v0.82.0）
- ch11：补 RPC 只读会话树访问 `get_entries`（带 `since`）/`get_tree`
  （`src/modes/rpc/rpc-types.ts:64-65`）；会话选择器按子树最近活动排序；
  可插拔存储导出与 JSONL header 自定义 metadata 一句带过
- ch13：新增项目本地资源配置层 — `pi config -l` 与 Tab 切换全局/项目作用域
  （v0.80.4）；新 settings 举例 `externalEditor`/`outputPad`/`shellPath` `~` 展开
- ch14：补「默认系统提示词移除当前日期以稳定 prompt cache」（v0.80.7）及其取舍
- ch15：扩展能力边界更新 — 完整 provider 扩展（v0.81.0）；新 hook
  `agent_settled`/`before_provider_headers`/`session_info_changed` 与
  `InlineExtension`；cache-friendly 动态工具加载（保留 prompt-cache 前缀，v0.80.7）
- ch17：补 #7106 — 上下文文件发现跳过与 context 文件同名的目录
  （`src/core/resource-loader.ts:73`）；项目本地资源覆盖与 ch13 交叉引用
- ch20/ch21/ch23：各补一句边界修正 — edit 模糊匹配保留未改动行块（v0.79.9）、
  read 错误不再语法高亮（v0.81.0）、`find` 尊重嵌套 git 仓库边界（v0.79.10）
- ch26：命令清单补 `get_entries`/`get_tree`/`get_available_thinking_levels`；
  事件补流式 `bash_execution_update` 与 `summarization_retry_*`；提 `./rpc-entry` 入口
- ch16 本区间无 skills 专属变更，仅章末尾注声明已对照 v0.82.1 无变化
- 受影响各章章末「版本演化说明」延伸声明已对照 v0.82.1

## Boundaries

### Allowed Changes
- pi-book/src/ch11-session-tree.md
- pi-book/src/ch12-compaction.md
- pi-book/src/ch13-config-layers.md
- pi-book/src/ch14-system-prompt.md
- pi-book/src/ch15-extensions.md
- pi-book/src/ch16-skills.md
- pi-book/src/ch17-resource-loader.md
- pi-book/src/ch19-tool-principles.md
- pi-book/src/ch20-edit-tool.md
- pi-book/src/ch21-read-tool.md
- pi-book/src/ch22-bash-tool.md
- pi-book/src/ch23-search-tools.md
- pi-book/src/ch26-rpc.md
- pi-book/src/ch26b-sdk.md
- pi-book/src/ch18-model-registry.md

### Forbidden
- 不要修改 pi-mono 源码
- 不要保留 `authStorage`/`modelRegistry` 作为 `createAgentSession` 现行选项的示例
- 不要把 Project Trust 描述成本区间有架构变化（实际无）

## Out of Scope

- pi-ai 侧 `Models`/`CredentialStore` 本体（见 update-v082-ai）
- TUI 渲染细节（见 update-v082-tui）
- 短命的 `/base` 入口（v0.79.8 引入、v0.80.0 移除，书中未引用则不提）

## Completion Criteria

场景: ch18 以 ModelRuntime 为核心改写
  测试: review_ch18_model_runtime
  假设 ch18 已更新
  当 检查 coding-agent 侧模型管理描述
  那么 `ModelRuntime` 为规范门面且 `ModelRegistry` 明确标注为 compat 投影
  并且 描述 models-store.json 持久化与 If-None-Match/304 增量刷新

场景: ch26b 示例可对应现行 API
  测试: review_ch26b_sdk_example
  当 检查 ch26b 最小示例代码
  那么 示例使用 `ModelRuntime.create()` 并以 `modelRuntime` 传入 `createAgentSession`
  并且 不再出现 `authStorage`/`modelRegistry` 选项
  并且 存在 evals harness 作为第二消费者的小节

场景: ch22 bash 会话环境变量
  测试: grep_ch22_pi_env_vars
  当 在 ch22 中搜索 "PI_SESSION_ID"
  那么 存在会话环境变量小节列出五个 PI_* 变量
  并且 举出 evals harness 用例

场景: ch19 受约束采样维度
  测试: grep_ch19_constrained_sampling
  当 在 ch19 中搜索 "constrainedSampling"
  那么 存在受约束采样的工具契约描述
  并且 提及 `supportsStrictTools` 与 `supportsGrammarTools` 能力位

场景: ch12 可恢复压缩与 usage 会计
  测试: review_ch12_retry_usage
  当 检查 ch12 的压缩失败处理与 usage 描述
  那么 描述重试策略与 summarization_retry_* 事件
  并且 说明 usage 会计覆盖工具/压缩/分支摘要
  并且 提及压缩请求禁用 prompt caching 的原因

场景: ch13 与 ch17 的项目本地层交叉一致
  测试: review_ch13_ch17_project_local
  当 检查 ch13 的配置分层与 ch17 的资源加载
  那么 ch13 描述 `pi config -l` 项目/全局作用域切换
  并且 ch17 描述同名目录跳过边界并与 ch13 交叉引用项目本地资源覆盖

场景: ch14 移除日期的取舍
  测试: grep_ch14_no_date
  当 检查 ch14 关于默认系统提示词内容的描述
  那么 说明当前日期已移除及 prompt cache 稳定性动机

场景: ch26 命令与事件清单补全
  测试: review_ch26_rpc_commands
  当 检查 ch26 的命令与事件清单
  那么 清单含 get_entries、get_tree、get_available_thinking_levels
  并且 事件含 bash_execution_update 与 summarization_retry_*

场景: 工具三章边界修正落笔
  测试: review_ch20_21_23_fixes
  当 检查 ch20/ch21/ch23 的边界行为描述
  那么 ch20 提及模糊匹配保留未改动行块
  并且 ch21 提及 read 错误不再被语法高亮
  并且 ch23 提及 find 尊重嵌套 git 仓库边界

场景: 受影响章节版本演化说明延伸
  测试: grep_coding_agent_chapters_evolution
  当 检查本合约 Allowed Changes 中各章的章末尾注
  那么 每章版本演化说明声明已对照 v0.82.1
  并且 ch16 明确记录本区间无 skills 专属变更
