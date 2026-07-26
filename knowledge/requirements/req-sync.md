---
kind: requirement
id: REQ-SYNC
title: "pi-book 与 pi-mono 的版本同步纪律"
status: accepted
---

# pi-book 与 pi-mono 的版本同步纪律

## Problem

pi-book 分析的是活跃演进中的 pi-mono(月均 200+ 提交、多次 breaking 重构)。
不设机制的书稿会静默腐化:file:line 失效、已删机制被当现状、示例代码无法运行。
v0.66.1→v0.79.7 与 v0.79.7→v0.82.1 两轮同步证明:漂移必须周期性、合约化地偿还。

## Requirements

- [REQ-SYNC-BASELINE] 每轮同步 MUST 以 release commit 界定版本区间,并逐包把 CHANGELOG 区间条目提炼为「影响章节的变更/丰富点/噪音」三类摘要。
- [REQ-SYNC-CONTRACT] 章节修订 MUST 由 lint 分数不低于 0.7 的 update 任务合约驱动,合约按包分组并以 depends 链声明执行顺序。
- [REQ-SYNC-EVOLUTION] 受影响章节 MUST 在章末「版本演化说明」写明已对照的版本号,并对 breaking 变更注明版本分界。
- [REQ-SYNC-VERIFY] 本轮新增的每处 file:line 引用 MUST 在写作会话中打开对应源文件实测核实;引用已删除文件 MUST 置于明确标注的历史对照语境。
- [REQ-SYNC-REVIEW] 每轮同步收尾 MUST 经 fact-checker、tech-reviewer、structure-editor 三视角评审,Critical 项 MUST 在提交前修复。

## Scenarios

场景: 一轮同步的完成形态
  假设 pi-mono 出现新的 release 区间
  当 执行一轮同步并收尾
  那么 specs/ 目录存在覆盖全部受影响章节的 update 合约且 agent-spec lint 退出码为 0
  并且 每个受影响章节文件包含「版本演化说明」且写明对照版本号
  并且 三视角评审报告中 Critical 计数为 0
  并且 mdbook build 退出码为 0

场景: 引用已删除文件而未标注历史语境被评审拦下
  假设 某章新增引用指向当前源码树中已不存在的文件且未标注历史对照
  当 fact-checker/tech-reviewer 评审本轮 diff
  那么 该处被记为 Critical
  并且 在修复该项之前本轮同步不得提交

## Dependencies

- 无前置需求文档。

## Source Trace

- specs/update-structural.spec.md 等 5 份(v0.66.1→v0.79.7 轮,2026-06)
- specs/update-v082-structural.spec.md 等 5 份(v0.79.7→v0.82.1 轮,2026-07)
- pi-book commit 0e6abbb(上一轮同步提交)
