# 第 26 章：RPC 模式 — pi 作为后端服务

> **定位**：本章解析 pi 的 RPC 模式 — 让 CLI 工具可以被其他进程驱动。
> 前置依赖：第 10 章（Agent 类）。
> 适用场景：当你想把 pi 集成到 IDE 插件、Web 前端或自动化管道中。

## 为什么 CLI 工具需要 RPC 模式？

pi 的主要界面是终端 TUI（第 24 章）。但有些场景不需要终端交互：

- **IDE 插件**：VS Code extension 需要通过进程间通信驱动 pi
- **Web 前端**：浏览器客户端需要通过 HTTP 或 WebSocket 驱动 pi
- **自动化管道**：CI/CD 脚本需要非交互式地使用 pi

RPC 模式（`packages/coding-agent/src/modes/rpc/`）提供了一个 JSON-L 协议，让外部进程可以：

1. 发送 prompt 和 steer/follow-up 请求
2. 接收流式事件（和 Agent 事件一一对应）
3. 切换模型、调整 thinking level
4. 管理会话（创建、切换、fork、导出）

```mermaid
graph LR
    IDE["IDE 插件\n(进程 A)"] -->|"JSON-L stdin"| RPC["pi --mode rpc\n(进程 B)"]
    RPC -->|"JSON-L stdout"| IDE
    RPC --> Agent["Agent"]
    Agent --> Loop["agentLoop"]
    
    style IDE fill:#e3f2fd
    style RPC fill:#fff3e0
```

RPC 模式和 interactive 模式共享同一个 Agent 实例（第 10 章）。不同的只是事件的消费方式 — interactive 模式渲染到终端，RPC 模式序列化成 JSON-L 输出到 stdout。

## 协议设计：JSON Lines over stdio

RPC 使用 JSON Lines 协议 — 每条消息是一行 JSON，以换行符分隔。选择 JSON-L over stdio 而非 HTTP/WebSocket 的理由：

1. **零配置**。不需要端口分配、不需要 TLS、不需要服务发现
2. **进程生命周期绑定**。父进程退出时子进程的 stdin 关闭，RPC 会话自然终止
3. **双向通信**。stdin 发命令，stdout 收响应和事件
4. **调试友好**。JSON Lines 可以用 `jq` 解析，管道可以用 `tee` 录制

## RPC 命令类型

命令通过 stdin 发送，每条命令是一个 JSON 对象，必须包含 `type` 字段：

```typescript
// packages/coding-agent/src/modes/rpc/rpc-types.ts:19-68
export type RpcCommand =
  // Prompting
  | { id?: string; type: "prompt"; message: string;
      images?: ImageContent[];
      streamingBehavior?: "steer" | "followUp" }
  | { id?: string; type: "steer"; message: string;
      images?: ImageContent[] }
  | { id?: string; type: "follow_up"; message: string;
      images?: ImageContent[] }
  | { id?: string; type: "abort" }
  | { id?: string; type: "new_session";
      parentSession?: string }

  // Model
  | { id?: string; type: "set_model";
      provider: string; modelId: string }
  | { id?: string; type: "cycle_model" }
  | { id?: string; type: "get_available_models" }

  // Thinking
  | { id?: string; type: "set_thinking_level";
      level: ThinkingLevel }

  // Session
  | { id?: string; type: "get_session_stats" }
  | { id?: string; type: "export_html";
      outputPath?: string }
  | { id?: string; type: "fork"; entryId: string }
  | { id?: string; type: "get_messages" }

  // ... 更多命令类型
```

随产品演进，命令清单已远不止上面这些。除了 `prompt`/`steer`/`follow_up`/`abort`/`set_model`/`fork`/`export_html` 等早期命令，现在还包括（rpc-mode.ts）：

- **会话**：`switch_session`（切换到已有会话）、`clone`（复制当前活动分支到新文件，与 `fork` 区别见第 11 章）、`get_last_assistant_text`、`get_commands`（列出可用命令）
- **只读会话树（v0.80.3）**：`get_entries`（带 `since` 增量拉取此后追加的 entry）、`get_tree`（返回完整 `SessionTreeNode[]`）—— 让外部宿主也能渲染第 11 章那棵会话树（`rpc-types.ts:64-65`）
- **压缩与重试**：`compact`（可带 `customInstructions`）、`set_auto_compaction`、`set_auto_retry` / `abort_retry`
- **执行控制**：`abort_bash`、`cycle_thinking_level`、`set_steering_mode` / `set_follow_up_mode`
- **思考档位**：`get_available_thinking_levels`（`rpc-types.ts:39`，返回当前模型支持的档位，配套 `RpcClient.getAvailableThinkingLevels()`；`max`/`xhigh` 也在此暴露）

