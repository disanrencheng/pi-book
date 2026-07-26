# 第 22 章：`bash` 与外部世界的边界

> **定位**：本章解析 bash 工具的定位 — 万能后备而非首选工具。
> 前置依赖：第 19 章（工具设计原则）。
> 适用场景：当你想理解结构化工具和非结构化工具的关系。

## bash 是后备，不是首选

bash 在 pi 里的定位是**万能后备**：用于没有专用工具覆盖的操作 — 安装依赖、运行测试、启动服务、执行 git 命令。读文件该用 read、编辑用 edit、搜索用 grep、查找文件用 find。

值得注意的是这个"后备"定位**如何被表达**。早期版本里，pi 的 system prompt 直接写着一条"当有专用工具时不要用 bash"的指引；到 v0.77.0，这类"用 prompt 教模型挑工具"的文字被删除了（详见第 14 章）。现在的判断是：工具的取舍应当由"**哪些工具被激活**"来表达 —— 如果一个会话同时激活了 grep/find/ls 和 bash，模型自然会用更顺手、反馈更好的结构化工具；只有在"仅有 bash、没有专用探索工具"时，prompt 才补一句让它用 bash 去做 ls/rg/find。bash 工具自己的 promptSnippet 也回归中性（`"Execute bash commands (ls, grep, find, etc.)"`，bash.ts:280）。

为什么倾向结构化工具不是审美偏好，而是工程约束：结构化工具有参数校验、自动截断、跨平台一致性，bash 没有。LLM 用 bash 搜索文件时可能忘记排除 `node_modules`，可能用了 macOS 特有的 `find` 参数，可能返回几万行输出。结构化工具替它兜住了这些风险。

## Schema 定义：极简但完整

```typescript
// packages/coding-agent/src/core/tools/bash.ts:27-30
const bashSchema = Type.Object({
  command: Type.String({
    description: "Bash command to execute"
  }),
  timeout: Type.Optional(Type.Number({
    description: "Timeout in seconds (optional, no default timeout)"
  })),
});
```

只有两个参数 — command 和 timeout。这和 read 的三参数、grep 的七参数形成了鲜明对比。bash 的 schema 越简单，LLM 的使用门槛越低。但代价是：bash 的一切细节（工作目录、环境变量、错误处理）都被压缩进了一个 command 字符串中。

`timeout` 是可选参数，没有默认值。这是一个有意的设计选择 — 大多数命令（`git status`、`npm install`）的执行时间差异巨大，设置一个统一的默认超时要么太短（中断正常操作）要么太长（无用）。让 LLM 根据命令性质自行决定超时更合理。

## `BashOperations`：可插拔的执行后端

和 read 工具一样，bash 也通过接口抽象了执行环境：

```typescript
// packages/coding-agent/src/core/tools/bash.ts:43-61
export interface BashOperations {
  exec: (
    command: string,
    cwd: string,
    options: {
      onData: (data: Buffer) => void;
      signal?: AbortSignal;
      timeout?: number;
      env?: NodeJS.ProcessEnv;
    },
  ) => Promise<{ exitCode: number | null }>;
}
```

这个接口的设计透露了几个关键决策：

**流式输出**。`onData` 回调接收 `Buffer` 而非等待执行完毕后返回字符串。这让 TUI 可以实时显示命令输出（比如 `npm install` 的进度条），而非等到命令结束才显示结果。

**AbortSignal 支持**。用户可以随时取消正在执行的命令。`signal` 参数传递到执行层，触发进程树的 kill。

**返回值是 exitCode 而非输出内容**。输出通过 `onData` 流式传递，返回值只关心"成功还是失败"。`exitCode` 为 `null` 表示进程被 kill（用户取消或超时）。

## 默认执行后端：本地 Shell

```typescript
// packages/coding-agent/src/core/tools/bash.ts:69-127
export function createLocalBashOperations(): BashOperations {
  return {
    exec: (command, cwd, { onData, signal, timeout, env }) => {
      return new Promise((resolve, reject) => {
        const { shell, args } = getShellConfig();
        if (!existsSync(cwd)) {
          reject(new Error(
            `Working directory does not exist: ${cwd}`
          ));
          return;
        }
        const child = spawn(shell, [...args, command], {
          cwd,
          detached: true,
          env: env ?? getShellEnv(),
          stdio: ["ignore", "pipe", "pipe"],
        });
        // Stream stdout and stderr
        child.stdout?.on("data", onData);
        child.stderr?.on("data", onData);
        // ...timeout and abort handling...
      });
    },
  };
}
```

