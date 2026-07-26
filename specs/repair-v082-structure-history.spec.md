spec: task
name: "修复v082-结构门禁与历史闭环"
inherits: project
tags: [chapter, repair, structure, history, v082]
satisfies: [REQ-SYNC]
depends: [repair-v082-semantics-security]
estimate: 1.5d
---

## Intent

在事实和语义收敛后，完成全书结构门禁与历史章节闭环。项目级规范已经在 repair 合约
创建前新增“每章至少一张 Mermaid”规则；本任务验证该治理前置项、补齐当前缺图、
拆分超长代码块，并让 ch27-29 的章首与章尾都明确使用 v0.66.1 历史时态，最后更新
live wiki 和验证产物。

## Decisions

- “每章至少一张 Mermaid”是 2026-07-26 新增且已经写入
  `specs/project.spec.md` 的 project-level 规则；本任务只验证并执行，不把它冒充旧规范
- 当前缺图章节为 ch21、ch22、ch23、ch25、ch28、ch29、ch32；
  每张图必须承载核心流程、状态或关系
- 当前超过 40 行的 fenced code block 为 ch08（93 行）、ch09（52 行）、
  ch10（51 行）、ch13（64/46 行两块）、ch17（41 行）、ch31（58 行）；
  这些当前违规块恰好都是 TypeScript，但门禁扫描所有语言
- ch27-29 的全部源码事实固定使用 pi-mono v0.66.1 commit `c779c14e`；
  章首与章尾都要说明移出状态、继任情况和保留原因
- ch01、ch03、ch24、ch26b 不强制新增“取舍分析”标题，
  但关键决策必须明确写出得到与放弃
- review 型场景由三视角评审给出 verdict；`verify` 的 `skip` 不是通过
- 创建 `scripts/check-chapter-structure.mjs`，机械检查全部章节的定位、Mermaid、
  版本演化说明和所有 fenced code block 的 40 行上限
- 收尾更新 `.agent-spec/wiki/learnings/sync-v0797-v0821.md`

## Boundaries

### Allowed Changes
- pi-book/src/ch01-prologue.md
- pi-book/src/ch03-reading-map.md
- pi-book/src/ch08-agent-loop.md
- pi-book/src/ch09-tool-execution.md
- pi-book/src/ch10-agent-class.md
- pi-book/src/ch13-config-layers.md
- pi-book/src/ch17-resource-loader.md
- pi-book/src/ch21-read-tool.md
- pi-book/src/ch22-bash-tool.md
- pi-book/src/ch23-search-tools.md
- pi-book/src/ch24-tui.md
- pi-book/src/ch25-editor.md
- pi-book/src/ch26b-sdk.md
- pi-book/src/ch27-web-ui.md
- pi-book/src/ch28-mom-slack.md
- pi-book/src/ch29-pods-gpu.md
- pi-book/src/ch30-minimal-core.md
- pi-book/src/ch31-contrarian-choices.md
- pi-book/src/ch32-boundaries.md
- pi-book/src/appendix.md
- pi-book/scripts/check-chapter-structure.mjs
- pi-book/.agent-spec/wiki/learnings/sync-v0797-v0821.md
- pi-book/.agent-spec/reviews/repair-v082/project.decisions.json
- pi-book/.agent-spec/reviews/repair-v082/project.lifecycle.json
- pi-book/.agent-spec/reviews/repair-v082/project.report.json
- pi-book/.agent-spec/reviews/repair-v082/structure-history.decisions.json
- pi-book/.agent-spec/reviews/repair-v082/structure-history.lifecycle.json
- pi-book/.agent-spec/reviews/repair-v082/structure-history.report.json
- pi-book/docs/superpowers/reviews/2026-07-26-pi-book-v082-content-repair-review.md

### Forbidden
- 不要修改 pi-mono 源码
- 不要删除 ch27-29 或把历史产品写成当前仓库组件
- 不要通过删除必要语义来缩短代码块
- 不要添加装饰性、无信息增益的 Mermaid
- 不要改写已经正确的版本尾注只为统一字面格式

## Out of Scope

- 当前 API 与统计修复
- 跨章运行时语义与安全边界
- 外部生态比较
- 新章节、目录重组或 server 独立章节

## Completion Criteria

<!-- lint-ack: error-path — 本任务为书稿结构、历史时态与构建门禁核查，无运行时失败路径 -->

场景: project spec 明确新增结构规则
  测试: review_project_spec_structure_rules
  当 检查 `specs/project.spec.md`
  那么 每章至少一张 Mermaid、定位锚点、版本演化说明和 40 行代码块上限均被明示
  并且 章节路径使用 `src/ch{chapter-id}-{slug}.md`
  并且 `chapter-id` 允许两位数字加可选小写字母后缀，例如 `26b`

场景: ch27-29 形成双端历史闭环
  测试: review_historical_chapters_head_and_tail
  当 检查 ch27、ch28、ch29 的章首与章尾
  那么 两端都声明 v0.66.1 commit `c779c14e` 历史基线
  并且 两端都说明对应包已移出当前仓库
  并且 两端都说明移出时间、当前继任情况与保留原因
  并且 正文使用历史时态

场景: 每章至少包含一张有效 Mermaid
  测试: script_mermaid_per_chapter
  当 对 `src/ch*.md` 统计 Mermaid fenced block
  那么 每章计数至少为 1
  并且 ch21、ch22、ch23、ch25、ch28、ch29、ch32 的新增图承载本章核心关系

场景: 所有代码块都不超过 40 行
  测试: script_fenced_block_length
  当 运行 `node scripts/check-chapter-structure.mjs`
  那么 每个连续代码块行数不超过 40
  并且 7 个当前超长块的必要语义仍可由片段和文字共同解释

场景: 每章保留定位和版本演化
  测试: script_positioning_and_evolution
  当 扫描 `src/ch*.md`
  那么 每章包含 `> **定位**` 锚点
  并且 每章包含章末「版本演化说明」

场景: 特殊章节仍明确记录取舍
  测试: review_tradeoffs_without_forced_headings
  当 检查 ch01、ch03、ch24、ch26b
  那么 每章关键决策明确写出得到与放弃
  并且 不要求仅为机械一致性新增“取舍分析”标题

场景: 结论章证据与附录 E2E trace 保持闭环
  测试: review_conclusions_and_e2e_traces
  当 检查 ch30-32 与 appendix
  那么 ch30-32 的关键当前结论有 `b4f29368` 源码锚点或明确回指技术章节
  并且 appendix 仍保留两条跨章 E2E trace
  并且 trace 中的当前 API 与路径对应 v0.82.1

场景: 书稿构建与 diff 门禁通过
  测试: script_mdbook_and_diff_gate
  当 运行 `mdbook build` 与 `git diff --check`
  那么 两个命令退出码均为 0
  并且 上级 pi-mono 源码没有变更

场景: live wiki 记录二次修复
  测试: review_wiki_repair_learning
  当 检查 `sync-v0797-v0821.md`
  那么 文档记录 repair 合约、关键源码证据、已关闭项与剩余风险

场景: 最终 caller resolution 所需证据齐备
  测试: review_evidence_ready_for_caller_resolution
  当 准备 project spec 与前三份内部 repair spec 的 caller decisions
  那么 每份合约都记录本任务实际触碰的显式 `--change` 清单
  并且 所有新增 file:line 已在 `b4f29368` 或 `c779c14e` 实测
  并且 已删除路径只出现在明确历史语境
  并且 fact-checker、tech-reviewer、structure-editor 的最终 verdict 均为 PASS
  并且 三视角报告 Critical 计数为 0
