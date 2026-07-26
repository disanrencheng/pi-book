---
title: "pi-mono"
type: external-project
project_id: pi-mono
repo: earendil-works/pi
role: "本书的分析对象:pi coding agent 的 TypeScript monorepo"
interfaces: [npm, cli]
protocols: [stdio, jsonl]
status: active
source_files:
  - CLAUDE.md
  - src/ch02-packages.md
external_sources:
  - https://github.com/earendil-works/pi
---

# pi-mono

pi-book 的分析对象,位于本仓库上级目录(pi-book 是其内的独立嵌套 git 仓库)。
当前对照版本 **v0.82.1**(release commit `b4f29368`)。

## 当前布局(v0.82.1)

7 个 workspace 业务包,lockstep 版本:

- `@earendil-works/pi-tui` / `pi-ai`(bin `pi-ai`)/ `pi-agent-core` /
  `pi-coding-agent`(bin `pi`)/ `pi-storage-sqlite-node` /
  `pi-server`(bin `server`,experimental)— 对外发布
- `@earendil-works/pi-evals` — private,vitest-evals 评测 harness

build 顺序:`tui → ai → agent → storage/sqlite-node → coding-agent → server`。

## 与书稿的版本对应

- 书核心分析基线:v0.66.0(commit `c779c14e`)
- 第一轮同步:→ v0.79.7(commit `c4ab61dc`,pi-book commit `0e6abbb`)
- 第二轮同步:→ v0.82.1(见 [[sync-v0797-v0821]])
