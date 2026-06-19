spec: task
name: "更新-结构性漂移同步 (v0.66→v0.79)"
inherits: project
tags: [chapter, update, structural]
estimate: 1d
---

## Intent

pi-book 写作基线为 pi-mono v0.66.1（commit c779c14e），当前代码已到 v0.79.7。
本任务修正全书的**结构性与事实性漂移**：monorepo 从 7 个包收缩到 4 个包、
npm scope 从 `@mariozechner/` 迁移到 `@earendil-works/`、mom/pods/web-ui 三个包
已移出主仓库。这些是不改即为错误的硬伤，优先于其他更新。

## Decisions

- 当前 4 个包: `@earendil-works/pi-ai`、`@earendil-works/pi-agent-core`、`@earendil-works/pi-coding-agent`、`@earendil-works/pi-tui`
- scope 迁移: 作为**当前 canonical 包名**的引用，`@mariozechner/pi-{ai,tui,agent-core,coding-agent}` → `@earendil-works/pi-{...}`（正文叙述与"当前状态"manifest 示例）
- **例外（不替换）**: (1) `@mariozechner/clipboard` 是 coding-agent 当前真实依赖（packages/coding-agent/package.json:65），保留；(2) ch15 的 jiti virtualModules 代码中 `@mariozechner/pi-*` 作为**向后兼容别名**仍被保留（loader.ts:56-60 同时注册新旧两套 key），ch15 应展示新 canonical key 并加一句说明旧名作为别名保留；(3) ch27/28/29 作为 v0.66.1 历史快照，包名引用可保留旧名，由章首历史快照注记统一覆盖
- ch15 附带订正: extension 运行时编译已从 `@mariozechner/jiti` fork 改为上游 `jiti@2.7.0`（package.json:50）
- 仓库链接替换: 若出现 `badlogic/pi-mono` 则改为 `earendil-works/pi`（经核实 src/ 中当前无残留）
- 已移出包的处理: 保留 ch27/ch28/ch29 章节，章首加「已迁出」注记并声明分析基于 v0.66.1 历史快照（commit c779c14e）
- mom 继任方向: GitHub `earendil-works/pi-chat`（commit 0ed0d434）；pods 无继任；web-ui 无继任（commit b141e1fa）
- ch02 标题「七个包不是七个项目」改为反映 4 包现状，并把 7→4 收缩作为"使用者分化则移出"设计论点的正向案例
- CLAUDE.md 源码参考清单删除 mom/pods/web-ui 三条不存在的路径

## Boundaries

### Allowed Changes
- pi-book/src/ch01-prologue.md
- pi-book/src/ch02-packages.md
- pi-book/src/ch03-reading-map.md
- pi-book/src/ch10-agent-class.md
- pi-book/src/ch15-extensions.md
- pi-book/src/ch27-web-ui.md
- pi-book/src/ch28-mom-slack.md
- pi-book/src/ch29-pods-gpu.md
- pi-book/src/SUMMARY.md
- pi-book/CLAUDE.md

### Forbidden
- 不要修改 pi-mono 源码
- 不要删除 ch27/ch28/ch29 章节文件（仅加注记）
- 不要把已移出包描述成仍在主仓库

## Out of Scope

- ai/agent/coding-agent/tui 各包的设计级内容更新（见 update-ai / update-runtime / update-coding-agent / update-tui）

## Completion Criteria

场景: 活跃章节迁移 canonical 包名
  测试: review_active_scope_migrated
  假设 已完成 scope 迁移
  当 检查 ch01/ch02/ch10 等活跃章节中作为当前包名的引用
  那么 使用 "@earendil-works/pi-" 作为 canonical 包名
  并且 不再以 "@mariozechner/pi-" 指代当前包

场景: 保留真实依赖与向后兼容别名
  测试: review_scope_exceptions_preserved
  假设 scope 迁移完成
  当 检查例外项
  那么 `@mariozechner/clipboard` 作为真实依赖被保留
  并且 ch15 展示 `@earendil-works/pi-*` 为 virtualModules key 并说明旧名作为向后兼容别名保留

场景: ch15 jiti 订正为上游版本
  测试: grep_ch15_jiti_upstream
  当 检查 ch15 关于 extension 运行时编译的描述
  那么 说明使用上游 jiti（2.7.0）
  并且 不再描述为 `@mariozechner/jiti` fork

场景: ch02 反映 4 包结构
  测试: review_ch02_four_packages
  假设 ch02-packages.md 已更新
  当 检查章节正文与规模表
  那么 正文不再声称当前有"七个包"或"7 个包"
  并且 规模表不包含 mom、pods、web-ui 三行
  并且 build 顺序描述为 "tui → ai → agent → coding-agent"

场景: ch02 mermaid 不再含已移除节点
  测试: review_ch02_mermaid
  当 检查 ch02 的 mermaid 分层图
  那么 图中不包含 mom、pods、web-ui 节点

场景: 已移出包章节带迁出注记
  测试: review_migrated_chapters_notice
  假设 ch27/ch28/ch29 保留
  当 检查每章章首
  那么 每章都声明对应包已从 pi-mono 主仓移出
  并且 注明分析基于 v0.66.1 历史快照（commit c779c14e）
  并且 ch28 指向 earendil-works/pi-chat

场景: ch03 阅读地图无失效路径
  测试: grep_ch03_no_dead_paths
  当 在 ch03-reading-map.md 中搜索 "packages/web-ui" 与 "packages/mom"
  那么 匹配数为 "0"
  并且 文件不再引用 "components/list.ts"（已更名为 select-list.ts）

场景: CLAUDE.md 源码参考无不存在路径
  测试: grep_claudemd_no_dead_pkgs
  当 在 pi-book/CLAUDE.md 的源码参考清单中搜索 "packages/mom"、"packages/pods"、"packages/web-ui"
  那么 匹配数为 "0"