`fork` 与 `clone` 的区别值得强调：`fork` 从某条更早的消息开新分支，`clone` 复制当前整条活动路径——对应第 11 章的 `/fork` 与 `/clone`。

几个设计要点：

**`id` 是可选的**。客户端可以为每条命令指定一个 correlation ID，响应会带上同一个 ID，让客户端可以匹配请求和响应。对于不需要匹配的场景（比如 fire-and-forget 的 abort），可以省略 ID。

**prompt 的三种变体**。`prompt` 是标准请求；`steer` 在 agent 思考过程中插入修正；`follow_up` 在 agent 完成后追加问题。这三种操作对应 Agent API 中不同的消息队列，RPC 直接暴露了这个区分。

**`streamingBehavior`**。`prompt` 命令可以指定 `streamingBehavior: "steer"` 或 `"followUp"`，让客户端控制当 agent 正在处理时新消息应该进入哪个队列。这和 interactive 模式中的"用户在 agent 思考时输入新消息"是同一个语义。

## RPC 响应类型

响应通过 stdout 发送，所有响应共享统一的信封格式：

```typescript
// packages/coding-agent/src/modes/rpc/rpc-types.ts:110-204
export type RpcResponse =
  // 成功响应 (部分)
  | { id?: string; type: "response"; command: "prompt";
      success: true }
  | { id?: string; type: "response"; command: "get_state";
      success: true; data: RpcSessionState }
  | { id?: string; type: "response"; command: "set_model";
      success: true; data: Model<any> }
  | { id?: string; type: "response"; command: "bash";
      success: true; data: BashResult }
  | { id?: string; type: "response"; command: "get_messages";
      success: true; data: { messages: AgentMessage[] } }

  // 错误响应 (任何命令都可能失败)
  | { id?: string; type: "response"; command: string;
      success: false; error: string };
```

响应的 TypeScript 类型是一个 discriminated union — 每种命令有自己的成功响应类型（带不同的 `data` 结构），但所有命令共享同一个错误响应类型。这让客户端可以先检查 `success` 字段，再根据 `command` 字段解析 `data`。

**异步命令和同步命令的区别**。`prompt`、`steer`、`follow_up` 的成功响应不包含 data — 因为这些操作是异步的，真正的结果通过后续的事件流（AgentSessionEvent）传递。`get_state`、`get_messages` 是同步查询，结果直接在响应的 `data` 中返回。

**`prompt` 的"单一权威响应" + preflight 契约（v0.67.3）**。`prompt` 命令对响应有一个明确约定：它**只回一次** `response`，且这个响应是一次 **preflight 裁决** —— 表示命令被 accepted / queued / handled。一旦命令被 accept，后续即便发生失败，也**只通过事件流**汇报，**不再发第二个 `response`**。换句话说，客户端可以安全地"一条 prompt 命令 ↔ 一个 response"地配对，而不必担心同一命令收到两次终态响应。这避免了早期"先 ack 再 error" 造成的客户端状态机混乱。另外，对**未知命令**的错误响应现在会带上原命令的 `id`（v0.79.7），让客户端能把错误关联回具体请求。

### 事件流：命令之外的增量信号

异步命令的真正结果走的是事件流（`AgentSessionEvent` 序列化成 JSON-L）。这条流本区间多了两类事件：

- **`bash_execution_update`**（`agent-session.ts:181`，v0.82.0）：直连 RPC 执行 bash 时，命令输出**边产生边流式**吐给客户端（`{ type: "bash_execution_update", id?, delta }`），而不是等命令结束才一次性给结果 —— 让 IDE 能像 TUI 一样实时显示 bash 输出（对应第 22 章的流式 `onData`）。
- **`summarization_retry_*`**（`agent-session.ts:167-181`，v0.81.1）：压缩/分支摘要失败重试时，`summarization_retry_scheduled` / `_attempt_start` / `_finished` 三个事件把重试过程透出给客户端（对应第 12 章的可恢复压缩）。客户端据此能显示"正在重试压缩"而非静默卡住。

## 会话状态

```typescript
// packages/coding-agent/src/modes/rpc/rpc-types.ts:90-103
export interface RpcSessionState {
  model?: Model<any>;
  thinkingLevel: ThinkingLevel;
  isStreaming: boolean;
  isCompacting: boolean;
  steeringMode: "all" | "one-at-a-time";
  followUpMode: "all" | "one-at-a-time";
  sessionFile?: string;
  sessionId: string;
  sessionName?: string;
  autoCompactionEnabled: boolean;
  messageCount: number;
  pendingMessageCount: number;
}
```

`RpcSessionState` 是 RPC 客户端了解 agent 状态的主要手段。通过 `get_state` 命令获取，包含了 UI 渲染需要的全部信息：当前模型、是否在流式输出、队列中有多少待处理消息。

