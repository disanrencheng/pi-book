# 第 10 章：`Agent` — 循环之上的有状态壳

> **定位**：本章解析为什么一个无状态循环引擎之上还需要一个有状态的 `Agent` 类。
> 前置依赖：第 8 章（agentLoop）、第 9 章（工具执行管道）。
> 适用场景：当你想理解"循环"和"运行时对象"为什么必须分开，或者想为自己的 agent 系统设计状态管理。

## 为什么循环引擎不够？

第 8 章展示了 `agentLoop` 的无状态设计。但一个真正可用的 agent 需要更多：

- 它需要**记住**对话历史（transcript）
- 它需要**通知**多个订阅者关于状态变化（listeners）
- 它需要**接收**用户在执行过程中发来的消息（queues）
- 它需要**能被中断**（abort）
- 它需要**防止**同时运行两次（mutual exclusion）

这些都是有状态的需求。如果把它们塞进循环引擎，循环就不再是纯函数了 — 它会变成一个"知道太多"的上帝对象。

pi 的解决方案是在循环引擎之上套一个有状态的壳：`Agent` 类。循环引擎负责"转"，`Agent` 负责"管"。

```mermaid
graph TB
    subgraph Agent["Agent（有状态壳）"]
        State["MutableAgentState\ntranscript, model, tools"]
        Listeners["listeners: Set<fn>"]
        SQ["steeringQueue"]
        FQ["followUpQueue"]
        AC["activeRun\nAbortController"]
    end
    
    subgraph Loop["agentLoop（无状态引擎）"]
        RunLoop["runLoop()"]
        Stream["streamAssistantResponse()"]
        Tools["executeToolCalls()"]
    end
    
    Agent -->|"createContextSnapshot()\ncreateLoopConfig()"| Loop
    Loop -->|"emit(event)"| Agent
    Agent -->|"processEvents()\n状态归约"| State
    
    style Agent fill:#e3f2fd
    style Loop fill:#fff3e0
```

`Agent` 向循环引擎提供两样东西：一个 context 快照（`createContextSnapshot`）和一个配置对象（`createLoopConfig`）。循环引擎通过事件回调（`emit`）把产出送回 `Agent`，`Agent` 在 `processEvents()` 中做状态归约。

## `Agent` 拥有什么

让我们逐一看 `Agent` 管理的五类状态。

### 1. MutableAgentState — 受控的可变状态

`Agent` 的核心状态是一个 `MutableAgentState` 对象：

```typescript
// packages/agent/src/agent.ts:57-91（简化）

type MutableAgentState = {
  systemPrompt: string;
  model: Model<any>;
  thinkingLevel: ThinkingLevel;
  // 外部看 readonly，赋值时自动 copy
  get tools(): AgentTool[];
  set tools(next: AgentTool[]);
  get messages(): AgentMessage[];
  set messages(next: AgentMessage[]);
  // 运行时状态
  isStreaming: boolean;
  streamingMessage?: AgentMessage;
  pendingToolCalls: Set<string>;
  errorMessage?: string;
};
```

这里有一个精巧的设计：`tools` 和 `messages` 使用 getter/setter 属性。当你赋值 `state.messages = newArray` 时，setter 会自动调用 `newArray.slice()` — 它总是存储一个副本。

```typescript
// packages/agent/src/agent.ts:80-85

get messages() {
  return messages;
},
set messages(nextMessages: AgentMessage[]) {
  messages = nextMessages.slice();  // ← 总是 copy
},
```

为什么要这样做？因为 `AgentMessage[]` 会被传递给循环引擎（`createContextSnapshot`）。如果不 copy，循环引擎修改数组时会直接影响 `Agent` 的状态，两者的状态就耦合了。copy-on-assign 保证了 `Agent` 的状态和循环引擎的工作数据是独立的。

同时，`isStreaming`、`streamingMessage`、`pendingToolCalls`、`errorMessage` 这四个字段在公开的 `AgentState` 接口中是 `readonly` 的：

