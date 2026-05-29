# Claude Code 查询引擎：AsyncGenerator 事件流全解

> 本文档完整解析 Claude Code 的核心数据流——从用户按下 Enter 到终端显示回复，数据如何通过 4 层 AsyncGenerator 管道流动，工具如何并发执行，以及每个 yield 点的含义。

---

## 目录

1. [核心概念](#核心概念)
2. [管道架构总览](#管道架构总览)
3. [第 1 层：claude.ts — API 通信](#第-1-层claudets--api-通信)
4. [第 2 层：query.ts — Agentic 循环](#第-2-层queryts--agentic-循环)
5. [第 3 层：QueryEngine — 会话管理](#第-3-层queryengine--会话管理)
6. [工具并发执行系统](#工具并发执行系统)
7. [完整时间线示例](#完整时间线示例)
8. [速查表](#速查表)

---

## 核心概念

### AsyncGenerator 的拉取模型

**yield 不是"发出去"，而是"暂停自己，把值交出去，等对方叫我继续"。**

```javascript
async function* producer() {
  console.log("1")
  yield "a"        // 停在这里，等 .next()
  console.log("2") // 只有 .next() 被调用后才执行
  yield "b"        // 又停在这里
  console.log("3")
}
```

执行过程：

```
producer 内部:             .next() 调用:
──────────────             ─────────────
打印 "1"                   
return {value:"a"} ←────── 第 1 次 .next()
⏸️ 冻结                    消费者拿到 "a"，处理中...
                           处理完毕
打印 "2"           ←────── 第 2 次 .next()  
return {value:"b"}
⏸️ 冻结                    消费者拿到 "b"，处理中...
                           处理完毕
打印 "3"           ←────── 第 3 次 .next()
return {done:true}
```

编译器把 generator 函数转换成**状态机**——yield 就是 return + 记住位置，函数体根本没有"在运行"。

### for await 的等价展开

```javascript
// 这段代码
for await (const val of producer()) {
  doSomething(val)
}

// 等价于
const it = producer()
let result = await it.next()     // 拿到 "a"
doSomething(result.value)
result = await it.next()         // 这时 producer 才从 yield "a" 恢复
doSomething(result.value)
result = await it.next()         // done: true
```

### Pull vs Push 模型

```javascript
// ═══ Push 模型（EventEmitter）═══
// 生产者不管消费者处不处理得过来，一直推
emitter.on('message', (msg) => {
  // 如果这里处理慢，下一条照样来，可能堆积
})

// ═══ Pull 模型（AsyncGenerator）═══
// 消费者主动拉，生产者只在被拉时才继续
for await (const msg of generator) {
  // 处理完这条，才会拉下一条
  // 生产者在此期间是暂停的
}
```

### 背压（Backpressure）

```
API SSE 流 → callModel yield → query yield → QueryEngine yield → 消费者
                                                                    │
                                                              阻塞（处理中）
                                                                    │
所有上游都暂停 ←────────────────────────────────────────────────────┘
```

TCP 层面：如果消费者一直不拉取，Node.js 的 stream 背压机制让 TCP 窗口缩小，API 服务器也会降速发送。整条管道自动限速。

### Promise vs Generator 的执行语义

这两个概念是理解工具并发执行的关键：

```javascript
// ═══ Promise（async function）：创建即执行 ═══
async function work() {
  console.log("步骤1")
  await sleep(1000)
  console.log("步骤2")
}

const promise = work()  // ← 不 await，但内部立刻开始执行！
console.log("外面继续")

// 输出：
// "步骤1"       ← 立刻（work 创建时就开始跑了）
// "外面继续"    ← 立刻（不等 work 完成）
// "步骤2"       ← 1 秒后（work 内部继续执行）

// ═══ Generator（async function*）：创建不执行，必须拉取 ═══
async function* runToolUse() {
  console.log("A: 检查权限")
  const permission = await canUseTool()
  console.log("B: 执行工具")
  const result = await tool.call()
  yield { message: result }
}

const generator = runToolUse()
// 输出：（什么都没有！Generator 内部一行代码都没执行）

for await (const update of generator) {
  // 现在才开始执行：A → B → yield
}
```

**核心区别**：
- `async function`：**创建即执行**，不 await = 不等它完成，它自己在后台跑
- `async function*`：**创建不执行**（惰性），必须 `for await` / `.next()` 才开始跑

---

## 管道架构总览

### 四层流水线

Claude Code 不是一个巨大函数，而是**一条 Unix 管道**——每层只做一件事：

```
claude.ts               → 只管和 API 通信（HTTP/SSE 收发）
query.ts                → 只管 agentic 循环（要不要继续调 API）
QueryEngine             → 只管会话状态（token 计数、持久化、限制检查）
streamingToolExecutor   → 只管工具执行（并行调度、权限、超时）
```

### 数据流全景

```
Anthropic API
  │
  │  原始 SSE bytes: event:content_block_delta\ndata:{"delta":{"text":"我"}}
  ▼
┌─────────────────────────────────────────────────────────────────┐
│ claude.ts                                                        │
│ 加工: 把碎片 SSE 拼成完整的结构化 message                         │
│ 输出: { type:'assistant', message:{ content:[完整block] } }       │
└───────────────────────────────┬─────────────────────────────────┘
                                │ yield assistant message
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ query.ts                                                         │
│ 加工: 执行工具 + 决定是否继续循环                                 │
│ 新增: user messages (tool_results from streamingToolExecutor)     │
│ 输出: assistant + user messages 交替                              │
│                                                                  │
│  ┌──────────────────────────────────────────┐                    │
│  │ streamingToolExecutor (内嵌的并行执行器)    │                    │
│  │ 输入: tool_use block                      │                    │
│  │ 输出: tool_result message                 │                    │
│  └──────────────────────────────────────────┘                    │
└───────────────────────────────┬─────────────────────────────────┘
                                │ yield assistant + user messages
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ QueryEngine                                                      │
│ 加工: 追踪状态 + 限制检查 + 格式化                               │
│ 新增: result message (cost, usage, stop_reason)                  │
│ 副作用: 持久化 transcript、累计 token、检查预算                    │
│ 输出: SDKMessage (标准化格式)                                     │
└───────────────────────────────┬─────────────────────────────────┘
                                │ yield SDKMessage
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ 消费者 (REPL / SDK / Remote Control)                             │
│ REPL: 更新 React 状态 → Ink 渲染 → 终端输出                      │
│ SDK:  JSON.stringify → stdout                                    │
│ RC:   WebSocket push → 客户端                                    │
└─────────────────────────────────────────────────────────────────┘
```

### 消费路径分支

```
query.ts yield →  QueryEngine.submitMessage() yield →  消费方处理
                                                       │
                                   ┌───────────────────┼───────────────────┐
                                   ▼                   ▼                   ▼
                             REPL 交互模式         SDK/headless(-p)     Remote Control
                             (更新 React UI)      (输出 JSON 到 stdout)  (推送到客户端)
```

### 为什么不能合并

| 假设 | 后果 |
|------|------|
| claude.ts + query.ts 合并 | API 解析和循环决策混在一起，无法换 provider，无法测试 |
| QueryEngine + query.ts 合并 | SDK 和 REPL 两种模式满是 `if (isSDK) else` 分支 |
| 没有 streamingToolExecutor | 工具只能串行执行，且并发逻辑混入循环主体 |

设计思想跟 Unix 管道 `curl | jq | grep | less` 一致——每个环节只关心输入输出格式。

---

## 第 1 层：claude.ts — API 通信

### Anthropic SSE 事件类型

API 使用 Server-Sent Events 流式返回，按时间顺序有 6 种事件：

```
API 请求发出
    ↓
┌─ message_start          ← 整个响应开始
│
├─ content_block_start    ← 第 1 个内容块开始 (可能是 thinking)
├─ content_block_delta    ← 内容片段（重复多次，每次几个 token）
├─ content_block_delta
├─ content_block_stop     ← 第 1 个内容块结束
│
├─ content_block_start    ← 第 2 个内容块开始 (可能是 text)
├─ content_block_delta
├─ content_block_stop     ← 第 2 个内容块结束
│
├─ content_block_start    ← 第 3 个内容块开始 (可能是 tool_use)
├─ content_block_delta    ← 工具参数 JSON 片段
├─ content_block_stop     ← 第 3 个内容块结束
│
├─ message_delta          ← 最终元数据（stop_reason, usage）
└─ message_stop           ← 整个响应结束
```

| type | 含义 | 包含什么 |
|------|------|---------|
| `message_start` | 响应开始 | `message: { id, model, usage: { input_tokens } }` |
| `content_block_start` | 内容块开始 | `index`, `content_block: { type: "text" / "tool_use" / "thinking" }` |
| `content_block_delta` | 增量片段 | `index`, `delta: { type: "text_delta" / "input_json_delta" / "thinking_delta" }` |
| `content_block_stop` | 内容块结束 | `index` |
| `message_delta` | 响应级元数据 | `delta: { stop_reason }`, `usage: { output_tokens }` |
| `message_stop` | 响应结束 | （空） |

#### content_block 的类型

| content_block.type | 含义 | delta.type |
|---|---|---|
| `text` | 文字回复 | `text_delta` (包含 `.text`) |
| `thinking` | 思考过程（extended thinking） | `thinking_delta` + `signature_delta` |
| `tool_use` | 工具调用 | `input_json_delta` (包含 `.partial_json`) |
| `server_tool_use` | 服务端工具（如 advisor） | `input_json_delta` |

### 原始 SSE 流示例

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_01X","model":"claude-sonnet-4-20250514","usage":{"input_tokens":4200}}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"我来"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"看一下"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":" auth.ts 的内容。"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: content_block_start
data: {"type":"content_block_start","index":1,"content_block":{"type":"tool_use","id":"toolu_01ABC","name":"Read","input":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":"{\"file_"}}

event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":"path\":\"/project/auth.ts\"}"}}

event: content_block_stop
data: {"type":"content_block_stop","index":1}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"tool_use"},"usage":{"output_tokens":85}}

event: message_stop
data: {"type":"message_stop"}
```

### claude.ts 的处理逻辑

```javascript
// services/api/claude.ts 核心 switch
for await (const part of stream) {
  switch (part.type) {
    case 'message_start':
      // 记录 input_tokens
      usage = updateUsage(usage, part.message.usage)
      break

    case 'content_block_start':
      // 根据 type 初始化累加器
      if (part.content_block.type === 'text')
        contentBlocks[part.index] = { type: 'text', text: '' }
      if (part.content_block.type === 'tool_use')
        contentBlocks[part.index] = { ...part.content_block, input: '' }
      if (part.content_block.type === 'thinking')
        contentBlocks[part.index] = { type: 'thinking', thinking: '' }
      break

    case 'content_block_delta':
      // 逐片段拼接
      if (delta.type === 'text_delta')
        contentBlocks[index].text += delta.text           // "我来" + "看一下" + ...
      if (delta.type === 'input_json_delta')
        contentBlocks[index].input += delta.partial_json  // {"file_ + path":"/project/auth.ts"}
      if (delta.type === 'thinking_delta')
        contentBlocks[index].thinking += delta.thinking
      break

    case 'content_block_stop':
      // 一个块完成 → yield 完整 message（关键 yield 点）
      yield { type: 'assistant', message: { content: [contentBlocks[index]] } }
      break

    case 'message_delta':
      // stop_reason + output_tokens
      stopReason = part.delta.stop_reason  // "end_turn" 或 "tool_use"
      usage = updateUsage(usage, part.usage)
      break

    case 'message_stop':
      break
  }
}
```

**关键设计**：`content_block_delta` 时只拼接不 yield，等 `content_block_stop` 时才 yield 完整的 message。这保证消费者收到的永远是完整的结构化数据。

---

## 第 2 层：query.ts — Agentic 循环

### 核心结构

```javascript
async function* queryLoop(params) {
  while (true) {
    // 1. 准备消息
    messagesForQuery = getMessagesAfterCompactBoundary(messages)
    microcompact(messagesForQuery)   // 压缩旧消息中过长的工具结果
    addCacheBreakpoints()            // 在消息末尾加 cache_control 标记

    yield { type: 'stream_request_start' }  // 内部信号

    // 2. 调 API（消费 claude.ts 的 yield）
    for await (const msg of callModel({...})) {
      yield msg                                 // 透传 stream_event 和 assistant
      if (msg 包含 tool_use) {
        streamingToolExecutor.addTool(block)    // 启动工具（不等完成）
      }
      // 非阻塞轮询已完成的工具结果
      for (const result of streamingToolExecutor.getCompletedResults()) {
        yield result.message
      }
    }

    // 3. API 流结束，等待剩余工具完成
    for await (const update of streamingToolExecutor.getRemainingResults()) {
      yield update.message
    }

    // 4. 决策：要不要继续循环
    if (stop_reason === 'end_turn') break       // 模型说完了
    if (stop_reason === 'tool_use') continue    // 带着工具结果再调 API
    if (context太长) compact后continue          // 压缩后重试
  }
}
```

### 关键设计：流式工具执行

工具不等 API 流结束就开始执行——API 返回 tool_use block 的**瞬间**就启动：

```
时间    API 流                  工具执行
────    ──────                  ────────
250ms   yield tool_use(Read)    → addTool → 开始读文件
300ms   yield tool_use(Bash)    → addTool → 开始跑命令
350ms   (API还在生成text...)     Read 完成! → getCompletedResults yield
420ms   message_stop             Bash 还在跑...
        ↓
        进入 getRemainingResults → 等 Bash 完成
3000ms                           Bash 完成! → yield 结果
```

---

## 第 3 层：QueryEngine — 会话管理

### submitMessage() 的结构

```javascript
async *submitMessage(prompt) {
  // 准备：构建消息、检查状态
  for await (const message of query({...})) {
    // 1. 记录状态
    this.mutableMessages.push(message)
    this.totalUsage = accumulate(usage)

    // 2. 副作用（fire-and-forget）
    void recordTranscript(messages)

    // 3. 限制检查
    if (getTotalCost() >= maxBudgetUsd) {
      yield { type: 'result', subtype: 'error_max_budget_usd' }
      return
    }

    // 4. 透传给消费者
    yield* normalizeMessage(message)
  }

  // 5. 循环结束，yield 最终结果
  yield { type: 'result', subtype: 'success', cost, usage, ... }
}
```

### 实际的消费者处理速度

消费者做的事很轻（push 数组、fire-and-forget 写文件），真正慢的是**生产者自己内部的等待**：

```
yield assistant(text)     → 消费者 0.01ms 处理完 → .next() → 生产者继续
                                                              │
                            生产者内部：等 API 下一个 SSE event（200ms）
                            这才是真正"慢"的地方
                                                              │
yield assistant(tool_use) → 消费者 0.01ms 处理完 → .next() → 生产者继续
                                                              │
                            生产者内部：执行工具 + 可能等用户确认（3000ms）
                                                              │
yield user(tool_result)   → ...
```

---

## 工具并发执行系统

### 整体设计

模型可能一次返回多个 tool_use（如同时读 3 个文件）。StreamingToolExecutor 实现不等待的并发执行：

```
串行: Read A(5ms) → Read B(5ms) → Read C(5ms) = 15ms
并发: Read A(5ms) ┐
      Read B(5ms) ├─ max = 5ms
      Read C(5ms) ┘
```

JS 不是真并行（多线程同时跑 CPU），而是 IO 并发——多个 IO 操作同时在操作系统内核里跑，主线程只是"发指令"然后"等回调"。

### StreamingToolExecutor 核心结构

```javascript
class StreamingToolExecutor {
  private tools: TrackedTool[] = []
  
  addTool(block, assistantMessage) {
    this.tools.push({ id, block, status: 'queued', ... })
    void this.processQueue()  // 不 await！fire-and-forget
  }

  private async processQueue() {
    for (const tool of this.tools) {
      if (tool.status !== 'queued') continue
      if (this.canExecuteTool(tool.isConcurrencySafe)) {
        await this.executeTool(tool)  // ← 看起来阻塞，实则瞬间返回
      }
    }
  }

  private async executeTool(tool) {
    tool.status = 'executing'
    
    const collectResults = async () => {
      // 真正的 IO 工作在这里面
      for await (const update of runToolUse(...)) {
        messages.push(update.message)
      }
      tool.results = messages
      tool.status = 'completed'
    }

    const promise = collectResults()  // ← 创建 Promise = 立刻开始执行（不 await）
    tool.promise = promise
    
    void promise.finally(() => {
      void this.processQueue()  // 完成后尝试启动下一个排队的工具
    })
    // ← executeTool 到这里就 return 了，不等工具执行完
  }
}
```

### 为什么 `await this.executeTool(tool)` 不阻塞

`executeTool` 内部没有 await `collectResults()`：

```
await this.executeTool(tool)
       │
       │  executeTool 内部做了什么：
       │  ① tool.status = 'executing'     ← 同步，瞬间
       │  ② collectResults()              ← 创建 Promise（启动 IO），不 await
       │  ③ tool.promise = promise         ← 存引用，瞬间
       │  ④ promise.finally(→processQueue) ← 注册回调，瞬间
       │  ⑤ return                         ← 结束！
       │
       ▼
  await 等的是 ⑤ return，不是工具执行完毕
  所以几乎 0ms，for 循环继续下一个 tool
```

这就是 Promise + Generator 结合的设计精髓：
- `collectResults`（async 函数）：**创建即跑**，不需要外部 await 触发
- `runToolUse`（async generator）：需要 `for await` 驱动，否则一行都不跑
- 外层不 await `collectResults`：让工具在后台执行，不阻塞 processQueue

### 并发控制规则

```javascript
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 ||
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
  )
}
```

| 当前在执行的 | 新工具是 | 能否并行？ |
|---|---|---|
| 无 | 任何 | ✅ |
| [Read, Read] (并发安全) | Read (并发安全) | ✅ 一起跑 |
| [Read, Read] (并发安全) | Edit (非并发) | ❌ 排队等 |
| [Edit] (非并发) | 任何 | ❌ 排队等 |

设计意图：Read 之间可以并行（只读不冲突），Edit/Write 必须独占（有副作用）。

### 自驱动链

```javascript
void promise.finally(() => {
  void this.processQueue()  // A 完成 → 尝试启动 B
})
```

```
A 完成 → processQueue() → 发现 B 可以开始 → 启动 B
B 完成 → processQueue() → 发现 C 可以开始 → 启动 C
```

### runToolUse 完整执行链

每个工具内部经过完整的校验-权限-执行链：

```
StreamingToolExecutor.executeTool()
  └─ collectResults()
       └─ for await (runToolUse())         ← services/tools/toolExecution.ts
            └─ for await (streamedCheckPermissionsAndCallTool())
                 └─ checkPermissionsAndCallTool()
                      ├─ 1. findToolByName()           — 找到工具定义
                      ├─ 2. inputSchema.safeParse()    — Zod 输入校验
                      ├─ 3. tool.validateInput()       — 业务逻辑校验
                      ├─ 4. runPreToolUseHooks()       — 前置钩子
                      ├─ 5. canUseTool()               — 权限检查（可能阻塞等用户）
                      ├─ 6. tool.call()                — 真正执行工具
                      └─ 7. runPostToolUseHooks()      — 后置钩子 + 格式化结果
```

每一步失败都返回错误消息（不崩溃），让模型决定下一步：

| 步骤 | 失败情况 | 返回什么给模型 |
|------|---------|--------------|
| 找工具 | 工具名不存在 | `Error: No such tool available: xxx` |
| Zod 校验 | 参数类型错误 | `InputValidationError: expected string, got number` |
| 业务校验 | old_string 不存在 | `Error: old_string not found in file` |
| 前置钩子 | 钩子阻止 | `Execution stopped by PreToolUse hook` |
| 权限检查 | 用户拒绝 | `Error: User rejected tool use` |
| 工具执行 | 运行时错误 | `Error calling tool (Edit): ENOENT: file not found` |

### 两个取结果的入口

#### 入口 1：`getCompletedResults()`（非阻塞，API 流进行中）

```javascript
// query.ts 在 API 流迭代中调用
for await (const message of callModel({...})) {
  // ... 处理 API 返回 ...
  for (const result of streamingToolExecutor.getCompletedResults()) {
    yield result.message   // 有完成的就立刻 yield
  }
}
```

实现逻辑（同步 Generator，不是 async）：

```javascript
*getCompletedResults(): Generator<MessageUpdate, void> {
  for (const tool of this.tools) {
    while (tool.pendingProgress.length > 0) {
      yield { message: tool.pendingProgress.shift()! }  // 进度消息
    }
    if (tool.status === 'yielded') continue
    if (tool.status === 'completed' && tool.results) {
      tool.status = 'yielded'
      for (const message of tool.results) {
        yield { message, newContext: this.toolUseContext }
      }
    } else if (tool.status === 'executing' && !tool.isConcurrencySafe) {
      break  // 非并发工具还在跑，后面的都等着（保证顺序）
    }
  }
}
```

#### 入口 2：`getRemainingResults()`（阻塞等待，API 流结束后）

```javascript
// query.ts 在 API 流结束后调用
for await (const update of streamingToolExecutor.getRemainingResults()) {
  yield update.message
}
```

实现逻辑：

```javascript
async *getRemainingResults(): AsyncGenerator<MessageUpdate, void> {
  while (this.hasUnfinishedTools()) {
    await this.processQueue()

    for (const result of this.getCompletedResults()) {
      yield result
    }

    // 还有工具在跑但没新结果 → 等最快完成的那个
    if (this.hasExecutingTools() && !this.hasCompletedResults()) {
      await Promise.race(
        this.tools.filter(t => t.status === 'executing').map(t => t.promise!)
      )
    }
  }

  // 最后再取一次
  for (const result of this.getCompletedResults()) {
    yield result
  }
}
```

### 结果流动路径总结

```
runToolUse() yield
    ↓ for await 收集
collectResults() 内的 messages[]
    ↓ 执行完毕
tool.results = messages, tool.status = 'completed'
    ↓ 被取出（两个时机）
getCompletedResults()  — API 流进行中，非阻塞轮询
getRemainingResults()  — API 流结束后，阻塞等待
    ↓ yield 出去
query.ts
    ↓ yield 出去
QueryEngine（push mutableMessages，追踪 token）
    ↓ yield 出去
REPL / SDK（渲染终端 / 输出 JSON）
```

---

## 完整时间线示例

### 场景：用户输入 "帮我修复 auth.ts 的 bug"

#### 阶段 1：输入处理（不涉及 API）

```
[0ms] REPL handlePromptSubmit()
  ├─ processUserInput("帮我修复 auth.ts 的 bug")
  │   → 不是 / 开头，不是 ! 开头
  │   → 构造 UserMessage
  ├─ messages.push(userMessage)
  └─ 进入 query() 循环
```

#### 阶段 2：准备请求

```
[5ms] query() 第 1 轮 while(true) 顶部
  ├─ messagesForQuery = getMessagesAfterCompactBoundary(messages)
  ├─ microcompact(messagesForQuery)
  └─ addCacheBreakpoints()
```

#### 阶段 3：yield ① stream_request_start

```
[8ms] yield { type: 'stream_request_start' }
      → 不显示，仅内部标记"开始请求了"
```

#### 阶段 4：调用 API

```
[10ms] deps.callModel({
         messages: [{ role: "user", content: "帮我修复 auth.ts 的 bug" }],
         systemPrompt: "You are Claude Code...",
         tools: [BashTool, FileReadTool, FileEditTool, ...40个],
       })
       → 发起 HTTPS 请求，开始接收 SSE 流
```

#### 阶段 5：yield ② stream_event (message_start)

```
[120ms] yield { type: 'stream_event', event: { type: 'message_start',
          message: { usage: { input_tokens: 4200 } } } }
        → QueryEngine 记录 input_tokens
        → REPL 开始显示 spinner
        → 终端: ⠋ Claude is thinking...
```

#### 阶段 6：yield ③ stream_event (content_block_start/delta)

```
[150ms] yield { type: 'stream_event', event: { type: 'content_block_start',
          content_block: { type: 'text' } } }
        → (中间多个 content_block_delta，每个带几个 token)
        → 终端开始打字机效果
```

#### 阶段 7：yield ④ assistant message (文字回复)

```
[350ms] yield { type: 'assistant', message: {
          content: [{ type: 'text', text: '我来看一下 auth.ts 的内容。' }] } }
        → QueryEngine: mutableMessages.push()
        → REPL: 渲染文字到聊天界面
        → 终端:
          ┌──────────────────────────────────┐
          │ 我来看一下 auth.ts 的内容。        │
          └──────────────────────────────────┘
```

#### 阶段 8：yield ⑤ assistant message (工具调用)

```
[400ms] yield { type: 'assistant', message: {
          content: [{ type: 'tool_use', id: 'toolu_01ABC',
            name: 'Read', input: { file_path: '/project/auth.ts' } }] } }
        → query.ts: streamingToolExecutor.addTool(block) → 后台开始执行！
        → REPL: 显示 "Reading auth.ts..."
        → 终端:
          ┌──────────────────────────────────┐
          │ 我来看一下 auth.ts 的内容。        │
          │ 📖 Reading /project/auth.ts      │
          └──────────────────────────────────┘
```

#### 阶段 9：yield ⑥ stream_event (message_delta + message_stop)

```
[420ms] yield { type: 'stream_event', event: { type: 'message_delta',
          delta: { stop_reason: 'tool_use' }, usage: { output_tokens: 85 } } }
[425ms] yield { type: 'stream_event', event: { type: 'message_stop' } }
        → 记录 output_tokens，标记 stop_reason = 'tool_use'
        → 不显示到终端
```

#### 阶段 10：工具执行（在阶段 8 已启动）

```
[430ms] streamingToolExecutor 内部:
        ├─ canUseTool(Read, { file_path: 'auth.ts' })
        │   → 匹配 allow 规则 "Read(*)" → 无需确认
        ├─ FileReadTool.call({ file_path: '/project/auth.ts' })
        │   → 返回文件内容（50 行代码）
        └─ 构造 tool_result, tool.status = 'completed'
```

#### 阶段 11：yield ⑦ user message (工具结果)

```
[450ms] yield { type: 'user', message: {
          content: [{ type: 'tool_result', tool_use_id: 'toolu_01ABC',
            content: '1\tconst jwt = require(...)\n...' }] },
          toolUseResult: true }
        → QueryEngine: mutableMessages.push(), turnCount++
        → REPL: 显示文件内容预览（折叠）
        → 终端:
          ┌──────────────────────────────────┐
          │ 📖 Read /project/auth.ts (50 行) │  ← 可展开
          └──────────────────────────────────┘
```

#### 阶段 12：进入第 2 轮循环

```
[455ms] needsFollowUp = true → continue
        messages: [user("帮我修复"), assistant(text+tool_use), user(tool_result)]
        → 第 2 次 API 请求（前 2 条消息 cache hit → 只新内容计费）
```

#### 阶段 13：yield ⑧ assistant (分析 + Edit 工具调用)

```
[650ms] yield { type: 'assistant', message: { content: [
          { type: 'text', text: '找到问题了，第 15 行...' }] } }

[700ms] yield { type: 'assistant', message: { content: [
          { type: 'tool_use', name: 'Edit', input: {
            file_path: '/project/auth.ts',
            old_string: 'if (token.expired)',
            new_string: 'if (token.expired || !token.valid)' } }] } }
        → 终端:
          ┌──────────────────────────────────────────────────────┐
          │ 找到问题了，第 15 行的 token 验证逻辑有误。            │
          │ ✏️  Edit /project/auth.ts                             │
          │   - if (token.expired)                               │
          │   + if (token.expired || !token.valid)               │
          │ ┌────────────────────────────────────────────────┐   │
          │ │ Allow this edit?  [Y]es  [N]o  [A]lways allow  │   │
          │ └────────────────────────────────────────────────┘   │
          └──────────────────────────────────────────────────────┘
```

#### 阶段 14：权限确认（阻塞等待用户）

```
[700ms ~ 3000ms] 🚫 整个 query 循环暂停！
        canUseTool(Edit) → 无匹配规则 → 需用户确认
        → REPL 渲染权限对话框
        → 等待 Promise（用户按键）
        [3000ms] 用户按 'Y' → canUseTool 返回 { behavior: 'allow' }
        → FileEditTool.call() → 读+替换+写 → "Successfully edited"
```

#### 阶段 15：yield ⑨ user message (编辑结果)

```
[3010ms] yield { type: 'user', message: { content: [{ type: 'tool_result',
           tool_use_id: 'toolu_02DEF',
           content: 'Successfully edited auth.ts (+1/-1 lines)' }] } }
         → 终端: ✏️ Edit /project/auth.ts ✓
```

#### 阶段 16：进入第 3 轮循环

```
[3015ms] needsFollowUp = true → continue → 第 3 次 API 请求
         messages: [user, assistant, user(read结果), assistant, user(edit结果)]
         （前 4 条全部 cache hit）
```

#### 阶段 17：yield ⑩ assistant message (最终回复)

```
[3200ms] yield { type: 'assistant', message: { content: [
           { type: 'text', text: '已修复 auth.ts 第 15 行的 bug...' }] } }
```

#### 阶段 18：yield ⑪ stream_event (end_turn)

```
[3250ms] yield { type: 'stream_event', event: { type: 'message_delta',
           delta: { stop_reason: 'end_turn' } } }
         → needsFollowUp = false → while(true) 退出
```

#### 阶段 19：query() 返回，QueryEngine 收尾

```
[3260ms] query() generator 结束
         → result = messages.findLast(m => m.type === 'assistant')
         → flushSessionStorage()
```

#### 阶段 20：yield ⑫ 最终结果

```
[3270ms] yield { type: 'result', subtype: 'success',
           result: '已修复 auth.ts 第 15 行的 bug...',
           num_turns: 3, duration_ms: 3270,
           usage: { input_tokens: 8400, output_tokens: 520,
             cache_read_input_tokens: 6200, cache_creation_input_tokens: 2200 },
           stop_reason: 'end_turn' }
         → SDK(-p): 输出最终文本，退出进程
         → REPL: 恢复输入框
         → 终端:
           ┌──────────────────────────────────────────────────┐
           │ 已修复 auth.ts 第 15 行的 bug。                    │
           │ 问题原因：原来的条件只检查了 token 过期，           │
           │ 没有检查 token 是否有效...                         │
           │ > _                     ← 光标回到输入框           │
           └──────────────────────────────────────────────────┘
```

### Token 计费视角

```
第 1 次 API 调用:
  input:  system(8000) + tools(3000) + user(20)         = 11,020 tokens (全新，创建缓存)
  output: text(15) + tool_use(50)                       = 65 tokens

第 2 次 API 调用:
  input:  system(8000) + tools(3000) + history(135)     = 11,135 tokens
          其中 11,020 cache hit！只有 115 是新的           ← 省了 99% input 费用
  output: text(30) + tool_use(80)                       = 110 tokens

第 3 次 API 调用:
  input:  system(8000) + tools(3000) + history(325)     = 11,325 tokens
          其中 11,135 cache hit！只有 190 是新的
  output: text(120)                                     = 120 tokens

总计: 实际计费 input ≈ 500 tokens (而非 33,480)
      cache read ≈ 33,000 tokens (按 $0.30/M 计费，便宜 10 倍)
```

---

## 速查表

### 所有 yield 类型分类

| # | yield 内容 | 到终端？ | 作用 |
|---|-----------|---------|------|
| ① | `stream_request_start` | ❌ | 内部信号：标记请求开始 |
| ② | `stream_event: message_start` | ❌ | 追踪 input_tokens |
| ③ | `stream_event: content_block_*` | ❌/⚡ | 流式模式下实时推送片段 |
| ④ | `assistant` (文字) | ✅ | **显示模型的文字回复** |
| ⑤ | `assistant` (tool_use) | ✅ | **显示工具调用 UI** |
| ⑥ | `stream_event: message_delta/stop` | ❌ | 追踪 output_tokens + stop_reason |
| ⑦ | `user` (tool_result) | ✅ | **显示工具执行结果** |
| ⑧~⑩ | (循环重复 ④~⑦) | ✅ | 多轮对话 |
| ⑪ | `stream_event: end_turn` | ❌ | 触发循环退出 |
| ⑫ | `result` | ❌/✅ | SDK 输出最终结果 / REPL 恢复 UI |

**规律**：
- `assistant` / `user` 类型 → 显示到终端
- `stream_event` 类型 → 不显示（内部追踪）
- `result` → SDK 输出 / REPL 恢复状态

### 各层职责速查

| 层 | 输入 | 输出 | 不关心 |
|---|---|---|---|
| claude.ts | SSE bytes | 结构化 message | 工具、循环、token 预算 |
| query.ts | assistant messages | assistant + user 交替 | SSE 解析、token 计费、UI |
| QueryEngine | 全部 messages | SDKMessage | API 调用、工具执行、循环控制 |
| StreamingToolExecutor | tool_use block | tool_result message | API、循环、token |

### AsyncGenerator FAQ

| 问题 | 答案 |
|------|------|
| yield 后消费方马上收到吗？ | **是的**，yield 瞬间消费方就能通过 `.next()` 拿到 |
| 是一条条阻塞处理吗？ | **是的**，消费者处理完一条才拉下一条 |
| 如果瞬间 yield 很多呢？ | **不会发生**——每个 yield 点生产者都暂停等消费者 |
| 消费者慢会怎样？ | 整条管道暂停（背压），包括 API SSE 流也降速 |
| 卡住的原因是单线程吗？ | **不是**——是 Generator 协议设计，即使多线程语言也如此 |

### 并发控制规则

| 当前执行中 | 新工具 | 能否并行？ | 原因 |
|---|---|---|---|
| 无 | 任何 | ✅ | 空闲 |
| [Read, Read] | Read | ✅ | 全部并发安全 |
| [Read, Read] | Edit | ❌ | Edit 有副作用 |
| [Edit] | 任何 | ❌ | Edit 独占 |
