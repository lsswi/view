# Event Loop 机制与 React Scheduler 调度原理

## 一、Event Loop 机制

### 1.1 为什么需要 Event Loop

JS 是单线程的，同一时刻只能执行一段代码。Event Loop 是一个无限循环机制，用于协调同步代码、异步回调、渲染之间的执行顺序。

### 1.2 浏览器 Event Loop

#### 一轮循环的完整步骤

```
┌────────────────────────────────────────────────────────────┐
│                     Event Loop 一轮                         │
│                                                            │
│  ① 执行一个宏任务（macrotask）                              │
│     例：script 整体代码 / setTimeout 回调 / click 回调       │
│                                                            │
│  ② 清空所有微任务（microtask）                              │
│     例：Promise.then / queueMicrotask / MutationObserver    │
│     注意：微任务中产生的新微任务也会在这一步全部清完          │
│                                                            │
│  ③ 浏览器判断是否需要渲染（通常 16ms 一次）                 │
│     如果需要：requestAnimationFrame → Layout → Paint        │
│                                                            │
│  ④ 回到 ①，取下一个宏任务                                  │
└────────────────────────────────────────────────────────────┘
```

#### 关键规则

- 每轮只取**一个**宏任务执行
- 微任务**全部清完**才进入下一步（包括微任务中产生的新微任务）
- 一个 `<script>` 标签 / 一个模块文件 = 一个宏任务，不管代码多长都要从头到尾跑完
- JS 引擎没有能力在同步代码中间"暂停"——没有抢占式调度

#### 示例

```js
console.log('1');                          // 同步

setTimeout(() => console.log('2'), 0);    // 注册宏任务

Promise.resolve().then(() => {
  console.log('3');                        // 注册微任务
  Promise.resolve().then(() => {
    console.log('4');                      // 微任务中产生的新微任务
  });
});

console.log('5');                          // 同步

// 输出顺序：1 → 5 → 3 → 4 → 2
```

执行过程：

```
第 1 轮宏任务：整个 script
  ├── 打印 '1'
  ├── setTimeout → 放入宏任务队列
  ├── Promise.then → 放入微任务队列
  ├── 打印 '5'
  └── 调用栈清空
        ↓
清空微任务：
  ├── 执行 then 回调 → 打印 '3'，产生新微任务
  └── 执行新微任务 → 打印 '4'
        ↓
渲染（如果需要）
        ↓
第 2 轮宏任务：setTimeout 回调
  └── 打印 '2'
```

### 1.3 Node.js Event Loop

Node.js 的 Event Loop 分成固定的几个阶段，按顺序循环执行：

```
   ┌───────────────────────────┐
   │         timers            │ ← setTimeout / setInterval 回调
   │          (阶段1)          │
   ├───────────────────────────┤
   │       pending I/O         │ ← 系统级回调（TCP 错误等）
   │          (阶段2)          │
   ├───────────────────────────┤
   │       idle / prepare      │ ← 内部使用
   │          (阶段3)          │
   ├───────────────────────────┤
   │         poll              │ ← I/O 回调（fs.readFile、网络请求等）
   │          (阶段4)          │    如果没有待处理的 I/O，会在这里等待
   ├───────────────────────────┤
   │         check             │ ← setImmediate 回调
   │          (阶段5)          │
   ├───────────────────────────┤
   │      close callbacks      │ ← socket.on('close') 等
   │          (阶段6)          │
   └───────────────────────────┘
              ↓
         回到 timers，下一轮
```

#### 与浏览器的区别

| 维度 | 浏览器 | Node.js |
|------|--------|---------|
| 结构 | 简单的"取一个宏任务" | 分阶段循环 |
| 渲染 | 有渲染步骤 | 无渲染概念 |
| 微任务执行时机 | 宏任务后 | 每个阶段之间 |
| 独有 API | MessageChannel、rAF | setImmediate、process.nextTick |

### 1.4 创建宏任务的操作

| 操作 | 环境 | 说明 |
|------|------|------|
| `<script>` 标签 | 浏览器 | 每个 script 标签是一个独立宏任务 |
| `setTimeout(fn, ms)` | 通用 | 延迟后放入队列 |
| `setInterval(fn, ms)` | 通用 | 周期性放入队列 |
| `setImmediate(fn)` | Node.js | I/O 后 check 阶段执行 |
| `MessageChannel.postMessage()` | 浏览器 | 产生一个无延迟的宏任务 |
| `requestAnimationFrame(fn)` | 浏览器 | 下一帧渲染前执行 |
| I/O 回调 | Node.js | fs.readFile / net.socket 等完成后 |
| 用户交互事件 | 浏览器 | click / keydown / scroll 等 |