```typescript
// packages/agent/src/types.ts:253-278

interface AgentState {
  // ... 可读写字段 ...
  readonly isStreaming: boolean;
  readonly streamingMessage?: AgentMessage;
  readonly pendingToolCalls: ReadonlySet<string>;
  readonly errorMessage?: string;
}
```

外部代码（UI 组件、extension）通过 `agent.state` 读取这些字段，但不能直接修改它们。只有 `Agent` 内部的 `processEvents()` 可以修改。这保证了运行时状态的单一真相源。

### 2. 事件订阅 — 有序且受信号保护

```typescript
// packages/agent/src/agent.ts:159-219

private readonly listeners = new Set<
  (event: AgentEvent, signal: AbortSignal) => Promise<void> | void
>();

subscribe(listener): () => void {
  this.listeners.add(listener);
  return () => this.listeners.delete(listener);
}
```

订阅模式的三个设计选择：

**1. listener 接收 AbortSignal**。每个 listener 都能感知当前 run 的中止信号。如果一个 listener 在处理事件时发现 `signal.aborted`，它可以选择跳过耗时操作（比如持久化）。

**2. listener 的 Promise 被 await**。`processEvents` 中的代码是：

```typescript
for (const listener of this.listeners) {
  await listener(event, signal);
}
```

这意味着 listener 按注册顺序串行执行。一个慢的 listener 会阻塞后续 listener。这是故意的 — 它保证了状态归约和事件通知的顺序一致性。如果 listener 并行执行，两个 listener 可能同时读到不一致的中间状态。

**3. `agent_end` 不等于 idle**。`agent_end` 事件只意味着循环引擎不再发射事件了。但 `Agent` 要等到**所有 listener 处理完 `agent_end`** 后才算真正 idle。这就是 `waitForIdle()` 和 `agent_end` 的区别：

```typescript
// packages/agent/src/agent.ts:293-295

waitForIdle(): Promise<void> {
  return this.activeRun?.promise ?? Promise.resolve();
}
```

`activeRun.promise` 在 `finishRun()` 中 resolve，而 `finishRun()` 在所有 listener 处理完毕之后才被调用。

### 3. 消息队列 — 两种节奏的输入

```typescript
// packages/agent/src/agent.ts:112-143

class PendingMessageQueue {
  private messages: AgentMessage[] = [];

  constructor(public mode: QueueMode) {}

  enqueue(message: AgentMessage): void {
    this.messages.push(message);
  }

  drain(): AgentMessage[] {
    if (this.mode === "all") {
      const drained = this.messages.slice();
      this.messages = [];
      return drained;
    }
    // "one-at-a-time" 模式
    const first = this.messages[0];
    if (!first) return [];
    this.messages = this.messages.slice(1);
    return [first];
  }
}
```

`Agent` 持有两个独立的消息队列：

- `steeringQueue`：`agent.steer(msg)` 入队，在 turn 间隙被消费
- `followUpQueue`：`agent.followUp(msg)` 入队，在 agent 本来要退出时被消费

每个队列有两种 drain 模式：

- `"all"`：一次性取出所有排队的消息
- `"one-at-a-time"`（默认）：一次只取一条，剩余的留到下次

为什么默认是 `one-at-a-time`？考虑这个场景：用户在 agent 执行 bash 命令时快速输入了三条 steering 消息。如果用 `"all"` 模式，三条消息会同时注入 context，LLM 需要一次理解三条指令。如果用 `"one-at-a-time"`，LLM 先处理第一条，在下一个 turn 间隙再收到第二条 — 就像人类对话中逐条回应，而不是一次性面对一堆请求。

队列和循环引擎的对接发生在 `createLoopConfig()` 中：

```typescript
// packages/agent/src/agent.ts:407-431（简化）

private createLoopConfig(): AgentLoopConfig {
  return {
    // ... 其他字段 ...
    getSteeringMessages: async () => {
      return this.steeringQueue.drain();
    },
    getFollowUpMessages: async () => {
      return this.followUpQueue.drain();
    },
  };
}
```

