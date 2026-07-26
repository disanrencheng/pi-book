spec: task
name: "更新-pi-ai 设计变更同步 (ch04-07,18)"
inherits: project
tags: [chapter, update, pi-ai]
estimate: 1.5d
depends: [update-structural]
---

## Intent

同步 packages/ai 从 v0.66.1 到 v0.79.7 的设计级变更到第 4-7 章与第 18 章。
重点是 OAuth 章（5→3 provider + 回调破坏性变更，改动最大）、Model Registry 章
（thinkingLevelMap 取代 reasoningEffortMap、验证栈 Ajv→typebox）、Provider 章
（API 协议数变化、compat 三分支），并新增图像生成子系统这一全新主题。

## Decisions

- ch04: KnownApi 从 10 种降为 9 种（移除 google-gemini-cli，v0.71.0）；KnownProvider 约 38 个；compat 条件类型新增第三分支 anthropic-messages（AnthropicMessagesCompat）
- ch05: 订正跨模型示例 api 值 "messages" → "anthropic-messages"；补 trailing tool-result 合成场景（v0.69.0）
- ch06: AssistantMessage 新增 responseModel、diagnostics 字段；Usage 新增 cacheWrite1h；新增 StreamOptions 扩展点小节（transport、onPayload、onResponse、env）
- ch07: 内建 OAuth provider 从 5 个降为 3 个（删 Google Gemini CLI、Antigravity，v0.71.0）；OAuthLoginCallbacks 新增必填 onDeviceCode/onSelect（v0.75.5）；新增 device-code 登录小节；订正 callback host 可经 PI_OAUTH_CALLBACK_HOST 配置（不再硬编码 127.0.0.1）
- ch18: thinkingLevelMap 取代 compat.reasoningEffortMap（v0.72.0），supportsXhigh() 已废弃；验证栈从 Ajv 迁移到 typebox 1.x（v0.69.0）
- 新增主题: 图像生成子系统（images-api-registry、ImagesModel、AssistantImages，v0.74.1），作为 ch06"为什么用事件流"的反例（图像返回 Promise 而非事件流）。放入 ch18 或附近小节，不单列新章

## Boundaries

### Allowed Changes
- pi-book/src/ch04-provider-registry.md
- pi-book/src/ch05-message-transform.md
- pi-book/src/ch06-event-stream.md
- pi-book/src/ch07-oauth.md
- pi-book/src/ch18-model-registry.md

### Forbidden
- 不要修改 pi-mono 源码
- 不要把 ch07 的 OAuth 流程仍描述为 5 种
- 不要在 ch18 把 ai 层参数校验描述为 Ajv

## Out of Scope

- scope 字面量替换（由 update-structural 统一处理）

## Completion Criteria

场景: ch04 协议数已更新为 9
  测试: review_ch04_nine_apis
  假设 ch04-provider-registry.md 已更新
  当 检查 KnownApi 描述
  那么 正文称内建 API 协议为 9 种
  并且 列表不再包含 "google-gemini-cli"

场景: ch04 补充 compat 第三分支
  测试: review_ch04_compat_branch
  当 检查 compat 条件类型代码块
  那么 包含 anthropic-messages 分支（AnthropicMessagesCompat）

场景: ch07 OAuth 降为 3 种
  测试: review_ch07_three_oauth
  假设 ch07-oauth.md 已更新
  当 检查 OAuth provider 框架与 mermaid 图
  那么 正文称内建 OAuth provider 为 3 种
  并且 不再出现 "Antigravity"

场景: ch07 旧 provider 不再作为当前 provider 列出
  测试: review_ch07_removed_not_active
  当 检查 ch07 的活跃 OAuth provider 框架、列表与 mermaid 图
  那么 Gemini CLI 与 Antigravity 不作为当前/活跃 provider 出现
  并且 仅在版本演化说明中以"已于 v0.71.0 移除"的历史语境被点名

场景: ch07 回调接口含新必填回调
  测试: review_ch07_callbacks
  当 检查 OAuthLoginCallbacks 接口与正文
  那么 列出 onDeviceCode 与 onSelect 两个回调
  并且 包含 device-code 登录流程的说明

场景: ch07 订正硬编码回调主机表述
  测试: grep_ch07_callback_host
  当 检查 ch07 取舍段关于回调地址的描述
  那么 提到 PI_OAUTH_CALLBACK_HOST 可配置主机
  并且 不再断言"硬编码了 127.0.0.1……被占用就失败"

场景: ch18 用 thinkingLevelMap 取代 reasoningEffortMap
  测试: review_ch18_thinking_level_map
  假设 ch18-model-registry.md 已更新
  当 检查 Model 接口与 compat schema
  那么 出现 thinkingLevelMap
  并且 不再以 reasoningEffortMap 描述当前模型能力

场景: ch18 验证栈不再描述为 Ajv
  测试: grep_ch18_no_ajv
  当 在 ch18 中检查参数校验技术的描述
  那么 ai 层校验描述为 typebox（非 Ajv）

场景: 新增图像生成反例小节
  测试: review_image_generation_section
  当 检查 ch18 或相邻小节
  那么 介绍图像生成子系统（ImagesModel/AssistantImages）
  并且 说明其返回 Promise 而非事件流，作为事件流设计的反例
