spec: task
name: "更新v082-TUI 层同步 (v0.79.7→v0.82.1)"
inherits: project
tags: [chapter, update, tui, v082]
satisfies: [REQ-SYNC]
depends: [update-v082-coding-agent]
estimate: 0.5d
---

## Intent

packages/tui 在本区间无架构级重构，但书稿的光标模型存在实质缺口：ch24/ch25 只讲
`CURSOR_MARKER` + 硬件光标，而编辑器实际渲染的是反色**软件光标**（硬件光标是
IME 用的叠加层，v0.81.0 的清屏修复暴露了这一点）。另有粘贴/undo 一体化、
换行与 Markdown 渲染的若干增补。

## Decisions

- ch24/ch25 光标模型补缺：编辑器渲染反色软件光标，硬件光标仅作为 IME 定位
  叠加层；终端关闭时先清软件光标再恢复硬件光标（`src/tui.ts:699-702`，#6790）
- ch25 键位订正：换行默认键为 `Shift+Enter` 与 `Ctrl+J`（`src/keybindings.ts:118`），
  补 `Ctrl+J` 并保留终端能力差异说明
- ch25 粘贴与 undo 一体化：`UndoSnapshot` 含 `pastes: Map<number,string>`
  （`src/components/editor.ts:215-218`，#6844），undo 恢复文本的同时恢复
  paste registry；paste marker 删除/清屏后的记账清理（`editor.ts:1265,1296`）
- ch25 滚动指示器宽度安全：窄终端下 `sliceByColumn` 截断（`editor.ts:261-267`，#7015）
- ch24 增补：ANSI 感知换行识别 CRLF/CR 并跨行保留样式（#6764）；Markdown 渲染器
  的 source-preservation 选项族补「保留 backslash 转义」（`src/components/markdown.ts:101`）；
  流式代码围栏防闪烁（#5846）各一句带过
- ch24/ch25 章末「版本演化说明」延伸声明已对照 v0.82.1

## Boundaries

### Allowed Changes
- pi-book/src/ch24-tui.md
- pi-book/src/ch25-editor.md

### Forbidden
- 不要修改 pi-mono 源码
- 不要把硬件光标描述成唯一光标机制
- 不要展开终端兼容细节清单（维持「探测降级」概括叙述）

## Out of Scope

- coding-agent 交互模式对 TUI 的使用（见 update-v082-coding-agent）
- 日志目录变更（#6958，非 UI 语义）

## Completion Criteria

<!-- lint-ack: error-path — 本任务为书稿事实核查与改写,场景均为核查型断言,无运行时失败路径 -->

场景: 光标模型双层描述
  测试: review_ch24_ch25_cursor_model
  当 检查 ch24 与 ch25 的光标相关小节
  那么 区分反色软件光标与 IME 用硬件光标叠加层
  并且 提及退出时先清软件光标的处理

场景: ch25 换行键位更新
  测试: grep_ch25_ctrl_j
  当 在 ch25 中搜索 "Ctrl+J"
  那么 换行键位描述包含 `Ctrl+J`

场景: ch25 粘贴与 undo 一体化
  测试: review_ch25_paste_undo
  当 检查 ch25 的粘贴折叠与 Undo Stack 小节
  那么 说明 undo 快照包含 paste registry
  并且 提及 marker 删除与清屏后的记账清理

场景: ch25 滚动指示器截断
  测试: grep_ch25_scroll_indicator
  当 检查 ch25 滚动小节
  那么 提及窄终端下指示器按列宽截断的宽度安全处理

场景: ch24 渲染增补与尾注延伸
  测试: review_ch24_render_additions
  当 检查 ch24 的换行与 Markdown 渲染描述及两章尾注
  那么 提及 CRLF/CR 归一化与保留 backslash 转义选项
  并且 ch24/ch25 版本演化说明声明已对照 v0.82.1