`Agent` 把队列的 `drain()` 方法包装成循环引擎需要的 `getSteeringMessages` 和 `getFollowUpMessages` 回调。循环引擎不知道消息从哪来 — 它只管调用回调取消息。

### 4. 中止控制 — 一个 AbortController 管全局

```typescript
// packages/agent/src/agent.ts:434-457（简化）

private async runWithLifecycle(
  executor: (signal: AbortSignal) => Promise<void>
): Promise<void> {
  const abortController = new AbortController();
  let resolvePromise = () => {};
  const promise = new Promise<void>((resolve) => {
    resolvePromise = resolve;
  });
  this.activeRun = { promise, resolve: resolvePromise, abortController };

  this._state.isStreaming = true;
  try {
    await executor(abortController.signal);
  } catch (error) {
    // 安全网：即使循环违反了"must not throw"契约，
    // Agent 也能合成一条失败消息而不是崩溃
    await this.handleRunFailure(
      error, abortController.signal.aborted
    );
  } finally {
    this.finishRun();
  }
}
```

`Agent` 还提供了 `continue()` 方法，它调用循环引擎的 `agentLoopContinue` — 从当前 transcript 继续，而不添加新的 prompt。当最后一条消息是 assistant 角色时，`continue()` 会先尝试排空 steering 队列或 follow-up 队列作为新的 prompt。这是 Agent 和循环引擎之间的一个精巧协调。

每次 `prompt()` 或 `continue()` 调用都会创建一个新的 `AbortController`。它的 signal 被传递给循环引擎、所有 listener、所有工具执行。当用户调用 `agent.abort()` 时：

```typescript
abort(): void {
  this.activeRun?.abortController.abort();
}
```

一个 `abort()` 调用就能中止整条链：LLM 流式响应被取消 → 工具执行被中止 → 循环退出。

### 5. 互斥锁 — 禁止重入

```typescript
async prompt(input): Promise<void> {
  if (this.activeRun) {
    throw new Error(
      "Agent is already processing a prompt. " +
      "Use steer() or followUp() to queue messages, " +
      "or wait for completion."
    );
  }
  // ...
}
```

`Agent` 通过检查 `activeRun` 来防止同时运行两个循环。这不是用 Mutex 实现的，而是一个简单的存在性检查 — 如果 `activeRun` 存在，说明有循环在跑，新的 `prompt()` 调用会抛异常。

注意错误信息的设计：它不只是说"不行"，还告诉调用者**应该怎么做** — "Use steer() or followUp() to queue messages, or wait for completion." 错误信息本身就是 API 文档。

## `processEvents`：状态归约器

`Agent` 接收循环引擎发射的事件，并在 `processEvents()` 中做状态归约。这个方法的逻辑类似 Redux 的 reducer — 给定当前状态和一个事件，更新状态 — 但不同于 Redux 的纯函数语义，这里是直接 mutation：

```typescript
// packages/agent/src/agent.ts:491-538（简化，省略了 signal 空值保护）

private async processEvents(event: AgentEvent): Promise<void> {
  switch (event.type) {
    case "message_start":
      this._state.streamingMessage = event.message;
      break;

    case "message_update":
      this._state.streamingMessage = event.message;
      break;

    case "message_end":
      this._state.streamingMessage = undefined;
      this._state.messages.push(event.message);
      break;

    case "tool_execution_start": {
      const pendingToolCalls = new Set(this._state.pendingToolCalls);
      pendingToolCalls.add(event.toolCallId);
      this._state.pendingToolCalls = pendingToolCalls;
      break;
    }

    case "tool_execution_end": {
      const pendingToolCalls = new Set(this._state.pendingToolCalls);
      pendingToolCalls.delete(event.toolCallId);
      this._state.pendingToolCalls = pendingToolCalls;
      break;
    }

    case "turn_end":
      if (event.message.role === "assistant"
        && event.message.errorMessage) {
        this._state.errorMessage = event.message.errorMessage;
      }
      break;

    case "agent_end":
      this._state.streamingMessage = undefined;
      break;
  }

  // 先归约状态，再通知 listener
  // 实际代码中还有 signal 空值保护：
  // if (!signal) throw new Error("listener invoked outside active run")
  const signal = this.activeRun?.abortController.signal;
  for (const listener of this.listeners) {
    await listener(event, signal);
  }
}
```