几个关键实现细节：

**`detached: true`**。创建独立的进程组，这样 `killProcessTree` 可以一次性杀掉主进程和它所有的子进程。没有这个选项，kill 主进程后子进程可能变成孤儿进程继续运行。

**`stdio: ["ignore", "pipe", "pipe"]`**。stdin 被忽略（LLM 不需要和命令交互），stdout 和 stderr 都被 pipe 出来通过 `onData` 回调传递。注意 stdout 和 stderr 合并到了同一个 `onData` — 这意味着输出顺序和终端中看到的一致，但没有办法区分标准输出和错误输出。

**工作目录校验**。在 spawn 之前检查 `cwd` 是否存在。这避免了一个常见的 debug 陷阱 — 如果工作目录不存在，`spawn` 会报一个含糊的错误，不如直接给出明确的错误信息。

## 超时处理

```typescript
// packages/coding-agent/src/core/tools/bash.ts:84-92
let timedOut = false;
let timeoutHandle: NodeJS.Timeout | undefined;
if (timeout !== undefined && timeout > 0) {
  timeoutHandle = setTimeout(() => {
    timedOut = true;
    if (child.pid) killProcessTree(child.pid);
  }, timeout * 1000);
}
```

超时时不是简单地 reject promise — 而是先 kill 进程树，然后在进程退出回调中检查 `timedOut` 标志再 reject。这个顺序很重要：如果先 reject 再 kill，调用方可能在进程还在运行时就开始处理"超时"结果，导致输出数据竞争。

超时 reject 的错误信息格式是 `timeout:${timeout}` — 这个格式化的字符串允许上层精确知道超时值，用于生成更有用的提示信息（"命令在 30 秒后超时，考虑增加超时时间"）。

## `BashToolDetails` 与结构化结果

bash 的执行结果不只是一个字符串。`BashToolDetails` 记录了截断元信息和完整输出文件路径：

```typescript
// packages/coding-agent/src/core/tools/bash.ts:34-37
export interface BashToolDetails {
  truncation?: TruncationResult;
  fullOutputPath?: string;
}
```

`fullOutputPath` 指向一个临时文件，保存了命令的完整输出。当输出被截断时，LLM 可以通过 read 工具去读取这个临时文件获取完整内容。这是一种精巧的工具间协作 — bash 截断输出以保护 context，但提供了一条"逃生通道"让 LLM 在需要时可以看到全部内容。

临时文件路径的生成使用了 crypto random：

```typescript
// packages/coding-agent/src/core/tools/bash.ts:22-25
function getTempFilePath(): string {
  const id = randomBytes(8).toString("hex");
  return join(tmpdir(), `pi-bash-${id}.log`);
}
```

## `BashSpawnHook`：命令执行前的最后一道关卡

```typescript
// packages/coding-agent/src/core/tools/bash.ts:130-136
export interface BashSpawnContext {
  command: string;
  cwd: string;
  env: NodeJS.ProcessEnv;
}

export type BashSpawnHook =
  (context: BashSpawnContext) => BashSpawnContext;
```

`BashSpawnHook` 在命令执行前被调用，可以修改命令、工作目录或环境变量。典型用途：

- **命令审计**：记录所有 LLM 执行的命令
- **命令改写**：在命令前添加 `set -e` 确保脚本在第一个错误时停止
- **环境注入**：添加特定的环境变量（比如 API key）

配合 `commandPrefix` 选项，可以在每条命令前自动添加前缀（比如 `source ~/.nvm/nvm.sh &&`），确保 shell 环境正确初始化。

## 输出截断策略

bash 的输出截断和 read 不同 — 它用的是 `truncateTail` 而非 `truncateHead`。这意味着 bash 保留的是**最后**的输出而非开头。

为什么？因为 bash 命令的关键信息通常在末尾 — 编译错误的最后几行、测试结果的 summary、安装完成的 success 信息。如果一个 `npm install` 输出了几千行依赖安装日志，LLM 需要看到的是最后的 "added 42 packages" 或 "ERR! missing dependency"，而不是开头的 "added foo@1.0.0"。

