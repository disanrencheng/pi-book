spec: task
name: "修复v082-外部生态比较"
inherits: project
tags: [chapter, repair, comparison, ecosystem, v082]
satisfies: [REQ-SYNC]
depends: [repair-v082-structure-history]
estimate: 1d
---

## Intent

独立核验并重写书中对 LangChain、Vercel AI SDK、Claude Code、Aider、CrewAI、
AutoGen、Cursor、Windsurf 等外部工具的能力比较。该任务只使用执行时可访问的
官方一手资料并标注核验日期；它不阻塞 pi-mono 内部事实、语义和结构修复。

## Decisions

- 外部能力只依据执行时读取的官方文档、官方仓库或官方发布说明
- 每组比较明确限定版本或核验日期、比较维度和适用范围
- 不使用“没有循环引擎”“只是调用层”“完全做不到”等无范围的绝对判断
- 比较同时描述双方在同一维度上的公开能力与限制，不用缺失信息替代证据
- 无法从官方来源确认的能力删除、降为待核验问题或明确标注未知，不做推测
- 本合约必须在 structure-history 之后执行，避免与内部合约并发改写同一文件；
  也可以在前三份内部 repair 合约完成后延期
- 外部比较修订不得回退前三份内部合约已经关闭的 API、语义、安全或结构修复

## Boundaries

### Allowed Changes
- pi-book/outline.md
- pi-book/src/ch01-prologue.md
- pi-book/src/ch30-minimal-core.md
- pi-book/src/ch31-contrarian-choices.md
- pi-book/src/ch32-boundaries.md
- pi-book/.agent-spec/reviews/repair-v082/ecosystem-comparison.decisions.json
- pi-book/.agent-spec/reviews/repair-v082/ecosystem-comparison.lifecycle.json
- pi-book/.agent-spec/reviews/repair-v082/ecosystem-comparison.report.json
- pi-book/docs/superpowers/reviews/2026-07-26-pi-book-v082-content-repair-review.md

### Forbidden
- 不要修改 pi-mono 源码
- 不要使用搜索摘要、二手博客或模型记忆作为唯一事实来源
- 不要无引用地保留绝对能力判断
- 不要为制造对比而缩窄外部工具的公开能力

## Out of Scope

- pi-mono v0.82.1 内部 API、语义、安全和结构修复
- 产品选型评分、采购建议或性能基准
- 对未公开内部实现的推断

## Completion Criteria

<!-- lint-ack: error-path — 本任务为官方来源核验与技术比较写作，场景均为审查型断言 -->

场景: 每个外部能力主张都有官方来源
  测试: review_external_claim_sources
  当 检查本合约修改的比较段落
  那么 每个可变能力主张附近存在官方文档、官方仓库或官方发布说明链接
  并且 文中标注统一的核验日期

场景: 比较维度和范围明确
  测试: review_comparison_scope
  当 检查 LangChain、Vercel AI SDK、Claude Code、Aider、CrewAI、AutoGen、
  Cursor 与 Windsurf 的比较
  那么 每组比较说明具体维度与适用范围
  并且 双方在同一维度上的能力与限制均被描述

场景: 无依据的绝对判断被移除
  测试: grep_no_unsupported_external_absolutes
  当 搜索“没有循环引擎”“只是调用层”“完全做不到”等绝对表述
  那么 每处要么删除，要么收窄到有官方来源支持的具体范围

场景: 无法核实的内容不会被猜测替代
  测试: review_unknown_external_capabilities
  当 某项外部能力无法从官方来源确认
  那么 该项被删除、标为未知或记录为待核验
  并且 不使用二手来源补成确定事实

场景: 外部比较不回退内部修复
  测试: review_ecosystem_preserves_internal_repairs
  当 检查本合约产生的变更与前三份 repair 合约最终报告
  那么 当前 API、运行时语义、安全边界和结构门禁仍保持通过
  并且 本合约没有与内部合约并发修改同一文件
