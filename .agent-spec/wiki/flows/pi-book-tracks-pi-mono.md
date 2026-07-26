---
title: "pi-book 对 pi-mono 的版本同步流"
type: project-flow
flow_id: pi-book-tracks-pi-mono
projects:
  - pi-book
  - pi-mono
kind: documents
protocols: [git, markdown]
requirements:
  - REQ-SYNC
specs:
  - specs/update-v082-structural.spec.md
  - specs/update-v082-ai.spec.md
  - specs/update-v082-runtime.spec.md
  - specs/update-v082-coding-agent.spec.md
  - specs/update-v082-tui.spec.md
source_files:
  - CLAUDE.md
  - src/SUMMARY.md
  - knowledge/requirements/req-sync.md
external_sources:
  - https://github.com/earendil-works/pi
---

# pi-book 对 pi-mono 的版本同步流

pi-book 追踪 pi-mono 的 release 区间并周期性偿还漂移。流程由
[[pi-mono]] 的 CHANGELOG 驱动,受 REQ-SYNC 约束:

1. 以 release commit 界定区间,逐包提炼 changelog 为三类摘要
   (影响章节的变更 / 丰富点 / 噪音)。
2. 按包分组撰写 update 合约(lint ≥0.7),depends 链声明执行顺序。
3. 分波执行章节修订(共享文件的合约串行),每处新 file:line 实测核实。
4. 三视角评审(fact-checker / tech-reviewer / structure-editor),
   Critical 清零后提交。

历史轮次记录见 [[sync-v0797-v0821]]。
