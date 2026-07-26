spec: task
name: "更新v082-pi-ai 架构范式迁移 (v0.79.7→v0.82.1)"
inherits: project
tags: [chapter, update, ai, v082]
satisfies: [REQ-SYNC]
depends: [update-v082-structural]
estimate: 2d
---

## Intent

pi-ai 在 v0.80.0 经历全局性 breaking 重构：全局注册表（`api-registry.ts`、
`stream.ts`、文本侧 `register-builtins.ts`）被删除，代之以显式的 `Models`/`Provider`
运行时集合；v0.80.8 又把 OAuth 从独立注册表重写为 per-provider 的认证子系统
（`src/auth/`）。ch04 与 ch07 因此**全章级失效**，ch05/ch06/ch18 存在路径与
类型漂移。本任务把这次「隐式全局单例 → 显式依赖注入集合」的范式迁移写成
各章的新主线，旧机制保留为历史对照。

## Decisions

- ch04 重写主线：`Models` 运行时（`models.ts:127`）+ `createModels`（`models.ts:529`）+
  `createProvider`（`models.ts:556`）+ provider factory / `providers/all.ts` 聚合；
  旧的 98 行 `api-registry.ts`、`createLazyStream`、`sourceId` 批量注销降级为
  「v0.66-v0.79 的历史形态」对照小节，并指出旧全局 API 迁到 `/compat` 临时入口
- 类型订正：`Provider` 联合类型已改名 `ProviderId`（`types.ts:73`）；`Provider`
  现指运行时 provider 接口（`models.ts:75`）；ch06/ch18 中 `provider: Provider`
  字段一并订正为 `ProviderId`
- 计数订正：`KnownApi` 9→10（新增 `pi-messages`，v0.80.8）；`KnownProvider`
  约 35→38（qwen-token-plan(-cn)、kimi-coding、xiaomi-token-plan-* 等）
- ch05：`transformMessages` 路径改为 `api/transform-messages.ts:64`，行号随之更新；
  新增 null/undefined content 归一化（`:71-73`，v0.80.4）作为防御性设计补充
- ch06：`Usage` 补 `reasoning` 字段；`ToolResultMessage` 补 `addedToolNames`
  （`types.ts:418`）与可选 `usage`；`StreamOptions`/`Tool` 扩展点补
  `constrainedSampling`（`types.ts:474`）与 `transformHeaders`（`models.ts:58`）
- ch07 重写主线：认证升级为一等子系统 `src/auth/` — `ProviderAuth`（`auth/types.ts:217`）、
  `CredentialStore`（`:60`）、`AuthContext`、`OAuthAuth`（login/refresh/toAuth，`:189`）；
  交互协议从 6 回调（`AuthLoginCallbacks`）收敛为 `AuthInteraction` 的
  `prompt()`/`notify()` 两方法（`auth/types.ts:150`），作为「接口收敛」取舍案例
- ch07 取舍反转订正：错误信息策略从「统一简化、丢弃原始信息」反转为
  `ModelsError` 追加底层 cause（`auth/resolve.ts:22-45`，v0.82.1），原取舍分析改写
- ch07 OAuth provider 列表：内建从 3 家扩为多家（+xAI、Kimi Coding、OpenRouter PKCE、
  Radius，见 `auth/oauth/` 目录），运行时入口为 provider-scoped 的
  `Models.checkAuth()/getAuth()/login()/logout()`
- ch18（ai 侧）：单体 `models.generated.ts` 拆为 per-provider `*.models.ts` +
  `providers/all.ts`（`getBuiltinModel(s)`，`:59/77`）；新增 `ModelsStore`/
  `InMemoryModelsStore` 与 `ModelsStoreEntry.etag` 条件刷新；
  `getBuiltinModelDataUrl` → `getBuiltinModelDataGeneratedAt()`（v0.82.0 Breaking）
- 各章「版本演化说明」尾注延伸：记录本节内容已对照 v0.82.1，注明 v0.80.0 与
  v0.80.8 两次 breaking 的分界

## Boundaries

### Allowed Changes
- pi-book/src/ch04-provider-registry.md
- pi-book/src/ch05-message-transform.md
- pi-book/src/ch06-event-stream.md
- pi-book/src/ch07-oauth.md
- pi-book/src/ch18-model-registry.md
- pi-book/src/SUMMARY.md

### Forbidden
- 不要修改 pi-mono 源码
- 不要删除旧注册表/旧 OAuth 注册表的叙述（改为明确标注的历史对照）
- 不要引用已删除文件的 file:line 却不标注「历史形态」

## Out of Scope

- ch18 中 coding-agent 侧 `ModelRuntime`/`ModelRegistry` 的改写（见 update-v082-coding-agent）
- ch08-10 对 `getApiKey` 回调叙述的连锁更新（见 update-v082-runtime）

## Completion Criteria

场景: ch04 以 Models 运行时为主线
  测试: review_ch04_models_runtime
  假设 ch04 已重写
  当 检查章节主线与代码引用
  那么 主线围绕 `Models`/`createModels`/`createProvider` 与 provider factory 展开
  并且 旧注册表机制置于明确标注的历史对照小节
  并且 正文注明旧全局 API 迁至 `/compat`

场景: ch04 不再把已删文件当作现状引用
  测试: grep_ch04_no_stale_refs
  当 在 ch04 中检查 `api-registry.ts`、`stream.ts`、`register-builtins.ts` 的引用
  那么 每处引用都位于历史对照语境或已改为现行文件路径
  并且 `Provider` 联合类型的现状描述使用 `ProviderId`

场景: ch05 路径与防御性归一化订正
  测试: review_ch05_transform_path
  当 检查 ch05 的源码引用
  那么 `transformMessages` 引用指向 `api/transform-messages.ts`
  并且 新增 null content 归一化的说明

场景: ch06 数据模型增补
  测试: review_ch06_usage_fields
  当 检查 ch06 的 `Usage` 与 `ToolResultMessage` 描述
  那么 `Usage` 含 `reasoning` 字段
  并且 `ToolResultMessage` 含 `addedToolNames` 与可选 `usage`
  并且 扩展点小节提及 `constrainedSampling` 与 `transformHeaders`

场景: ch07 以认证子系统为主线
  测试: review_ch07_auth_subsystem
  假设 ch07 已重写
  当 检查章节主线
  那么 主线围绕 `src/auth/` 的 `ProviderAuth`/`CredentialStore`/`OAuthAuth` 展开
  并且 六回调到 `prompt()`/`notify()` 的收敛写入取舍分析
  并且 OAuth 注册表叙述置于历史对照语境

场景: ch07 错误信息取舍反转已改写
  测试: grep_ch07_error_cause
  当 检查 ch07 关于刷新错误信息的段落
  那么 描述为 `ModelsError` 保留底层 cause
  并且 不再声称错误被统一简化丢弃原始信息

场景: ch18 目录拆分与新鲜度接口订正
  测试: review_ch18_ai_side
  当 检查 ch18 中 ai 包侧的目录结构描述
  那么 描述 per-provider `*.models.ts` 拆分与 `getBuiltinModel(s)` 读法
  并且 提及 `ModelsStore` 持久化与 etag 条件刷新
  并且 不再把 `getBuiltinModelDataUrl` 作为现行接口

场景: 五章版本演化说明延伸到 v0.82.1
  测试: grep_ai_chapters_evolution
  当 检查 ch04/ch05/ch06/ch07/ch18 章末的版本演化说明
  那么 每章尾注声明已对照 v0.82.1
  并且 ch04 与 ch07 注明 v0.80.0/v0.80.8 breaking 分界