几个值得注意的设计细节：

**1. `pendingToolCalls` 每次修改都创建新 Set**。`tool_execution_start` 和 `tool_execution_end` 不是在原 Set 上 add/delete，而是创建一个新的 Set 再赋值。这是因为 `AgentState.pendingToolCalls` 是 `ReadonlySet` — 外部代码持有的引用不会被意外修改。新建 Set 保证了不可变语义。

**2. 状态归约在 listener 通知之前**。`switch` 语句先更新状态，然后才 `for` 循环通知 listener。这意味着 listener 在收到 `message_end` 事件时，`state.messages` 已经包含了这条消息，`state.streamingMessage` 已经被清空。listener 总是看到一致的状态。

**3. 不是所有事件都有状态变更**。`agent_start`、`turn_start`、`tool_execution_update` 都没有对应的状态修改 — 它们只被透传给 listener。归约器只处理真正影响状态的事件。

## `CustomAgentMessages`：类型安全的扩展点

`Agent` 管理的 `messages` 数组的类型是 `AgentMessage[]`。这个类型的定义隐藏了一个精妙的扩展机制：

```typescript
// packages/agent/src/types.ts:236-245

// 默认为空 — 应用通过声明合并扩展
export interface CustomAgentMessages {
  // Empty by default
}

// AgentMessage = LLM 消息 + 所有自定义消息
export type AgentMessage =
  Message | CustomAgentMessages[keyof CustomAgentMessages];
```

`CustomAgentMessages` 是一个空接口，但它使用了 TypeScript 的声明合并（declaration merging）。应用层可以这样扩展它：

```typescript
// 在 pi-coding-agent 中
declare module "@earendil-works/pi-agent-core" {
  interface CustomAgentMessages {
    custom: CustomMessage;        // compaction 摘要、分支标记等
    bashExecution: BashMessage;   // bash 工具的结构化结果
  }
}
```

扩展之后，`AgentMessage` 自动变成：

```typescript
type AgentMessage =
  | Message          // user, assistant, toolResult
  | CustomMessage    // compaction, branch, notification
  | BashMessage;     // bash 结构化结果
```

**为什么不用普通的联合类型？**

如果把自定义消息硬编码到联合类型里，pi-agent-core 就需要知道 pi-coding-agent 的消息类型 — 依赖方向反了。声明合并让 pi-agent-core 定义框架（空的 `CustomAgentMessages`），pi-coding-agent 填充内容，依赖方向保持正确。

**为什么不用 `any` 或泛型？**

用 `any` 会丢失类型安全。用泛型 `Agent<TMessage>` 会让每个使用 Agent 的地方都要传类型参数。声明合并在全局生效，不需要传递类型参数，所有使用 `AgentMessage` 的地方自动包含自定义类型。

这和 `convertToLlm` 回调配合形成完整的设计：自定义消息在循环内部是一等公民（类型安全、可以被 `transformContext` 处理），在出门见 LLM 时被 `convertToLlm` 过滤掉。类型系统保证你不会忘记处理某种自定义消息。

## Agent 不管什么

理解 `Agent` 的边界和理解它的能力同样重要。以下是 `Agent` 明确不管的事情：