`steeringMode` 和 `followUpMode` 控制消息队列的行为：`"all"` 表示累积所有消息一起处理，`"one-at-a-time"` 表示逐条处理。这影响了 IDE 插件的 UX 设计 — 如果用户快速发送多条消息，客户端可以选择是让它们排队还是合并。

## 会话管理

RPC 提供了完整的会话管理能力：

- **`new_session`**：创建新会话，可选地从指定父会话继承上下文
- **`switch_session`**：切换到已有会话
- **`fork`**：从指定消息处分叉对话（类似 git branch）
- **`get_fork_messages`**：获取可用于 fork 的消息列表
- **`export_html`**：导出当前会话为 HTML 文件
- **`set_session_name`**：给会话命名

`fork` 操作是一个高级功能 — 用户可以回到对话中的某个节点，从那里开始新的对话分支。这在 interactive 模式中通过 TUI 交互实现，在 RPC 模式中通过 `fork` 命令 + `entryId` 参数实现。`get_fork_messages` 返回可 fork 的消息列表和它们的 ID。

## Extension UI 桥接

RPC 模式需要处理一个特殊问题：extension 的 UI 交互。在 interactive 模式中，extension 通过 UI context 的 `select()`、`confirm()`、`input()`、`notify()`、`setStatus()`、`editor()` 等方法与用户交互。但 RPC 模式没有 TUI — 这些交互需要被序列化为 JSON 请求，发送给 RPC 客户端处理。

```typescript
// packages/coding-agent/src/modes/rpc/rpc-types.ts:211-246
export type RpcExtensionUIRequest =
  | { type: "extension_ui_request"; id: string;
      method: "select"; title: string;
      options: string[]; timeout?: number }
  | { type: "extension_ui_request"; id: string;
      method: "confirm"; title: string;
      message: string; timeout?: number }
  | { type: "extension_ui_request"; id: string;
      method: "input"; title: string;
      placeholder?: string; timeout?: number }
  | { type: "extension_ui_request"; id: string;
      method: "editor"; title: string;
      prefill?: string }
  | { type: "extension_ui_request"; id: string;
      method: "notify"; message: string;
      notifyType?: "info" | "warning" | "error" }
  | { type: "extension_ui_request"; id: string;
      method: "setStatus"; statusKey: string;
      statusText: string | undefined }
  | { type: "extension_ui_request"; id: string;
      method: "setWidget"; widgetKey: string;
      widgetLines: string[] | undefined }
  | { type: "extension_ui_request"; id: string;
      method: "set_editor_text"; text: string }
```

流程是：extension 发起 UI 请求 → RPC 层序列化为 `RpcExtensionUIRequest` 输出到 stdout → 客户端展示 UI → 客户端发送 `RpcExtensionUIResponse` 到 stdin → RPC 层传回 extension。

```typescript
// packages/coding-agent/src/modes/rpc/rpc-types.ts:253-256
export type RpcExtensionUIResponse =
  | { type: "extension_ui_response"; id: string;
      value: string }
  | { type: "extension_ui_response"; id: string;
      confirmed: boolean }
  | { type: "extension_ui_response"; id: string;
      cancelled: true };
```

这是一个完整的 UI 代理模式 — extension 不知道也不关心 UI 是在终端还是在浏览器中渲染的。

## RPC 模式的启动

```typescript
// packages/coding-agent/src/modes/rpc/rpc-mode.ts:46-53
export async function runRpcMode(
  runtimeHost: AgentSessionRuntime
): Promise<never> {
  takeOverStdout();
  let session = runtimeHost.session;
  let unsubscribe: (() => void) | undefined;

  const output = (
    obj: RpcResponse | RpcExtensionUIRequest | object
  ) => {
    writeRawStdout(serializeJsonLine(obj));
  };
```

除了 `pi --mode rpc` 这个 CLI 入口，包还导出了一个 **`./rpc-entry`** 子路径入口（v0.80.3）：`import "@earendil-works/pi-coding-agent/rpc-entry"` 会直接以 RPC 模式启动整个进程。这对"想 spawn 一个纯 RPC 子进程、又不想拼 CLI 参数"的宿主很方便 —— 子进程的 entry 就是这一个 import。

`takeOverStdout()` 劫持了 `console.log` 和 `process.stdout.write`，防止其他代码意外向 stdout 写入非 JSON 内容。这是 RPC 模式的核心防御 — stdout 是协议通道，任何非 JSON 输出都会破坏客户端的解析。

`writeRawStdout` 绕过了劫持层直接写入 stdout。只有 RPC 层自己可以往 stdout 写数据。

## Pending Extension Requests

```typescript
// packages/coding-agent/src/modes/rpc/rpc-mode.ts:71-74
const pendingExtensionRequests = new Map<
  string,
  { resolve: (value: any) => void;
    reject: (error: Error) => void }
>();
```

