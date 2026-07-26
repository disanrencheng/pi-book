# pi-book 写作指南

## 项目概述

本目录是《pi 的设计艺术：构建生产级 Coding Agent 的架构决策》一书的写作空间。

全书以 pi-mono 项目的**设计决策**为骨架，源码为佐证，面向有工程经验、想真正理解 agent 系统设计的开发者。

## 大纲

完整大纲见 [outline.md](outline.md)。Codex v1 大纲见 [outline-v1.md](outline-v1.md) 供参考。

## 章节 Spec（任务合约）

每篇/每组章节有一个 agent-spec 任务合约，定义了写作要求、验收标准和取舍约束。
**写任何章节前，必须先读对应的 spec。**

| Spec 文件 | 覆盖章节 | 预估工时 | 依赖 |
|-----------|---------|---------|------|
| [project.spec.md](specs/project.spec.md) | 全书约束 | - | - |
| [ch01-prologue.spec.md](specs/ch01-prologue.spec.md) | 第 1 章：序章 | 0.5d | - |
| [ch02-03-layering.spec.md](specs/ch02-03-layering.spec.md) | 第 2-3 章：分层的纪律 | 1d | - |
| [ch04-07-pi-ai.spec.md](specs/ch04-07-pi-ai.spec.md) | 第 4-7 章：pi-ai 设计 | 2d | ch02-03 |
| [ch08-10-runtime.spec.md](specs/ch08-10-runtime.spec.md) | 第 8-10 章：Agent Runtime | 2d | ch04-07 |
| [ch11-14-product.spec.md](specs/ch11-14-product.spec.md) | 第 11-14 章：产品化 | 2d | ch08-10 |
| [ch15-18-extensibility.spec.md](specs/ch15-18-extensibility.spec.md) | 第 15-18 章：能力外置 | 1.5d | ch11-14 |
| [ch19-23-tools.spec.md](specs/ch19-23-tools.spec.md) | 第 19-23 章：工具设计 | 2d | ch08-10 |
| [ch24-27-ui.spec.md](specs/ch24-27-ui.spec.md) | 第 24-27 章：UI 层 | 1.5d | ch11-14 |
| [ch28-29-products.spec.md](specs/ch28-29-products.spec.md) | 第 28-29 章：产品化实证 | 1d | ch11-14 |
| [ch30-32-philosophy.spec.md](specs/ch30-32-philosophy.spec.md) | 第 30-32 章：设计哲学 | 1.5d | ch15-18, ch19-23 |
| [appendix.spec.md](specs/appendix.spec.md) | 附录 A-F | 0.5d | ch08-10, ch19-23 |

### 更新合约（v0.66.1 → v0.79.7 同步）

基线 v0.66.1（commit c779c14e）当时落后于目标版本 v0.79.7。以下更新合约定义了该轮同步任务，按 `depends` 顺序执行：

| Spec 文件 | 覆盖 | 预估工时 | 依赖 |
|-----------|------|---------|------|
| [update-structural.spec.md](specs/update-structural.spec.md) | 7→4 包、scope 迁移、移出包 ch02/03/27/28/29 | 1d | - |
| [update-ai.spec.md](specs/update-ai.spec.md) | ch04-07,18：OAuth 5→3、thinkingLevelMap、图像生成 | 1.5d | update-structural |
| [update-runtime.spec.md](specs/update-runtime.spec.md) | ch08-10：terminate、turn 回调、AgentHarness | 1d | update-ai |
| [update-coding-agent.spec.md](specs/update-coding-agent.spec.md) | ch11-17,19-23,26：Project Trust、XML 边界、SDK 新章 | 3d | update-runtime |
| [update-tui.spec.md](specs/update-tui.spec.md) | ch24-25：Container.render、Kitty 图片、grapheme | 1d | update-coding-agent |

### 更新合约（v0.79.7 → v0.82.1 同步）

基线 v0.79.7（commit c4ab61dc）落后于当前 v0.82.1（commit b4f29368），其后共有 515 次提交、18 个 release。本轮更新合约同样按 `depends` 顺序执行：

| Spec 文件 | 覆盖 | 预估工时 | 依赖 |
|-----------|------|---------|------|
| [update-v082-structural.spec.md](specs/update-v082-structural.spec.md) | ch02/03，preface/ch01：4→7 包、server/sqlite-node/evals、build 6 步、^0.82.1、标题「包不是项目」 | 1d | - |
| [update-v082-ai.spec.md](specs/update-v082-ai.spec.md) | ch04-07,18：api-registry/stream 删除 → `Models`/`createProvider`、OAuth per-provider 子系统 | 2d | update-v082-structural |
| [update-v082-runtime.spec.md](specs/update-v082-runtime.spec.md) | ch08-10：runtime 层漂移 | 1d | update-v082-ai |
| [update-v082-coding-agent.spec.md](specs/update-v082-coding-agent.spec.md) | ch11-23,26,26b：产品层漂移 | 2.5d | update-v082-runtime |
| [update-v082-tui.spec.md](specs/update-v082-tui.spec.md) | ch24-25：Ctrl+J 换行、paste registry undo、ANSI 换行 | 0.5d | update-v082-coding-agent |

依赖链：`structural → ai → runtime → coding-agent → tui`。structural 无前置依赖，是本轮的入口。

### 内容修复合约（v0.82.1 二次收敛）

同步稿完成后发现的剩余事实、语义和结构问题由以下 repair 合约处理：