| 关注点 | Agent 的态度 | 谁管 |
|--------|-------------|------|
| 会话持久化 | 不知道"会话"的存在 | SessionManager（第 11 章） |
| UI 渲染 | 只发射事件，不管谁听 | TUI / Web UI（第 24 章） |
| 认证 | 通过 `getApiKey` 回调获取，不管 token 怎么来的 | OAuth 模块（第 7 章） |
| 模型选择 | 只持有一个 `model` 字段，不管怎么选的 | ModelRegistry（第 18 章） |
| Context 压缩 | 通过 `transformContext` 委托，不管怎么压 | Compaction（第 12 章） |
| System prompt 拼接 | 只持有一个 `systemPrompt` 字符串 | system-prompt.ts（第 14 章） |
| 工具注册 | 只持有 `tools[]`，不管工具从哪来 | Extension（第 15 章） |

这张表揭示了一个设计原则：**Agent 只管运行时状态，不管配置和策略。** 它知道自己正在用什么模型（`state.model`），但不知道为什么选这个模型。它知道有哪些工具可用（`state.tools`），但不知道这些工具是怎么被发现和注册的。它知道 system prompt 是什么（`state.systemPrompt`），但不知道 prompt 是怎么从多个来源拼接出来的。

## 另一条路：`AgentHarness` 这个"厚壳"

上面那张表的每一行 —— 会话持久化、压缩、skills、工具来源 —— `Agent` 都明确委托给了上层（coding-agent 包）。但 packages/agent 在 v0.79 周期里长出了**第二条路径**：`AgentHarness`（`packages/agent/src/harness/agent-harness.ts:174`），它把那张表里委托出去的东西**反过来收编回了 agent 包内部**。

如果说 `Agent` 是"薄壳 + 上层自己拼装一切"，`AgentHarness` 就是一个"自带电池"的厚壳：

| 表里委托出去的 | `AgentHarness` 的内聚实现 |
|---|---|
| 会话持久化（SessionManager） | 两层拆分：底层 `SessionStorage`（条目/树持久化，`harness/types.ts:498`）+ 上层 `SessionRepo`（会话集合生命周期，`:528`）；实现 `JsonlSessionStorage` / `InMemorySessionStorage`、`JsonlSessionRepo` / `InMemorySessionRepo`（`harness/session/`）、session tree、`fork()`、`navigateTree()` |
| Context 压缩（Compaction） | 内置 `compact()` / `prepareCompaction` + 分支摘要（`harness/compaction/`），可配 `retry?: RetryPolicy` 与 retry_* 事件 |
| Skills / Prompt 模板 | `skill(name)` / `promptFromTemplate(name, args)`（`harness/skills.ts`、`prompt-templates.ts`） |
| 工具来源（Extension） | `getTools`/`setTools` 与 `getActiveTools`/`setActiveTools` 二层模型 + 重名校验；工具类型改为上下文感知的 `AgentHarnessTool<TContext>` |
| 文件系统 / shell 触达 | `ExecutionEnv`（`FileSystem` + `Shell`，`harness/types.ts:373`）被包装为 `ExecutionToolContext`（`harness/tools/tool-context.ts:4`），喂给内置工具 `harness/tools/{read,write,edit,bash}.ts`；Node 实现见 `harness/env/nodejs.ts` |

它还提供了比 `Agent.subscribe()` 更丰富的**类型化钩子链** `on(type, handler)`（`agent-harness.ts:1050`）—— `before_provider_request`、`after_provider_response`、`tool_call`/`tool_result`、`session_before_compact` / `session_compact`、`session_before_tree` / `session_tree` 等 —— 以及一套 `Result<T, E>` 风格的错误模型（`AgentHarnessError` 归一化 `SessionError`/`CompactionError`/`BranchSummaryError`，`agent-harness.ts:145-151`）和显式的运行阶段机 `AgentHarnessPhase = "idle" | "turn" | "compaction" | "branch_summary" | "retry"`（`types.ts:553`），用 `phase !== "idle"` 守卫拒绝重入。

```mermaid
flowchart LR
    subgraph thin["薄壳路径（生产在用）"]
        Agent["Agent\n只管运行时状态"]
        Upper["coding-agent 上层\nSessionManager/Compaction/\nExtension/Skills"]
        Agent -.委托.-> Upper
    end
    subgraph thick["厚壳路径（演进中）"]
        Harness["AgentHarness\n自带会话/压缩/skills/ExecutionEnv"]
    end
    Loop["agentLoop\n（同一个无状态引擎）"]
    Agent --> Loop
    Harness --> Loop
```

