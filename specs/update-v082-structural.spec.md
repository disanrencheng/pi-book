spec: task
name: "更新v082-结构性漂移同步 (v0.79.7→v0.82.1)"
inherits: project
tags: [chapter, update, structural, v082]
satisfies: [REQ-SYNC]
estimate: 1d
---

## Intent

pi-book 上一轮已同步到 pi-mono v0.79.7（commit c4ab61dc），当前代码已到 v0.82.1
（commit b4f29368），其后共有 515 次提交、18 个 release。本任务修正**仓库布局层面**的
漂移：monorepo 从 4 个包扩张为 7 个 workspace 业务包（新增 server、storage/sqlite-node、
evals），build 拓扑从 4 步变 6 步。ch02「四个包」的核心命题需要续写为
「收缩之后再扩张」的完整曲线。

## Decisions

- 当前 7 个 workspace 业务包：`pi-tui`、`pi-ai`、`pi-agent-core`、`pi-coding-agent`、
  `pi-storage-sqlite-node`、`pi-server`、`pi-evals`（scope 均为 `@earendil-works/`），
  前 6 个对外发布，`pi-evals` 为 private
- 新包定位（写入 ch02）：`server` = 实验性常驻服务宿主（supervisor + RPC 子进程 + IPC，
  唯一运行时依赖 `pi-coding-agent`，坐在产品层之上）；`storage/sqlite-node` = 可选
  SQLite 会话存储后端（无发布包依赖它，仅 agent 测试使用）；`evals` = 基于 `vitest-evals`
  的私有评测 harness（#7085）
- `server` 的历史：目录以 `orchestrator` 名落地（commit 7ece19b0），v0.81.0 改名
  `server`（commit 8495f9d0，#6898）；README 声明 Experimental、可能变更或移除
- **不为 server 单开新章**：README 声明其 CLI/API 不稳定，书只在 ch02 登记定位，
  架构展开推迟到该包稳定后
- build 顺序更新为 6 步：`tui → ai → agent → storage/sqlite-node → coding-agent → server`；
  evals 不参与 build
- workspaces 数组新增 `"packages/storage/*"`；包间交叉依赖版本 lockstep 至 `^0.82.1`
- ch02「为什么是 4 个包」一节续写：3→7→4→7 的曲线用同一「独立使用者则独立成包」
  标尺解释扩张，不推翻原论点
- ch02 标题改为计数无关的「包不是项目」（避免包数再变时标题再漂移），
  SUMMARY.md 同步更新
- 全书基线声明策略：核心分析基线保持 v0.66.0 不变；preface/ch01 与各章「版本演化说明」
  的对照版本从 v0.79.7 延伸到 v0.82.1（各分册章节的尾注更新由对应 update-v082-* 合约负责）
- pi-book/CLAUDE.md：新增本轮 update-v082-* 合约表；源码参考清单补 server、
  storage/sqlite-node、evals 三条

## Boundaries

### Allowed Changes
- pi-book/src/preface.md
- pi-book/src/ch01-prologue.md
- pi-book/src/ch02-packages.md
- pi-book/src/ch03-reading-map.md
- pi-book/src/SUMMARY.md
- pi-book/CLAUDE.md

### Forbidden
- 不要修改 pi-mono 源码
- 不要为 server 包新建章节文件
- 不要把 private 的 evals 描述成对外发布的包
- 不要改动 ch27/28/29 的历史快照注记

## Out of Scope

- ai/agent/coding-agent/tui 各包的设计级内容更新（见 update-v082-ai /
  update-v082-runtime / update-v082-coding-agent / update-v082-tui）
- server 包内部架构（supervisor/IPC 协议）的展开分析

## Completion Criteria

<!-- lint-ack: error-path — 本任务为书稿事实核查与改写,场景均为核查型断言,无运行时失败路径 -->

场景: ch02 标题计数无关且 SUMMARY 同步
  测试: grep_ch02_title_countless
  当 检查 ch02 首行标题与 SUMMARY.md 对应条目
  那么 标题为「包不是项目」且两处一致
  并且 标题不再包含具体包数

场景: ch02 反映 7 包布局与发布状态
  测试: review_ch02_seven_packages
  假设 ch02-packages.md 已更新
  当 检查章节正文、规模表与分层图
  那么 正文登记 7 个 workspace 业务包且标注 6 个发布、evals 为 private
  并且 server 标注为 experimental 并给出「不单开章节」的理由
  并且 分层图含 server（第 5 层）与 sqlite-node（可选后端）节点

场景: ch02 build 拓扑与 workspaces 更新
  测试: review_ch02_build_topology
  当 检查 ch02 的 build 顺序描述与 workspaces 代码块
  那么 build 顺序为 "tui → ai → agent → storage/sqlite-node → coding-agent → server"
  并且 workspaces 数组包含 "packages/storage/*"

场景: ch02 不再声称当前只有四个包
  测试: grep_ch02_no_stale_four
  当 检查 ch02 中「当前包含 4 个」「只有内部的 4 个包才发布」类表述
  那么 此类表述改为 7 包/6 发布口径
  并且 「为什么是 4 个包」论证续写为收缩-再扩张曲线且沿用「独立使用者」标尺

场景: 版本与依赖号更新
  测试: grep_version_refs_updated
  当 在 ch02 中搜索 "^0.79.0" 与 "v0.79.7"
  那么 交叉依赖版本示例为 "^0.82.1"
  并且 lockstep 当前版本表述为 v0.82.1

场景: ch03 口径同步且无死路径
  测试: review_ch03_updated
  当 检查 ch03-reading-map.md 的包数与源文件口径
  那么 不再以「4 个包」作为当前框架表述
  并且 对 server/sqlite-node/evals 各有一句定位
  并且 文中引用的 packages/ 路径在当前仓库均存在

场景: preface 与 ch01 对照版本延伸
  测试: review_baseline_statement
  当 检查 preface.md 与 ch01-prologue.md 的版本基线段落
  那么 核心分析基线仍为 v0.66.0
  并且 声明已对照 v0.82.1（2026 年 7 月）核实并注明结构变化见第 2 章

场景: CLAUDE.md 登记本轮合约与新包
  测试: review_claudemd_v082
  当 检查 pi-book/CLAUDE.md
  那么 更新合约表含 update-v082-* 五份合约及依赖顺序
  并且 源码参考清单含 packages/server、packages/storage/sqlite-node、packages/evals
