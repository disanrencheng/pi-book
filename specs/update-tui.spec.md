spec: task
name: "更新-pi-tui 变更同步 (ch24-25)"
inherits: project
tags: [chapter, update, tui]
estimate: 1d
depends: [update-coding-agent]
---

## Intent

同步 packages/tui 从 v0.66.1 到 v0.79.7 的设计级变更到第 24-25 章。重点修正两处
与现实相反/缺字段的事实性错误（Container.render 代码、AutocompleteProvider 接口），
并补充 Kitty 图片协议、终端 light/dark 检测、grapheme 断行重写等新能力面。

## Decisions

- ch24: 替换 Container.render 的 spread-push 代码为循环 push（栈溢出修复 #2651，v0.67.0）；新增终端 light/dark 检测小节（queryTerminalColorScheme，OSC 11，v0.79.x）；新增 Kitty 图片协议与降级小节（terminal-image.ts）；补 Node 最低版本升至 22.19.0（v0.75.0）
- ch25: AutocompleteProvider 接口补 triggerCharacters 与 shouldTriggerFileCompletion；SlashCommand 补 argumentHint；word wrap 段改为按 grapheme 边界断行（v0.79.x），补 Unicode 词边界导航
- 行号会漂移，引用以 v0.66 为准或重新核对

## Boundaries

### Allowed Changes
- pi-book/src/ch24-tui.md
- pi-book/src/ch25-editor.md

### Forbidden
- 不要修改 pi-mono 源码
- 不要保留与现实相反的 Container.render spread 代码

## Completion Criteria

场景: ch24 Container.render 代码修正
  测试: review_ch24_container_render
  假设 ch24-tui.md 已更新
  当 检查 Container.render 代码示例
  那么 使用循环 push 而非 spread-into-push
  并且 说明动机是避免长会话栈溢出（Maximum call stack size exceeded）

场景: ch24 新增终端配色检测
  测试: review_ch24_color_scheme
  当 检查终端能力检测段
  那么 介绍终端 light/dark 检测（如 queryTerminalColorScheme / OSC 11）

场景: ch24 新增 Kitty 图片协议
  测试: review_ch24_kitty_image
  当 检查 ch24
  那么 包含 Kitty 图片（graphics）协议与降级的小节
  并且 与既有 Kitty keyboard 协议区分

场景: ch25 AutocompleteProvider 接口补字段
  测试: review_ch25_autocomplete
  假设 ch25-editor.md 已更新
  当 检查 AutocompleteProvider 接口
  那么 包含 triggerCharacters 与 shouldTriggerFileCompletion 字段

场景: ch25 断行改为 grapheme 边界
  测试: review_ch25_grapheme_wrap
  当 检查 word wrap 段
  那么 说明按 grapheme 边界断行
  并且 修正旧的 CJK 大尾隙问题描述