厚壳在 v0.79 之后又经历了几次实质重构，值得单独展开三处，因为它们恰好都是"把厚度收回内核"这个选择必须付的账。

### 工具模型：从上下文无关的 `AgentTool` 到 `AgentHarnessTool<TContext>`

薄壳路径里，工具是**上下文无关**的 —— `AgentTool.execute(id, args, signal, onUpdate)` 拿不到任何应用级依赖，要触达文件系统或 shell 得靠闭包捕获。v0.82.0 的 breaking 重构给厚壳换了一套工具类型 `AgentHarnessTool<TContext>`（`harness/types.ts:99-112`），它在 `AgentTool` 的 `execute` 末尾**多加了一个 `context: TContext` 参数**：

```typescript
// packages/agent/src/harness/types.ts:99-112（简化）
export type AgentHarnessTool<TContext extends object | undefined, ...> =
  Omit<AgentTool<...>, "execute"> & {
    execute(
      toolCallId: string,
      params: Static<TParameters>,
      signal: AbortSignal | undefined,
      onUpdate: AgentToolUpdateCallback<TDetails> | undefined,
      context: TContext,          // ← 每个 turn 快照解析出的应用自定义 context
    ): Promise<AgentToolResult<TDetails>>;
  };
```

这个 `context` 从哪来？厚壳的构造选项里多了一个 `toolContext`（`harness/types.ts:955`）：应用要么给一个静态值，要么给一个零参 provider，由 harness **在每个 turn 的快照时刻解析一次**，再注入当轮所有工具的 `execute`。类型上还做了纪律约束 —— 当 `TContext` 为 `undefined`（上下文无关的 harness）时 `toolContext?: undefined`，一旦声明了非空 `TContext`，`toolContext` 就变成**必填**（`harness/types.ts:942-956`）。

那原来那套 `ExecutionEnv`（`FileSystem` + `Shell`）去哪了？它**没有被删**（`harness/types.ts:373` 仍在），而是被**降级成一种具体的 context 形态**：内置工具需要的文件/ shell 触达被收进 `ExecutionToolContext`（`harness/tools/tool-context.ts:4`），其定义就是薄薄一层 `{ env: ExecutionEnv }`。厚壳自带的上下文感知工具 `harness/tools/{read,write,edit,bash}.ts` 正是以这个 `ExecutionToolContext` 为 `TContext`，通过 `execute(..., context)` 的最后一个参数拿到 `env` 去读写磁盘、跑命令。

**这个改动的取舍**：得到的是**工具依赖显式化** —— 工具不再靠闭包偷偷捕获环境，而是由 harness 在 turn 边界统一注入、可随快照替换（比如切换工作目录 / 沙箱）；付出的是**又一个 breaking 的类型分叉**，厚壳的工具从此和薄壳的 `AgentTool` 不是同一个类型，两条路径的工具不能直接互换。

### 会话两层：`SessionStorage` 与 `SessionRepo`

前面表里把会话持久化一句带过成"`SessionRepo` 抽象"。厚壳的会话其实是**两层**，把"一个会话内部怎么存"和"多个会话怎么管"拆开了。这个两层结构在 v0.80.x 就已成型（v0.80.4 起导出 `Jsonl` / `InMemory` 两套实现），v0.81.0 则重构了它的契约（新增 `getPathToRootOrCompaction`、cursor 式读取，以及 compaction checkpoint 语义）：