### 1.5 创建微任务的操作

| 操作 | 环境 | 说明 |
|------|------|------|
| `Promise.then/catch/finally` | 通用 | Promise 状态变化后的回调 |
| `queueMicrotask(fn)` | 通用 | 最直接的创建方式 |
| `async/await` | 通用 | await 之后的代码相当于 .then() |
| `MutationObserver` | 浏览器 | DOM 变化监听的回调 |
| `process.nextTick(fn)` | Node.js | 比其他微任务更优先（特殊） |

### 1.6 setTimeout 的延迟不可控

```
实际执行时间 = max(设定延迟, 最小延迟) + 排队等待时间
```

延迟来源：

| 来源 | 能控制吗 | 说明 |
|------|---------|------|
| delay 参数本身 | ✅ | 你设定的 |
| 4ms 最小延迟 | ❌ | 嵌套 ≥ 5 层时浏览器强制 |
| 当前同步代码没跑完 | ❌ | 必须等调用栈清空 |
| 微任务队列没清完 | ❌ | 微任务优先级更高 |
| 前面有其他宏任务排队 | ❌ | 先来先服务 |

`setTimeout(fn, 100)` 的含义是"至少 100ms 后执行"，不是"精确 100ms 后执行"。

### 1.7 setTimeout 嵌套与 4ms 惩罚

嵌套是指 setTimeout 的回调里面又调 setTimeout，形成链式调用：

```js
function work() {
  // 做一些工作...
  setTimeout(work, 0);  // 回调里递归调度 → 嵌套层级 +1
}
work();

// 第 1~4 次：间隔 ~1ms（基础开销）
// 第 5 次起：间隔 ~4ms（浏览器强制最小延迟）
```

并列调用不算嵌套：

```js
setTimeout(fn1, 0);  // 各自独立，都是第 1 层
setTimeout(fn2, 0);
setTimeout(fn3, 0);
```

---

## 二、React Scheduler 的三种调度机制

### 2.1 为什么需要调度

React 的组件树 diff 可能很耗时，同步执行会阻塞主线程。React 18 引入时间切片：每 5ms 让出一次主线程，让浏览器有机会渲染和响应用户输入。

"让出"之后，需要一种机制来"回来继续干活"——这就是调度器的作用。

### 2.2 React 的特性检测逻辑

```js
// node_modules/scheduler 源码（简化）
if (typeof setImmediate === 'function') {
  // 优先用 setImmediate（Node.js）
  schedulePerformWorkUntilDeadline = () => {
    setImmediate(performWorkUntilDeadline);
  };
} else if (typeof MessageChannel !== 'undefined') {
  // 其次用 MessageChannel（浏览器）
  const channel = new MessageChannel();
  channel.port1.onmessage = performWorkUntilDeadline;
  schedulePerformWorkUntilDeadline = () => {
    channel.port2.postMessage(null);
  };
} else {
  // 兜底用 setTimeout
  schedulePerformWorkUntilDeadline = () => {
    setTimeout(performWorkUntilDeadline, 0);
  };
}
```

不是根据"引擎是什么"判断，而是通过 `typeof` 检测全局 API 是否存在——鸭子类型。

### 2.3 三种机制的原理对比

#### setTimeout(fn, 0)

```
setTimeout(fn, 0)
     ↓
① 创建 Timer 对象
     ↓
② 注册到定时器线程（独立于 JS 主线程）
     ↓
③ 定时器线程开始计时
     ↓
④ 规范检查：嵌套 ≥ 5 且 delay < 4？→ 强制 delay = 4ms
     ↓
⑤ 等待到期
     ↓
⑥ 把 fn 放入宏任务队列
     ↓
⑦ 等 Event Loop 取出执行
```

#### MessageChannel

```
port2.postMessage(null)
     ↓
① 直接把 onmessage 回调放入宏任务队列（无计时环节）
     ↓
② 等 Event Loop 取出执行
```

#### setImmediate（Node.js）

```
setImmediate(fn)
     ↓
① 直接放入 check 阶段队列（无计时环节）
     ↓
② 当前轮 poll 阶段结束后立即执行
```

### 2.4 核心区别：经不经过定时器系统

| 步骤 | setTimeout | MessageChannel | setImmediate |
|------|-----------|----------------|--------------|
| 创建 Timer 对象 | ✅ | ❌ | ❌ |
| 注册到定时器堆 | ✅（堆排序 O(log n)） | ❌ | ❌ |
| 通知定时器线程 | ✅（跨线程通信） | ❌ | ❌ |
| 嵌套层级检查 | ✅ | ❌ | ❌ |
| 等待计时到期 | ✅ | ❌ | ❌ |
| 放入宏任务队列 | ✅ | ✅ | ✅ |

