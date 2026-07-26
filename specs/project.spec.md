spec: project
name: "pi-book"
tags: [book, documentation, design]
---

## Intent

撰写一本关于 pi-mono 项目设计艺术的技术书籍。全书以设计决策为骨架，源码为佐证，
面向有工程经验、想真正理解 agent 系统设计的开发者。

## Constraints

- 每章以一个设计问题开头，回答"为什么这样做"而非"代码怎么写"
- 每个关键设计决策必须标注取舍（得到什么、放弃什么）
- 源码引用精准到文件和行号，只在需要解释设计时出场
- 中文写作，技术术语保留英文
- 每章 3000-6000 字，不超过 8000 字
- 每章必须保留 `> **定位**` 锚点与章末「版本演化说明」
- 每章至少一张 Mermaid 图，图必须承载本章核心流程、状态或关系
- 代码片段只展示设计相关的核心部分，单个连续代码块不超过 40 行

## Decisions

- 书名: 《pi 的设计艺术：构建生产级 Coding Agent 的架构决策》
- 大纲: 参照 `outline.md` (v2 修订版)
- 章节文件: `src/ch{chapter-id}-{slug}.md` 格式；`chapter-id` 为两位数字加可选小写字母
  后缀，例如 `26b`
- 图示: 使用 Mermaid 语法，放在章节内
- 当前事实基线: pi-mono v0.82.1（commit `b4f29368`）
- 历史案例基线: ch27-29 使用 pi-mono v0.66.1（commit `c779c14e`），不得冒充当前能力
- 源码引用格式: `packages/ai/src/models.ts:127-156`

## Boundaries

### Allowed Changes
- pi-book/**

### Forbidden
- 不要修改 pi-mono 源码
- 不要创建与书无关的文件
- 不要在章节中贴超过 40 行的连续代码块

## Completion Criteria

<!-- lint-ack: error-path — 本项目规范约束书稿结构与事实口径，场景均为静态核查型断言 -->

场景: 章节围绕设计问题与取舍展开
  测试: review_chapter_design_question_and_tradeoffs
  当 检查任一章节的开头与关键设计决策
  那么 章节以一个设计问题开头并回答为什么这样做
  并且 每个关键设计决策标注得到什么与放弃什么

场景: 章节语言、字数与源码证据符合规范
  测试: review_chapter_language_budget_and_evidence
  当 检查任一章节的正文与源码引用
  那么 正文使用中文且技术术语保留英文
  并且 章节为 3000-6000 字且不超过 8000 字
  并且 源码引用精准到文件和行号

场景: 章节结构与代码块满足机械门禁
  测试: script_chapter_structure_gate
  当 对 `src/ch*.md` 运行章节结构检查
  那么 每章包含 `> **定位**` 锚点、至少一张 Mermaid 图与章末「版本演化说明」
  并且 每个连续代码块不超过 40 行

场景: 当前章节与历史案例使用不同事实基线
  测试: review_current_and_historical_baselines
  当 检查当前事实章节与 ch27-29 的版本声明
  那么 当前事实以 pi-mono v0.82.1 commit `b4f29368` 为基线
  并且 ch27-29 只以 pi-mono v0.66.1 commit `c779c14e` 作为历史案例

场景: 书名、目录与引用格式保持一致
  测试: review_project_identity_and_paths
  当 检查项目规范、`outline.md` 与 `src/SUMMARY.md`
  那么 书名为《pi 的设计艺术：构建生产级 Coding Agent 的架构决策》
  并且 章节文件使用 `src/ch{chapter-id}-{slug}.md`
  并且 `chapter-id` 允许 `26b` 这类两位数字加小写字母后缀
  并且 图示使用 Mermaid 且源码引用采用 `packages/.../file.ts:line` 格式
