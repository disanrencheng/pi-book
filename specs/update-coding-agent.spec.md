spec: task
name: "更新-coding-agent 变更同步 (ch11-17,19-23,26 + 新章)"
inherits: project
tags: [chapter, update, coding-agent]
estimate: 3d
depends: [update-runtime]
---

## Intent

同步 packages/coding-agent 从 v0.66.1 到 v0.79.7（约 1096 次提交）的设计级变更，
覆盖第 11-17、19-23、26 章。重点是三个横切/新增主题：Project Trust 信任闸门
（v0.79.0 最大安全设计）、SDK 嵌入路径（pi 定位为可嵌入库）、system prompt 边界
从 Markdown 标题改为 XML 标签；以及各工具与 RPC 命令清单的过时修正。

## Decisions

- ch11: 补 /clone 与 fork position；--session-id、UUIDv7 会话 ID（v0.67.1）；订正 /resume 从"整文件读成字符串"改为逐行流式读 JSONL（v0.78.1）
- ch12: 摘要不再强制 high thinking（复用当前级别）；摘要 system prompt 中性化为 "AI assistant"（v0.79.0）；补区间起点回退 + tokensBefore 重算
- ch13: 新增"第四道闸门：Project Trust"主节（defaultProjectTrust、--approve、/trust、trust.json、project_trust 事件）；RetrySettings 分裂为 app 层退避与 retry.provider.* SDK 层（v0.70.1）；CONFIG_DIR_NAME 作为 rebrand 扩展点
- ch14: system prompt 边界从 Markdown 标题改为 XML 标签 <project_context>/<project_instructions path>（v0.75.0）；删除已移除的"prefer grep/find over bash"指引（v0.77.0）
- ch15: 导入从 @sinclair/typebox 改为 typebox（v0.69.0）；工具选择改名字 allowlist（tools: string[]，工厂 createReadTool(cwd)）；会话替换后旧 ctx 失效需用 withSession 模式；新增 hook（project_trust、after_provider_response、message_end 替换）
- ch16: 补显式 /skill:name 触发的 XML 包装；name 与目录不一致从 error 放宽为约定（v0.74.1）
- ch17: cwd 改为必填（移除 process.cwd() 兜底，v0.68.0）；项目本地资源受 Project Trust 门控
- ch19: 工具激活模型重写为"名字白名单 + cwd 工厂"；统一表述为"6 个结构化工具 + bash 后备 = 7 个内置工具"
- ch20: edits 字段为 JSON 字符串时先 parse；结果附 unified patch（generateUnifiedPatch 已导出，v0.79.7）
- ch21: 订正 AGENTS.md/CLAUDE.md/SKILL.md 等折叠为单行（非"只显示前 10 行"，v0.73.0）
- ch22: 增量流式输出入上下文；新增 excludeFromContext（!! 前缀，v0.76.0）小节
- ch23: grep 改走 ripgrep --json 流式；find 用 fd（fdfind 回退链）
- ch26: 补全 RPC 命令清单（clone/switch_session/compact/set_auto_*/abort_* 等）；prompt 单一权威响应契约；RpcClient typed client 已导出；LF framing 陷阱
- 新增主题: SDK 嵌入（createAgentSession/AgentSession），建议新增章节与 ch26 RPC 并列（嵌入 vs 子进程两条集成路径）
- 通则: 去 process-global（cwd 处处显式传入）；边界统一用 XML 标签

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
- pi-book/src/SUMMARY.md

### Forbidden
- 不要修改 pi-mono 源码
- 不要把项目本地资源/设置仍描述为无条件加载（须经 Project Trust）
- 不要在 ch14 把 project context 边界描述为 Markdown 标题

## Out of Scope

- Containerization/Gondolin 沙箱、包管理自更新、遥测等运维主题（留待 update-philosophy，本任务不覆盖）

## Completion Criteria

场景: ch13 新增 Project Trust 闸门
  测试: review_ch13_project_trust
  假设 ch13-config-layers.md 已更新
  当 检查配置加载流程
  那么 包含 Project Trust 信任闸门小节
  并且 提到 defaultProjectTrust、--approve、trust.json
  并且 不再把项目 .pi/settings.json 描述为无条件覆盖

场景: ch14 边界改为 XML 标签
  测试: review_ch14_xml_boundary
  假设 ch14-system-prompt.md 已更新
  当 检查 project context 的边界表示
  那么 使用 <project_context> 与 <project_instructions path=...> XML 标签
  并且 不再使用 "# Project Context" Markdown 标题包裹

场景: ch14 删除已移除的 bash 指引
  测试: grep_ch14_no_prefer_grep
  当 在 ch14 检查 system prompt 工具指引
  那么 不再展示"优先用 grep/find/ls 而非 bash"的已删除分支

场景: ch15 工具选择改为名字白名单
  测试: review_ch15_tool_allowlist
  假设 ch15-extensions.md 已更新
  当 检查工具选择 API
  那么 说明 tools 为 string[] 名字白名单
  并且 介绍 withSession 模式处理会话替换后 ctx 失效

场景: ch15 typebox 导入修正
  测试: grep_ch15_typebox
  当 在 ch15 检查 TypeBox 导入
  那么 使用 "typebox"
  并且 不再使用 "@sinclair/typebox"

场景: ch17 cwd 必填且资源受信任门控
  测试: review_ch17_cwd_trust
  假设 ch17-resource-loader.md 已更新
  当 检查资源加载契约
  那么 说明 cwd 为必填（无 process.cwd() 兜底）
  并且 说明项目本地资源受 Project Trust 门控

场景: ch21 折叠语义订正
  测试: review_ch21_collapse
  当 检查 ch21 对 AGENTS.md/CLAUDE.md/SKILL.md 的渲染描述
  那么 说明这些文件折叠为单行（可 Ctrl+O 展开）
  并且 不再断言这些文件"只显示前 10 行"

场景: ch22 新增 excludeFromContext
  测试: review_ch22_exclude_context
  假设 ch22-bash-tool.md 已更新
  当 检查 bash 工具
  那么 介绍 excludeFromContext（!! 前缀）执行但输出不入上下文

场景: ch26 RPC 命令清单补全
  测试: review_ch26_rpc_commands
  假设 ch26-rpc.md 已更新
  当 检查 RPC 命令清单
  那么 包含 clone、switch_session、compact 等新增命令
  并且 提到 RpcClient typed client 已从包根导出

场景: 新增 SDK 嵌入章节
  测试: review_sdk_chapter
  当 检查 ch26b-sdk.md 与 SUMMARY.md
  那么 ch26b-sdk.md 介绍 createAgentSession/AgentSession 嵌入路径
  并且 SUMMARY.md 已登记该新章节

场景: 工具数表述统一
  测试: grep_ch19_seven_tools
  当 在 ch19-tool-principles.md 检查内置工具表述
  那么 表述为"6 个结构化工具 + bash 后备"
