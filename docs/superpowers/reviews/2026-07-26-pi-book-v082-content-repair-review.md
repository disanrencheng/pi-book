# pi-book v0.82.1 内容修复治理评审

## 范围

本报告只评审内容修复的治理基线，不代表正文修复已经完成。评审范围：

- `docs/superpowers/specs/2026-07-26-pi-book-v082-content-repair-design.md`
- `specs/project.spec.md`
- 四份 `specs/repair-v082-*.spec.md`
- `CLAUDE.md`
- `.agent-spec/wiki/learnings/sync-v0797-v0821.md`

## 第一轮 verdict

fact-checker、tech-reviewer、structure-editor 均给出 `NEEDS_CHANGES`。阻塞项集中在：

1. 把仍公开的 compat `registerApiProvider` 错写成仅历史能力。
2. 把 Anthropic adapter 的可变 accumulator 推广成所有 provider 的协议保证。
3. facts 合约漏掉 ch18 与跨章类型/hook 示例，安全合约漏验 AGENTS.md 路径。
4. ecosystem 合约可与内部关键路径并发改写相同文件。
5. project spec、structure 合约和 CLAUDE.md 对代码块范围、ch26b 命名、附录数量不一致。
6. caller 验证缺少显式 change set、boundary 产物、resolve report 和零 skip 的 post-resolve gate。
7. 自身 report 被写成自身验收条件，且可选 ecosystem 与 structure 形成闭环依赖。

## 已关闭项

- `registerApiProvider` 统一为 v0.82.1 的临时 compat/迁移 API；规范主线为
  `Models`、`createModels`、`createProvider` 与 `providers/*`。
- `partial` 统一为发射时的累计消息状态；协议不保证对象身份或不可变性，并区分
  Anthropic 同对象复用与 faux 浅拷贝。
- facts 合约改为全书当前 API 扫描并纳入 ch04–10、ch13–19、ch25 与旧章节 spec。
- semantics 合约补齐九条语义、五类安全边界和完整 `b4f29368` 证据锚点。
- ecosystem 改为依赖 structure-history；延期时保持 open，不阻塞内部闭环。
- 代码块门禁统一扫描所有语言；当前 7 个违规块恰好都是 TypeScript。
- 章节命名改为 `src/ch{chapter-id}-{slug}.md`，明确允许 `ch26b`。
- caller 命令统一从 pi-mono 根运行，使用 `--code pi-book` 和显式
  `--change pi-book/...`。
- lifecycle、decisions、resolve report 分别落盘；独立 post-resolve gate 同时要求
  lifecycle boundary PASS、final failed/skipped/uncertain 为 0、三视角 Critical 为 0。
- `pending-ai-requests.json` 按合约串行 lifecycle → review → resolve，避免共享文件覆盖。

## 最终 verdict

| 视角 | Verdict | Critical |
|---|---|---:|
| fact-checker | PASS | 0 |
| tech-reviewer | PASS | 0 |
| structure-editor | PASS | 0 |

治理基线 verdict：**PASS**。

## 已执行验证

- project spec：5 个场景，lint 83%，通过 0.7 门槛。
- facts-api：7 个场景，lint 100%。
- semantics-security：9 个场景，lint 100%。
- structure-history：10 个场景，lint 100%。
- ecosystem-comparison：5 个场景，lint 100%。
- `agent-spec graph` 显示内部链
  `facts-api → semantics-security → structure-history → ecosystem-comparison`。
- project 与四份 repair spec 各用一个允许路径实测 boundary，均为 PASS；内容场景尚未
  resolve，因此命令整体按预期非零。
- 当前机械重扫：7 章缺 Mermaid，7 个 fenced code block 超过 40 行；所有章节已有
  定位锚点和版本演化说明。
- `git diff --check` 通过。

## 尚未执行

- facts-api、semantics-security、structure-history 的正文修复。
- 内容级 caller lifecycle/resolve-ai 最终报告。
- 最终 `mdbook build` 和内容级三视角评审。
- 可延期的外部生态官方来源核验。

这些未执行项仍为 open，不得用本报告的治理 PASS 代替。