**setTimeout 的 0 不是"零开销"，而是"零延迟设定值"，系统处理这个设定本身就有开销。**

**MessageChannel 和 setImmediate 根本不走定时器系统，直接往队列里放任务。**

### 2.5 性能差异

```
setTimeout 方案（嵌套后）：
│ work 5ms │ 4ms等待 │ work 5ms │ 4ms等待 │ work 5ms │
  一帧 16ms 内只能完成 ~2 个切片

MessageChannel / setImmediate 方案：
│ work 5ms │~0ms│ work 5ms │~0ms│ work 5ms │
  一帧 16ms 内可以完成 ~3 个切片
```

### 2.6 setImmediate 在 Node.js 中为什么更快

在 I/O 回调中调度时：

```
当前轮 Event Loop：
... → poll（I/O 回调执行）→ check（setImmediate 执行）→ ...
                                ↑ 同一轮，紧跟着 I/O

下一轮 Event Loop：
timers（setTimeout 执行）→ ...
   ↑ 多等了一整轮
```

```js
const fs = require('fs');
fs.readFile('file.txt', () => {
  setTimeout(() => console.log('timeout'), 0);   // 下一轮 timers
  setImmediate(() => console.log('immediate'), 0); // 当前轮 check
});
// 输出：immediate → timeout（setImmediate 永远先）
```

### 2.7 总结表

| 特性 | setTimeout(fn, 0) | MessageChannel | setImmediate |
|------|-------------------|----------------|--------------|
| 环境 | 所有 | 浏览器 | Node.js |
| 经过定时器系统 | ✅ | ❌ | ❌ |
| 有最小延迟 | ✅ 4ms（嵌套≥5） | ❌ | ❌ |
| 本质语义 | "N ms 后做" | "下一轮立刻做" | "I/O 完了立刻做" |
| 设计目的 | 延时执行 | 消息通信 | 异步调度 |
| React 优先级 | 第 3（兜底） | 第 2（浏览器首选） | 第 1（Node.js 首选） |

---

## 三、React 时间切片的实现原理

### 3.1 为什么能"暂停"

React 16+ 用 Fiber 架构把组件树转成链表结构，用 while 循环逐个处理。进度保存在 `currentFiber` 变量里，`return` 出去后数据还在内存中，下次进来接着走：

```js
// ❌ 递归方式 — 进度在调用栈里，return 了就丢了
function diffTree(node) {
  process(node);
  for (const child of node.children) {
    diffTree(child);  // 层层嵌套，无法中途退出
  }
}

// ✅ Fiber 链表方式 — 进度在变量里，随时能停
let currentFiber = rootFiber;

function performUnitOfWork(fiber) {
  process(fiber);
  return getNextFiber(fiber);  // 返回下一个待处理节点
}
```

### 3.2 workLoop 核心逻辑

```js
const TIME_SLICE = 5; // 5ms

function workLoop() {
  const deadline = performance.now() + TIME_SLICE;

  while (currentFiber !== null) {
    currentFiber = performUnitOfWork(currentFiber);

    if (performance.now() >= deadline) {
      // 时间到了，让出主线程
      return true;  // 还有工作没做完
    }
  }
  return false;  // 全部完成
}

function scheduleWork() {
  const hasMore = workLoop();
  if (hasMore) {
    // 用 MessageChannel / setImmediate 把自己排到下一个宏任务
    scheduleCallback(scheduleWork);
  }
}
```

### 3.3 完整流程

```
React setState
     ↓
Scheduler 创建任务（带优先级）
     ↓
scheduleCallback → MessageChannel.postMessage() / setImmediate()
     ↓
下一个宏任务开始 → workLoop 执行
     ↓
while 循环处理 fiber 节点，每处理一个检查时间
     ↓
超过 5ms → return 出来 → 再次 scheduleCallback
     ↓
Event Loop 有机会：渲染 / 处理用户输入 / 处理其他任务
     ↓
下一个宏任务 → workLoop 从 currentFiber 继续
     ↓
全部完成 → commit 阶段（同步，不可打断）→ DOM 更新
```

### 3.4 类比

```
传统递归（React 15）= 看书一口气看完
  → 500 页看完之前不响应任何事情

Fiber 链表（React 18）= 看书可以夹书签
  → 每看几页检查时间
  → 到点了夹上书签，去做别的
  → 回来打开书签继续

"书签"     = currentFiber
"看一页"   = performUnitOfWork
"检查时间" = performance.now() >= deadline
"去做别的" = return + scheduleCallback
```