这和 read 的 `truncateHead`（保留开头）形成了有意的对比：读文件时，开头（函数签名、import 语句、文件头注释）更重要；执行命令时，结尾（结果、错误信息）更重要。

## Sandbox 讨论：bash 的安全边界

bash 是安全风险最大的工具。它可以执行任意命令，包括 `rm -rf /`。pi 通过多个层次来控制这个风险：

**1. `beforeToolCall` 钩子（第 9 章）**。产品层可以通过这个钩子实现任意安全策略 — 命令白名单、确认弹窗、日志审计等。具体实现由上层决定。

**2. 独立的执行器抽象**。mom（Slack bot，第 28 章）通过 `Executor` 接口（`DockerExecutor` / `HostExecutor`）让所有命令在 Docker 容器中执行，限制 agent 的文件系统访问范围。这不是替换 `BashOperations`，而是 mom 的工具层自己封装了执行环境（详见第 28 章）。

**3. 环境隔离**。`BashSpawnHook` 可以过滤环境变量，防止 API key 等敏感信息泄露给 LLM 执行的命令。

这三层保护形成了一个从"提示确认"到"物理隔离"的安全梯度。不同的产品场景可以选择合适的安全级别 — 个人使用可能只需要第一层，企业部署可能需要三层全开。

## TUI 中的实时渲染

bash 命令在 TUI 中有特殊的渲染逻辑。执行期间，TUI 实时显示输出（通过 `onData` 回调驱动 `requestRender`），并在命令旁边显示经过时间。完成后，输出被折叠为最多 5 行的预览：

```typescript
// packages/coding-agent/src/core/tools/bash.ts:152
const BASH_PREVIEW_LINES = 5;
```

折叠状态下，用户可以展开查看完整输出。这个 UX 设计平衡了"不丢失信息"和"不让长输出淹没对话流"两个需求。

## `excludeFromContext`：执行但不进上下文（v0.76.0）

有一类命令用户想**执行并看到输出，但不希望它进入 LLM 的上下文** —— 典型的是 `cat 一个很大的日志`、`env`（含敏感变量）、或纯粹给人看的探查命令。把这些塞进上下文既浪费 token，又可能污染后续推理。v0.76.0 为此加了 `excludeFromContext`。

```typescript
// packages/coding-agent/src/core/agent-session.ts:2595-2598, 2639
async executeBash(
  command: string,
  onChunk?: (chunk: string) => void,
  options?: { excludeFromContext?: boolean; operations?: BashOperations },
): Promise<BashResult> { /* ... */ }

// 结果记录到会话时带上该标记：
recordBashResult(command, result, options): void {
  const bashMessage: BashExecutionMessage = {
    role: "bashExecution", command, output: result.output,
    /* ... */ excludeFromContext: options?.excludeFromContext,
  };
}
```

交互界面上对应的语法是 **`!!` 前缀** —— 用户输入 `!!` 开头的命令时，它照常执行、输出照常显示在 TUI 里，但这条 `bashExecution` 记录被标记为 `excludeFromContext: true`，在装配发给 LLM 的上下文时被跳过。注意它**仍然写入会话文件**（持久化与"是否发给模型"是两个正交维度，呼应第 11 章的会话模型）。RPC 模式下 `bash` 命令也透传这个选项（见第 26 章）。

另外两点与上下文相关的细节：bash 的输出从 v0.73.0 起**增量地进入模型上下文**（流式 chunk 边产生边累积，而非命令结束才一次性塞入），进程退出后还会继续 drain stdout/stderr 以免丢尾部输出；`shellPath` 与工作目录都跟随**会话当前 cwd**（`sessionManager.getCwd()`，agent-session.ts:2610），而非 pi 启动时的目录 —— 又一次"去 process-global"。

## 会话环境变量：把会话身份注入子进程（v0.82.0）

bash 执行的命令常常需要知道"我正跑在哪个会话里、用的什么模型"。v0.82.0 起，pi 在 spawn 子进程前会往它的环境里注入一组 **`PI_*` 会话变量**：