- **`SessionStorage`**（`harness/types.ts:498`）是**单会话内的条目/树存储**：`appendEntry` / `getEntry` / 游标式 `getEntries`、`getLeafId` / `setLeafId` 追踪当前叶子、`getSessionName` / `getSessionStats`，以及关键的 `getPathToRootOrCompaction(leafId)`（`:512`）—— 从某个叶子回溯到根**或到最近一次 compaction 为止**，让压缩尾部成为一个自包含的 checkpoint，回放不必每次都拉到最开头。
- **`SessionRepo`**（`harness/types.ts:528`）是**跨会话的集合生命周期**：`create` / `open` / `list` / `delete` / `fork`。`fork` 就落在这一层，配合 storage 的 tree 能力实现分支。

两层各有内存与 JSONL 两套实现：`InMemorySessionStorage` / `JsonlSessionStorage`（v0.80.4 起导出）与 `InMemorySessionRepo` / `JsonlSessionRepo`；JSONL 侧的仓库接口收窄为 `JsonlSessionRepoApi`（`harness/types.ts:550`）。分层的意义在于：换存储介质只需替换 `SessionStorage`（比如未来接数据库），而 `SessionRepo` 的会话管理语义与 `fork` / 树导航逻辑不必跟着改。

### 压缩重试：`RetryPolicy` 与三个 retry_* 事件

前面提到厚壳的运行阶段机里有一个 `"retry"` 阶段（`types.ts:553`）。v0.81.1 把它补齐成了一套**可配置的重试策略 + 生命周期事件**：构造选项新增 `retry?: RetryPolicy`（`harness/types.ts:934`，类型来自 pi-ai），专门作用于**生成式的 compaction 与 branch-summary 请求**（这两步要额外调 LLM，是最容易踩到瞬时失败的地方）。重试过程通过三个事件对外可观测：

- `retry_scheduled`（`harness/types.ts:666`）：带 `operation`（`"compaction"` | `"branch_summary"`）、`attempt` / `maxAttempts` / `delayMs` / `errorMessage`，宣告"第几次重试将在多久后开始、上次为何失败"。
- `retry_attempt_start`（`:675`）：一次重试真正启动。
- `retry_finished`（`:680`）：重试序列结束。

它们和既有的 `"retry"` 阶段呼应 —— 阶段机用 `phase === "retry"` 守卫重入，事件流则让订阅者看到重试的节奏。这正是薄壳"不能自己重试"（第 8 章取舍）在厚壳里被收编的地方：`Agent` 把重试留给上层，`AgentHarness` 则把它做进了内核，代价是又多背了一份策略配置。要注意，这套 harness 内核里的 `RetryPolicy` / `retry_*` 与第 12 章 coding-agent 产品层的 `summarization_retry_*` 是**不同层的同构机制** —— 前者在 agent 内核重试生成式 compaction / branch-summary，后者在产品层重试会话摘要，二者互不依赖、各自成套。

### 认证收敛：models-only

薄壳 `Agent` 走 `getApiKey` 回调拿密钥（上面那张"Agent 不管什么"的表仍然成立，`agent.ts:102`）。但厚壳这一侧在 v0.80.0 做了 breaking 收敛：**移除了 `getApiKeyAndHeaders`**，认证统一收口到一个必填的 `models: Models`（`harness/types.ts:923`）—— turn 流式、compaction、branch-summary 全部经由 `Models` 集合，认证由各 provider 自己的 auth 解析。`compact()` / `generateSummary()` / `generateBranchSummary()` 也相应改收 `Models`。换句话说，厚壳不再有"裸密钥 + headers"这条老路，只认 provider 运行时对象。**这只影响 `AgentHarness` 一侧；`Agent` 路径的 `getApiKey` 叙述保持不变。**

**必须如实说明它的采用状态**：截至 v0.82.1，**生产的 coding-agent 仍然用 `new Agent({...})`**（`packages/coding-agent/src/core/sdk.ts:294`），`AgentHarness` 目前只在 agent 包自身的测试与 `docs/{agent-harness,durable-harness,hooks}.md` 里使用，coding-agent 源码中没有任何 `AgentHarness` 引用。所以它是一个**演进中的并行抽象**，不是已经取代 `Agent` 的新默认。