| Spec 文件 | 覆盖 | 预估工时 | 依赖 |
|-----------|------|---------|------|
| [repair-v082-facts-api.spec.md](specs/repair-v082-facts-api.spec.md) | README/outline、旧 API、类型/hook 示例、错误统计、当前二次开发入口 | 1.5d | - |
| [repair-v082-semantics-security.spec.md](specs/repair-v082-semantics-security.spec.md) | 跨章运行时语义与安全边界 | 2d | repair-v082-facts-api |
| [repair-v082-structure-history.spec.md](specs/repair-v082-structure-history.spec.md) | 历史章节闭环、Mermaid、代码块与 wiki | 1.5d | repair-v082-semantics-security |
| [repair-v082-ecosystem-comparison.spec.md](specs/repair-v082-ecosystem-comparison.spec.md) | 外部生态能力比较与官方来源 | 1d | repair-v082-structure-history |

内部修复关键路径为 `facts-api → semantics-security → structure-history`。外部生态比较使用独立事实源，可延期且不阻塞内部修复；若执行，必须在 structure-history 之后开始，避免并发改写重叠文件。

### 初始章节写作合约依赖（关键路径用粗线标注）

```
ch01-prologue (0.5d)
ch02-03-layering (1d) ═══> ch04-07-pi-ai (2d) ═══> ch08-10-runtime (2d) ═══> ch11-14-product (2d) ═══> ch15-18-extensibility (1.5d) ═══> ch30-32-philosophy (1.5d)
                                                          │                         │
                                                          ├──> ch19-23-tools (2d) ──┤──> appendix (0.5d)
                                                          │                         │
                                                          │                         ├──> ch30-32-philosophy
                                                          │
                                                          ├──> ch24-27-ui (1.5d)
                                                          └──> ch28-29-products (1d)
```

**关键路径**: ch02-03 → ch04-07 → ch08-10 → ch11-14 → ch15-18 → ch30-32（总计 10d）

## 写作规则

1. **每章以设计问题开头** — 不是"这个模块有什么"，而是"为什么要这样设计"
2. **每个关键决策标注取舍** — 得到什么、放弃什么
3. **源码只在需要时出场** — 引用格式：`packages/ai/src/models.ts:127-156`
4. **中文写作，术语保留英文** — 如 provider, stream, tool call, compaction
5. **每章 3000-6000 字** — 不超过 8000 字
6. **代码块不超过 40 行** — 只展示设计核心，不贴完整实现
7. **每章至少一张 Mermaid 图** — 图必须承载核心流程、状态或关系，放在章节内，不创建单独图片文件
8. **章节文件命名** — `src/ch{chapter-id}-{slug}.md`，chapter-id 为两位数字加可选小写字母后缀（如 `26b`）
9. **目录结构由 `src/SUMMARY.md` 控制** — 修改章节顺序或添加新章节时同步更新

## 构建与预览

```bash
# 构建 HTML 书籍
cd pi-book && mdbook build

# 本地预览（自动刷新）
cd pi-book && mdbook serve --open
```

## 写作前的检查

以下命令从上级 pi-mono 仓库根目录运行：

```bash
# 初始化验证产物目录
mkdir -p pi-book/.agent-spec/reviews/repair-v082 pi-book/docs/superpowers/reviews

# 读取对应 spec
agent-spec contract pi-book/specs/ch04-07-pi-ai.spec.md

# 检查语法和质量门槛
agent-spec parse pi-book/specs/ch04-07-pi-ai.spec.md
agent-spec lint pi-book/specs/ch04-07-pi-ai.spec.md --min-score 0.7

# 脏工作树中只验证本任务实际触碰的路径；每个路径重复一个 --change
agent-spec lifecycle pi-book/specs/ch04-07-pi-ai.spec.md --code pi-book \
  --ai-mode caller --review-mode strict --format json \
  --change pi-book/src/ch04-provider-registry.md \
  > pi-book/.agent-spec/reviews/repair-v082/example.lifecycle.json

# 写入 caller decisions 后合并最终 verdict
agent-spec resolve-ai pi-book/specs/ch04-07-pi-ai.spec.md --code pi-book \
  --decisions pi-book/.agent-spec/reviews/repair-v082/example.decisions.json \
  --format json \
  > pi-book/.agent-spec/reviews/repair-v082/example.report.json
```

不得在当前脏工作树上使用 `--change-scope worktree` 代替显式清单。caller 尚未 resolve 时 lifecycle 可因 skip 非零退出，但其 boundary 必须 PASS 且 failed 为 0。review 型场景必须由 fact-checker、tech-reviewer、structure-editor 给出明确 verdict，并把 lifecycle、决策与最终报告保存在 `pi-book/.agent-spec/reviews/`。独立 post-resolve gate 要求 lifecycle boundary PASS、final failed/skipped/uncertain 为 0、Critical 为 0；合约场景不得自引用尚未生成的最终报告。`pending-ai-requests.json` 为共享文件，必须逐份完成 lifecycle → review → resolve-ai 后再处理下一份。`agent-spec verify` 返回 `skip` 不计为通过。

## 源码参考

写作时需要参考的 pi-mono 源码在上级目录：

- `../packages/ai/` — pi-ai 层
- `../packages/agent/` — pi-agent-core 层
- `../packages/coding-agent/` — pi-coding-agent 层
- `../packages/tui/` — pi-tui 层
- `../packages/server/` — pi-server 层（experimental：常驻服务宿主，supervisor + RPC 子进程 + IPC，坐在 coding-agent 之上）
- `../packages/storage/sqlite-node/` — pi-storage-sqlite-node（可选 SQLite 会话存储后端，无已发布包依赖它）
- `../packages/evals/` — pi-evals（private：基于 vitest-evals 的评测 harness，不发布）

> 注：`mom`、`pods`、`web-ui` 三个包已从主仓库移出（详见第 2 章及 ch27/28/29 章首注记），不再位于 `../packages/` 下。第 27/28/29 章的源码引用应以 v0.66.1 历史快照（commit `c779c14e`）为准。