```typescript
// packages/coding-agent/src/core/tools/bash.ts:166-180（节选）
const env = { ...getShellEnv() };
delete env.PI_SESSION_ID;   // 先清掉继承来的旧值，避免串味
delete env.PI_SESSION_FILE;
delete env.PI_PROVIDER;
delete env.PI_MODEL;
delete env.PI_REASONING_LEVEL;
if (exposeSessionEnvironment && ctx) {
  env.PI_SESSION_ID = ctx.sessionManager.getSessionId();
  const sessionFile = ctx.sessionManager.getSessionFile();
  if (sessionFile) env.PI_SESSION_FILE = sessionFile;
  if (ctx.model) {
    env.PI_PROVIDER = ctx.model.provider;
    env.PI_MODEL = ctx.model.id;
  }
  if (ctx.thinkingLevel) env.PI_REASONING_LEVEL = ctx.thinkingLevel;
}
```

五个变量各自对应一维会话身份：

- **`PI_SESSION_ID`** —— 当前会话 id（第 11 章的 UUIDv7）
- **`PI_SESSION_FILE`** —— 会话 JSONL 文件的绝对路径（若会话落盘）
- **`PI_PROVIDER`** —— 当前模型的 provider（如 `anthropic`）
- **`PI_MODEL`** —— 当前模型 id（如 `claude-opus-4-6`）
- **`PI_REASONING_LEVEL`** —— 当前 thinking 档位（如 `medium`）

有两个设计细节值得注意。其一，注入前**先 `delete` 一遍**：pi 自己可能就跑在一个带 `PI_*` 的环境里（比如嵌套调用），不清掉旧值，子进程会读到上一层的会话身份。其二，注入受 `exposeSessionEnvironment` 开关控制 —— 它默认开启，但宿主可以关掉，因为把会话文件路径暴露给任意 LLM 执行的命令并非所有场景都想要。

一个真实用例是第 26b 章的 **evals harness**：它用 `PI_PROVIDER` / `PI_MODEL` 从环境里读出要评测的模型（`getRequiredModelSelection()`），再装配 session。也就是说，这组会话变量不只是给用户脚本看的 —— 它已经是 pi 自己评测管线的输入。配套地，直连 RPC 的 bash 执行会通过流式 `bash_execution_update` 事件把输出增量吐给客户端（见第 26 章）。

## 取舍分析

### 得到了什么

**灵活性兜底**。当专用工具无法覆盖的场景出现时（比如一个特殊的 CLI 工具），bash 保证 agent 不会"束手无策"。

**可适配的安全模型**。从本地执行到 Docker sandbox，`BashOperations` 接口让安全边界可以按需收紧。

### 放弃了什么

**bash 是安全风险最大的工具**。可插拔架构只是提供了控制点，实际的安全策略需要产品层去实现。一个忘记配置 sandbox 的部署环境，bash 就是一个完全开放的后门。

**输出解析不可靠**。bash 返回的是纯文本，没有结构化信息。LLM 需要自己解析命令输出来判断成功还是失败（虽然 `exitCode` 提供了基本信号）。这是结构化工具（read、grep）相对 bash 的核心优势。

---

### 版本演化说明
> 本章核心分析基于 pi-mono v0.66.0，已对照 **v0.82.1**。Bash 工具的输出处理经历多次改进 —
> 输出 tail 截断、`BashSpawnHook`、`fullOutputPath` 临时文件机制均保持。
> 主要设计级变化：① 会话环境变量 `PI_SESSION_ID`/`PI_SESSION_FILE`/`PI_PROVIDER`/`PI_MODEL`/`PI_REASONING_LEVEL` 注入子进程（v0.82.0，`bash.ts:166-180`，evals harness 依赖 `PI_PROVIDER`/`PI_MODEL`），直连 RPC bash 流式 `bash_execution_update`（第 26 章）；
> ② `excludeFromContext`（`!!` 前缀，v0.76.0）—— 命令执行并显示但输出不入 LLM 上下文；
> ③ 输出从 v0.73.0 起增量进入上下文，进程退出后继续 drain（v0.79.4）；
> ④ `shellPath`/cwd 跟随会话当前 cwd 而非启动 cwd（v0.68.0）。
> 此外 system prompt 不再硬性指引"有专用工具时勿用 bash"（v0.77.0，详见第 14 章）。