这两条路线本身就是本书主线的一次张力实验：`Agent` 把厚度留给上层、换取内核纯净；`AgentHarness` 把厚度收回内核、换取开箱即用的持久会话与分支能力。两者共享同一个 `agentLoop` 引擎 —— 厚薄之争发生在引擎之上，而引擎自己始终只管转。

## 取舍分析

### 得到了什么

**1. 清晰的职责边界**。循环引擎是纯计算，Agent 是状态管理。两者可以独立演进 — 改循环逻辑不影响状态管理，改状态结构不影响循环逻辑。

**2. 可预测的状态变更**。所有状态修改都通过 `processEvents()` 这一个入口。想知道某个状态字段什么时候会变？只需要在 `processEvents()` 的 `switch` 语句中搜索。

**3. 灵活的消费者模型**。`subscribe()` 让任意多个消费者同时观察 Agent 的行为。TUI 订阅事件来渲染，session manager 订阅事件来持久化，extension 订阅事件来做自定义逻辑 — 它们互不干扰。

### 放弃了什么

**1. Agent 是一个胖接口**。`Agent` 类有 30+ 个公开方法和属性（包括 `subscribe`、`prompt`、`continue`、`steer`、`followUp`、`abort`、`waitForIdle`、`reset` 等方法，以及 `convertToLlm`、`transformContext`、`beforeToolCall`、`afterToolCall`、`streamFn`、`sessionId`、`transport`、`toolExecution` 等可配置字段）。如果你只想跑一个简单的 agent 循环，直接调用 `runAgentLoop()` 比创建一个 Agent 实例更直接。Agent 的价值在有状态、有交互的场景，对于一次性脚本反而是负担。

**2. 状态同步依赖事件顺序**。因为 listener 串行执行，一个慢的 listener（比如写磁盘的 session manager）会延迟后续 listener（比如渲染 UI 的 TUI）收到事件的时间。在实践中这通常不是问题（listener 的处理时间远小于 LLM 响应时间），但在极端情况下可能导致 UI 延迟。

**3. Agent 是单线程模型**。同一时间只能有一个 `prompt()` 或 `continue()` 在运行。这意味着不能实现"后台持续运行、前台随时查询"的模式。如果需要这种模式，必须在 Agent 之上再包一层异步调度器。

---

### 版本演化说明
> 本章核心分析基于 pi-mono v0.66.0，并已对照 v0.82.1。`Agent` 类的核心结构自引入
> 以来保持稳定，仍是生产 coding-agent 的运行时壳（`coding-agent/src/core/sdk.ts:294`）。
> `PendingMessageQueue` 的 `"all"` | `"one-at-a-time"` 模式是后来的增强，
> 早期版本只有 `"one-at-a-time"` 行为。`CustomAgentMessages` 声明合并机制
> 在 pi-agent-core 从 pi-coding-agent 分离时引入，解决了包间类型依赖问题
> （模块名随 scope 迁移为 `@earendil-works/pi-agent-core`）。v0.79 周期新增的
> `AgentHarness` 子系统（`packages/agent/src/harness/`）把会话/压缩/skills/ExecutionEnv
> 内聚进 agent 包，是与 `Agent` 并存的"厚壳"路径，目前仍仅在 agent 包自身测试/docs 中使用。
> 该厚壳在 v0.79 之后持续重构：v0.80.0 认证收敛为 models-only（移除 `getApiKeyAndHeaders`，
> `models: Models` 必填）；v0.81.0 会话拆成 `SessionStorage` + `SessionRepo` 两层
> （新增 `getPathToRootOrCompaction`）；v0.81.1 为 compaction / branch-summary 补上
> `retry?: RetryPolicy` 与 `retry_scheduled` / `retry_attempt_start` / `retry_finished` 事件；
> v0.82.0 工具模型 breaking 重构为上下文感知的 `AgentHarnessTool<TContext>` + `toolContext`，
> `ExecutionEnv` 被包装为 `ExecutionToolContext` 喂给内置的 read/write/edit/bash 工具。