Extension UI 请求是异步的 — RPC 层发送请求后需要等待客户端响应。`pendingExtensionRequests` Map 以请求 ID 为 key 保存了 Promise 的 resolve/reject 回调。当客户端的 `extension_ui_response` 到达时，查找对应的 pending request 并 resolve。

这个 Map 也处理了超时和取消 — 如果 extension 设置了 timeout，超时后 pending request 会被 reject；如果客户端发送 `cancelled: true`，request 也会被相应处理。

## Agent API 到 RPC 的映射

RPC 命令和 Agent API 的对应关系：

| RPC Command | Agent API |
|---|---|
| `prompt` | `session.prompt()` |
| `steer` | `session.steer()` |
| `follow_up` | `session.followUp()` |
| `abort` | `session.abort()` |
| `set_model` | `session.setModel()` |
| `compact` | `session.compact()` |
| `get_messages` | `session.getMessages()` |
| `bash` | direct bash execution |
| `fork` | `session.fork()` |

这个映射几乎是一对一的。RPC 层不添加业务逻辑 — 它只做序列化/反序列化和 stdout 保护。这种薄层设计让 RPC 的行为和 interactive 模式完全一致。其中 `bash` 命令也透传了 `excludeFromContext` 选项（v0.76.0）——RPC 客户端可以执行命令、拿到输出、但让输出不进入 LLM 上下文（对应第 22 章的 `!!` 前缀）。

## 自带 typed client：`RpcClient`

外部进程当然可以自己手写 JSON-L 的收发，但 pi 从 v0.67.67 起**直接从包根导出了一个 typed client** `RpcClient`（连同 `RpcClientOptions`，index.ts:315）。它封装了"spawn 子进程、按行 framing、按 `id` 关联请求与响应、订阅事件流"这些样板，让 Node 调用方用类型安全的方法调用代替手拼 JSON。子进程退出时，所有 pending 请求会被 reject（v0.76.0），不会让调用方永久挂起。

> **LF framing 陷阱**：RPC 是严格的"一行一条消息"协议，分隔符是 `\n`。这里有一个**不能用 Node 内置 `readline`** 的坑 —— `readline` 会把 Unicode 行分隔符 `U+2028`/`U+2029` 也当成换行，而这些字符完全可能出现在 LLM 生成的内容里，一旦被误当行边界，整条 JSON 就被切碎、解析失败。因此 RPC 的 framing 必须严格只按 `\n` 切分。配套地，往 stdout 写时还要处理背压：写满时重试 / flush，避免大块事件输出被截断（v0.76.0）。如果你要自己实现客户端而不用 `RpcClient`，这是必须自己复刻的细节。

## 取舍分析

### 得到了什么

**pi 可以被任何前端驱动**。同一个 agent 内核，终端用户通过 TUI 交互，IDE 用户通过 RPC 交互。代码复用，行为一致。

**Extension UI 的透明代理**。Extension 不需要为不同的 UI 后端写不同的代码 — RPC 层自动桥接 UI 交互。

### 放弃了什么

**增加了一个运行模式的维护成本**。RPC 协议需要版本管理、backward compatibility、错误序列化。每次 Agent API 变化，RPC 层都需要同步更新。

**stdout 污染是隐蔽的 bug 源**。任何第三方库的 `console.log` 都可能破坏 RPC 协议。`takeOverStdout` 是必要的防御，但它也让调试变得更困难 — 你不能用 `console.log` 来 debug RPC 模式。

---

### 版本演化说明
> 本章核心分析基于 pi-mono v0.66.0，已对照 **v0.82.1**。RPC 的 JSON-L over stdio、薄层映射、Extension UI 桥接保持不变。
> 主要演进：① 命令清单大幅扩展，新增只读会话树 `get_entries`（带 `since`）/`get_tree`（v0.80.3，`rpc-types.ts:64-65`）与 `get_available_thinking_levels`（v0.81.0，`:39`），以及 `clone`/`switch_session`/`compact`/`set_auto_compaction`/`set_auto_retry`/`abort_retry`/`abort_bash`/`get_commands`/`cycle_thinking_level` 等；② 事件流新增流式 `bash_execution_update`（v0.82.0）与 `summarization_retry_*`（v0.81.1，`agent-session.ts:167-181`）；③ `./rpc-entry` 包导出直接以 RPC 模式启动进程（v0.80.3）；④ `prompt` 改为单一权威响应 + preflight 契约（v0.67.3），未知命令错误响应带 `id`（v0.79.7）；⑤ 从包根导出 typed `RpcClient`（v0.67.67），子进程退出时 reject pending 请求（v0.76.0）；⑥ `bash` 透传 `excludeFromContext`（v0.76.0）；⑦ 严格 LF framing（不可用 Node `readline`）+ stdout 背压处理。
> 若要把 pi 当库**嵌入**同一进程（而非 spawn 子进程驱动），见第 26b 章 SDK。
