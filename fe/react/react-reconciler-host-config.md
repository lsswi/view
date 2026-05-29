# React Reconciler HostConfig 完全指南

> 基于 react-reconciler@0.29.0 分析，结合 celeste 项目 LightEngineHostConfig 实现。

---

## 一、React Reconciler 整体架构

```
用户代码 (JSX)
     │
     ▼
React.createElement → ReactElement 虚拟树
     │
     ▼
┌─────────────────────────────────────────────┐
│           React Reconciler                   │
│                                             │
│  ┌─────────┐    ┌──────────┐    ┌────────┐ │
│  │ Render  │ →  │ Commit   │ →  │ Passive│ │
│  │ Phase   │    │ Phase    │    │ Effects│ │
│  └─────────┘    └──────────┘    └────────┘ │
│       │              │                      │
│       ▼              ▼                      │
│  HostConfig.*   HostConfig.*                │
└─────────────────────────────────────────────┘
     │
     ▼
宿主环境 (DOM / Native / Canvas / LightEngine...)
```

Reconciler 本身是**平台无关**的。它通过 `HostConfig` 接口将所有宿主环境操作委托出去，实现一次协调逻辑、多平台复用。

---

## 二、核心阶段详解

### 2.1 Render 阶段（也叫 Reconciliation 阶段）

#### 特征

- **可中断**：在 Concurrent 模式下可以暂停和恢复
- **无副作用**：不会修改宿主环境的真实节点
- **纯计算**：只产生 "effect 标记"（effectTag / flags），记录需要做什么

#### 工作内容

Reconciler 从根 Fiber 开始，深度优先遍历整棵 Fiber 树：

```
beginWork (向下) → 处理每个 Fiber 节点
    │
    ├── 函数组件 → 执行函数，得到 children ReactElement
    ├── 类组件   → 调用 render()，得到 children ReactElement
    ├── 宿主组件 → 调用 HostConfig 方法创建/对比实例
    │
    ▼
completeWork (向上) → 完成该节点的处理
```

#### 在 Render 阶段调用的 HostConfig 方法

| 方法 | 调用时机 | 说明 |
|------|----------|------|
| `shouldSetTextContent(type, props)` | beginWork 处理宿主组件时最先调用 | 决定是否将子文本当作该组件的内部属性处理 |
| `getRootHostContext(rootContainer)` | 处理 HostRoot 时 | 产生根级上下文 |
| `getChildHostContext(parentCtx, type, container)` | 处理每个宿主组件的 beginWork | 派生子上下文（如 SVG 命名空间切换） |
| `createInstance(type, props, container, hostCtx, fiber)` | completeWork 中，首次挂载宿主节点时 | 创建宿主实例（离屏，还未挂载） |
| `createTextInstance(text, container, hostCtx, fiber)` | completeWork 中，首次挂载文本节点时 | 创建文本实例 |
| `appendInitialChild(parent, child)` | completeWork 中，将已完成的子节点挂到父节点 | 构建离屏子树 |
| `finalizeInitialChildren(instance, type, props, container, hostCtx)` | completeWork 末尾 | 最终初始化，返回 true 则标记需要 commitMount |
| `prepareUpdate(instance, type, oldProps, newProps, container, hostCtx)` | completeWork 中，更新路径 | 计算 updatePayload，null 表示无需更新 |

#### Render 阶段的关键约束

```
┌───────────────────────────────────────────────────┐
│  Render 阶段的代码必须是幂等的、无副作用的。        │
│  因为 Concurrent 模式下可能被中断并重新执行。        │
│  所以 HostConfig 中 render 阶段的方法不应修改         │
│  任何宿主环境的可见状态。                           │
└───────────────────────────────────────────────────┘
```

---

### 2.2 Commit 阶段

#### 特征

- **同步执行**：一旦开始不可中断
- **有副作用**：真正修改宿主环境的节点树
- **三个子阶段**：Before Mutation → Mutation → Layout

#### 三个子阶段详解

```
Commit Phase
│
├── 1. Before Mutation 阶段
│   ├── prepareForCommit(container)         ← 保存宿主状态（如 DOM selection）
│   ├── getSnapshotBeforeUpdate()           ← 类组件生命周期
│   └── 读取变更前的宿主状态
│
├── 2. Mutation 阶段 ★ 核心修改发生在这里
│   ├── commitUpdate(instance, payload, type, prevProps, nextProps, fiber)
│   ├── commitTextUpdate(textInst, oldText, newText)
│   ├── appendChild(parent, child)
│   ├── appendChildToContainer(container, child)
│   ├── insertBefore(parent, child, beforeChild)
│   ├── insertInContainerBefore(container, child, beforeChild)
│   ├── removeChild(parent, child)
│   ├── removeChildFromContainer(container, child)
│   ├── resetTextContent(instance)
│   ├── clearContainer(container)
│   ├── hideInstance(instance) / unhideInstance(instance)
│   ├── hideTextInstance(inst) / unhideTextInstance(inst)
│   └── detachDeletedInstance(node)         ← 清理引用防止内存泄漏
│
├── 3. Layout 阶段
│   ├── commitMount(instance, type, props, fiber)  ← finalizeInitialChildren 返回 true 时
│   ├── componentDidMount / componentDidUpdate     ← 类组件生命周期
│   ├── useLayoutEffect 的回调执行
│   └── ref 的设置（getPublicInstance 在此时被调用）
│
└── resetAfterCommit(container)             ← 恢复宿主状态
```

#### Mutation 子阶段的操作顺序

```
对于每个标记了 effect 的 Fiber 节点，按以下顺序处理：

1. Deletion（删除）
   → removeChild / removeChildFromContainer
   → detachDeletedInstance

2. Placement（新增/移动）
   → appendChild / appendChildToContainer
   → insertBefore / insertInContainerBefore

3. Update（属性更新）
   → commitUpdate / commitTextUpdate

4. Visibility（可见性变化，Suspense/Offscreen）
   → hideInstance / unhideInstance
   → hideTextInstance / unhideTextInstance
```

---

### 2.3 Passive Effects 阶段

#### 特征

- **异步执行**：在浏览器绘制后，通过 `requestIdleCallback` 或 MessageChannel 调度
- 不涉及 HostConfig 方法
- 执行 `useEffect` 的清理和回调

```
Paint (浏览器绘制)
     │
     ▼
Passive Effects 阶段
     ├── 执行上一轮 useEffect 的清理函数
     └── 执行本轮 useEffect 的回调函数
```

---

## 三、两种渲染模式

### 3.1 Mutation 模式（`supportsMutation: true`）

类似 DOM 的操作方式：直接在现有树上增删改。

```
现有树:  A → [B, C, D]
操作:    removeChild(A, C)  →  A → [B, D]
         appendChild(A, E)  →  A → [B, D, E]
```

**必须实现的方法：**
- `appendChild`, `appendChildToContainer`
- `insertBefore`, `insertInContainerBefore`
- `removeChild`, `removeChildFromContainer`
- `commitUpdate`, `commitTextUpdate`
- `resetTextContent`, `clearContainer`

---

### 3.2 Persistence 模式（`supportsPersistence: true`）

不可变树的方式：每次更新创建新的子节点集合，然后原子性替换。

```
旧树:  Container → [A, B, C]
           ↓ 创建新 ChildSet
新集合: [A, B', C, D]    ← B 更新为 B'，新增 D
           ↓ 原子替换
新树:  Container → [A, B', C, D]
```

**必须实现的方法：**
- `createContainerChildSet(container)` — 创建空的子节点集合
- `appendChildToContainerChildSet(childSet, child)` — 向集合中追加
- `finalizeContainerChildren(container, newChildren)` — 替换前的最终处理
- `replaceContainerChildren(container, newChildren)` — 原子替换

**适用场景：** 不可变数据结构的渲染目标（如 React Native Fabric 的某些路径）。

---

### 3.3 对比

| 特征 | Mutation | Persistence |
|------|----------|-------------|
| 操作方式 | 逐个节点增删改 | 整体替换子集合 |
| 性能模型 | 细粒度，操作少时快 | 粗粒度，节点多时可能慢 |
| 实现复杂度 | 需要处理各种增删场景 | 接口简单但需要克隆能力 |
| 典型宿主 | DOM、大部分 Native | 函数式/不可变渲染目标 |

> 注意：通常**二选一**，同时开启两者时 Reconciler 行为取决于具体版本实现。

---

## 四、HostConfig 接口完整分类

### 4.1 必须实现（无论哪种模式）

```typescript
interface RequiredHostConfig {
  // ========== 创建 ==========
  // 创建宿主元素实例
  createInstance(
    type: Type,
    props: Props,
    rootContainer: Container,
    hostContext: HostContext,
    internalHandle: Fiber
  ): Instance;

  // 创建文本节点实例
  createTextInstance(
    text: string,
    rootContainer: Container,
    hostContext: HostContext,
    internalHandle: Fiber
  ): TextInstance;

  // ========== 判断 ==========
  // 该类型是否自行管理文本内容
  shouldSetTextContent(type: Type, props: Props): boolean;

  // ========== 上下文 ==========
  // 根级宿主上下文
  getRootHostContext(rootContainer: Container): HostContext;

  // 子级宿主上下文（可基于父上下文和类型派生）
  getChildHostContext(
    parentHostContext: HostContext,
    type: Type,
    rootContainer: Container
  ): HostContext;

  // ========== 初始化组装 ==========
  // render 阶段：将初始子节点挂到父实例（离屏）
  appendInitialChild(parentInstance: Instance, child: Instance | TextInstance): void;

  // render 阶段：最终初始化，返回 true 则需要 commitMount
  finalizeInitialChildren(
    instance: Instance,
    type: Type,
    props: Props,
    rootContainer: Container,
    hostContext: HostContext
  ): boolean;

  // ========== 更新准备 ==========
  // render 阶段：对比 props 产出 updatePayload
  prepareUpdate(
    instance: Instance,
    type: Type,
    oldProps: Props,
    newProps: Props,
    rootContainer: Container,
    hostContext: HostContext
  ): UpdatePayload | null;

  // ========== Commit 前后 ==========
  prepareForCommit(containerInfo: Container): Record<string, any> | null;
  resetAfterCommit(containerInfo: Container): void;

  // ========== 公共实例 ==========
  getPublicInstance(instance: Instance): PublicInstance;

  // ========== 调度 ==========
  scheduleTimeout(fn: (...args: unknown[]) => unknown, delay?: number): TimeoutHandle;
  cancelTimeout(id: TimeoutHandle): void;
  noTimeout: TimeoutHandle; // 哨兵值

  // ========== 优先级 ==========
  getCurrentEventPriority(): number;

  // ========== 清理 ==========
  detachDeletedInstance(node: Instance): void;

  // ========== 配置标志 ==========
  supportsMutation: boolean;
  supportsPersistence: boolean;
  supportsHydration: boolean;
  isPrimaryRenderer: boolean;
}
```

### 4.2 Mutation 模式必须实现

```typescript
interface MutationHostConfig {
  // 添加子节点到末尾
  appendChild(parentInstance: Instance, child: Instance | TextInstance): void;
  appendChildToContainer(container: Container, child: Instance | TextInstance): void;

  // 在指定节点前插入
  insertBefore(
    parentInstance: Instance,
    child: Instance | TextInstance,
    beforeChild: Instance | TextInstance
  ): void;
  insertInContainerBefore(
    container: Container,
    child: Instance | TextInstance,
    beforeChild: Instance | TextInstance
  ): void;

  // 移除子节点
  removeChild(parentInstance: Instance, child: Instance | TextInstance): void;
  removeChildFromContainer(container: Container, child: Instance | TextInstance): void;

  // 属性更新
  commitUpdate(
    instance: Instance,
    updatePayload: UpdatePayload,
    type: Type,
    prevProps: Props,
    nextProps: Props,
    internalHandle: Fiber
  ): void;

  // 文本更新
  commitTextUpdate(textInstance: TextInstance, oldText: string, newText: string): void;

  // 文本内容重置
  resetTextContent(instance: Instance): void;

  // 清空容器
  clearContainer(container: Container): void;
}
```

### 4.3 Persistence 模式必须实现

```typescript
interface PersistenceHostConfig {
  // 创建空的子节点集合
  createContainerChildSet(container: Container): ChildSet;

  // 向集合追加子节点
  appendChildToContainerChildSet(childSet: ChildSet, child: Instance | TextInstance): void;

  // 替换前的最终处理
  finalizeContainerChildren(container: Container, newChildren: ChildSet): void;

  // 原子替换
  replaceContainerChildren(container: Container, newChildren: ChildSet): void;
}
```

### 4.4 可见性控制（Suspense / Offscreen）

```typescript
interface VisibilityHostConfig {
  hideInstance(instance: Instance): void;
  unhideInstance(instance: Instance, props: Props): void;
  hideTextInstance(textInstance: TextInstance): void;
  unhideTextInstance(textInstance: TextInstance, text: string): void;
}
```

### 4.5 可空实现（声明即可，通常返回 null / 空函数）

```typescript
interface OptionalHostConfig {
  // Portal 挂载前准备
  preparePortalMount(containerInfo: Container): void;

  // 事件系统
  getInstanceFromNode(node: any): Fiber | null | undefined;
  beforeActiveInstanceBlur(): void;
  afterActiveInstanceBlur(): void;

  // Scope API（实验性）
  prepareScopeUpdate(scopeInstance: any, instance: any): void;
  getInstanceFromScope(scopeInstance: any): Instance | null;
}
```

---

## 五、完整生命周期流程图

### 5.1 首次渲染（Mount）

```
ReactReconciler.createContainer(rootContainer)
ReactReconciler.updateContainer(element, container)
│
▼ ─── Render 阶段 ───
│
├── getRootHostContext(rootContainer)
│
├── 遍历 Fiber 树 (beginWork → completeWork)
│   │
│   ├── 对每个宿主组件:
│   │   ├── getChildHostContext(parentCtx, type, container)
│   │   ├── shouldSetTextContent(type, props)
│   │   ├── createInstance(type, props, container, hostCtx, fiber)
│   │   │   或 createTextInstance(text, container, hostCtx, fiber)
│   │   ├── appendInitialChild(parent, child)  ← 对每个子节点
│   │   └── finalizeInitialChildren(instance, type, props, container, hostCtx)
│   │
│   └── 标记 effect flags (Placement, Update 等)
│
▼ ─── Commit 阶段 ───
│
├── Before Mutation
│   └── prepareForCommit(container)
│
├── Mutation
│   └── appendChildToContainer(container, rootChild)  ← 首次挂载根节点
│
├── Layout
│   ├── commitMount(instance, type, props, fiber)  ← 如果 finalizeInitialChildren 返回 true
│   └── ref 赋值 (getPublicInstance)
│
└── resetAfterCommit(container)
```

### 5.2 更新（Update）

```
setState / forceUpdate / parent re-render
│
▼ ─── Render 阶段 ───
│
├── 遍历变化的 Fiber 子树
│   │
│   ├── 对需要更新的宿主组件:
│   │   └── prepareUpdate(instance, type, oldProps, newProps, container, hostCtx)
│   │       → 返回 updatePayload (非 null 则标记 Update flag)
│   │
│   ├── 对新增的宿主组件:
│   │   ├── createInstance / createTextInstance
│   │   ├── appendInitialChild
│   │   └── finalizeInitialChildren
│   │   → 标记 Placement flag
│   │
│   └── 对删除的组件:
│       → 标记 Deletion flag
│
▼ ─── Commit 阶段 ───
│
├── Before Mutation
│   └── prepareForCommit(container)
│
├── Mutation (按 effect list 顺序处理)
│   │
│   ├── Deletion:
│   │   ├── removeChild(parent, child)
│   │   │   或 removeChildFromContainer(container, child)
│   │   └── detachDeletedInstance(node)
│   │
│   ├── Placement:
│   │   ├── appendChild(parent, child)
│   │   │   或 appendChildToContainer(container, child)
│   │   │   或 insertBefore(parent, child, beforeChild)
│   │   │   或 insertInContainerBefore(container, child, beforeChild)
│   │   └── (选择哪个取决于是否有 sibling 需要插入到其前面)
│   │
│   └── Update:
│       ├── commitUpdate(instance, payload, type, prevProps, nextProps, fiber)
│       └── commitTextUpdate(textInstance, oldText, newText)
│
├── Layout
│   └── 生命周期、useLayoutEffect、ref
│
└── resetAfterCommit(container)
```

### 5.3 卸载（Unmount）

```
root.unmount() 或 条件渲染移除
│
▼ ─── Commit 阶段 ───
│
├── Mutation
│   ├── 递归对子树每个节点:
│   │   ├── removeChild(parent, child)
│   │   └── detachDeletedInstance(node)
│   └── removeChildFromContainer(container, rootChild)  ← 根节点
│
└── clearContainer(container)  ← 如果是 root.unmount()
```

---

## 六、调用时机速查表

| HostConfig 方法 | 阶段 | 触发条件 |
|----------------|------|----------|
| `getRootHostContext` | Render | 处理根 Fiber |
| `getChildHostContext` | Render | 处理每个宿主组件的 beginWork |
| `shouldSetTextContent` | Render | 处理宿主组件之前，决定文本策略 |
| `createInstance` | Render (completeWork) | 首次挂载宿主组件 |
| `createTextInstance` | Render (completeWork) | 首次挂载文本节点 |
| `appendInitialChild` | Render (completeWork) | 将子实例添加到刚创建的父实例 |
| `finalizeInitialChildren` | Render (completeWork) | 所有初始子节点添加完毕后 |
| `prepareUpdate` | Render (completeWork) | 宿主组件 props 变化时 |
| `prepareForCommit` | Commit - Before Mutation | Commit 阶段开始 |
| `commitUpdate` | Commit - Mutation | prepareUpdate 返回非 null |
| `commitTextUpdate` | Commit - Mutation | 文本内容变化 |
| `appendChild` | Commit - Mutation | 子节点被新增到非根父节点 |
| `appendChildToContainer` | Commit - Mutation | 子节点被新增到根容器 |
| `insertBefore` | Commit - Mutation | 需要在特定位置插入（非末尾） |
| `insertInContainerBefore` | Commit - Mutation | 在根容器特定位置插入 |
| `removeChild` | Commit - Mutation | 节点从非根父节点移除 |
| `removeChildFromContainer` | Commit - Mutation | 节点从根容器移除 |
| `resetTextContent` | Commit - Mutation | 文本容器的子节点类型变化 |
| `clearContainer` | Commit - Mutation | 根容器需要完全清空 |
| `hideInstance` / `unhideInstance` | Commit - Mutation | Suspense/Offscreen 切换 |
| `detachDeletedInstance` | Commit - Mutation | 节点删除后清理引用 |
| `resetAfterCommit` | Commit - 结束 | Commit 阶段完成 |
| `getPublicInstance` | Commit - Layout | ref 赋值时 |
| `getCurrentEventPriority` | 随时 | Reconciler 确定更新优先级 |

---

## 七、Effect Flags（内部标记）

Reconciler 在 Render 阶段为 Fiber 打上 flags，Commit 阶段据此执行操作：

| Flag | 含义 | 对应 HostConfig 操作 |
|------|------|---------------------|
| `Placement` | 新增或移动 | appendChild / insertBefore |
| `Update` | 属性变化 | commitUpdate / commitTextUpdate |
| `Deletion` | 删除 | removeChild + detachDeletedInstance |
| `Visibility` | 显隐切换 | hideInstance / unhideInstance |
| `Ref` | ref 需要更新 | getPublicInstance |
| `Snapshot` | 需要快照 | prepareForCommit |

---

## 八、优先级系统（与 HostConfig 的交互）

```
事件触发
  │
  ▼
getCurrentEventPriority()
  │
  ├── 1 = DiscreteEventPriority (点击、输入等离散事件)
  ├── 4 = ContinuousEventPriority (拖拽、滚动等连续事件)
  └── 16 = DefaultEventPriority (默认，如 setTimeout 回调)
  │
  ▼
Reconciler 根据优先级决定:
  ├── 是否可以打断当前 Render
  ├── 本次更新的 lane（车道）
  └── 调度策略（同步 or 时间切片）
```

---

## 九、与 LightEngine 项目的映射关系

```
React JSX               HostConfig 方法              LightEngine Native
─────────               ──────────────              ──────────────────
<div>                → createInstance('div')       → FlexContainerImpl
                                                      → new LightEngineFlexContainer()
<img src="x.png">   → createInstance('img')       → ImageElementImpl
                                                      → new LightEngineImage()
<p>Hello</p>         → shouldSetTextContent → true
                       (不会 createTextInstance)    → TextElementImpl 内部处理文本

props 变化           → prepareUpdate → true
                     → commitUpdate               → instance.updateProps(newProps)
                                                      → updateBaseProps() 链式调用
                                                      → Native setXxx() 方法

节点删除             → removeChild                 → parentInstance.removeChild(child)
                                                      → Native removeChild()
```

---

## 十、实现 HostConfig 的最佳实践

### 10.1 性能优化

```typescript
// ❌ 不好：prepareUpdate 总是返回 true，导致每次都 commitUpdate
prepareUpdate(instance, type, oldProps, newProps) {
  return true;
}

// ✅ 好：精细对比，只在有差异时返回 payload
prepareUpdate(instance, type, oldProps, newProps) {
  const diff = diffProps(oldProps, newProps);
  return diff.length > 0 ? diff : null;
}
```

### 10.2 insertBefore 不可忽略

```typescript
// ❌ 危险：空实现会导致列表顺序错乱
insertBefore(parent, child, beforeChild) {}

// ✅ 正确：必须实现节点的位置插入
insertBefore(parent, child, beforeChild) {
  const index = parent.children.indexOf(beforeChild);
  parent.children.splice(index, 0, child);
  parent.nativeInsertBefore(child.nativeNode, beforeChild.nativeNode);
}
```

### 10.3 detachDeletedInstance 防内存泄漏

```typescript
// ✅ 清理循环引用和 Native 资源
detachDeletedInstance(node: Instance) {
  node.parentNode = null;
  node.childrenNodes = [];
  node.lightEngineComponentInstance?.destroy();
  node.lightEngineComponentInstance = null;
}
```

### 10.4 模式选择

- 如果宿主环境是可变的（如 DOM、Native View 树）→ 选 **Mutation 模式**
- 如果宿主环境是不可变的（如 Canvas 重绘、游戏引擎 Scene Graph）→ 选 **Persistence 模式**
- **不要同时开启两者**，除非明确了解 Reconciler 在该版本下的行为

---

## 十一、文本处理机制（shouldSetTextContent 与 createTextInstance）

### 11.1 核心问题：文字怎么到达组件

JSX 中的文字最终会出现在 `props.children` 里：

```jsx
<p>Hello World</p>
// 编译为:
React.createElement('p', null, 'Hello World')
// 产出: { type: 'p', props: { children: 'Hello World' } }
```

React 需要决定：这段文字是**由父组件内部消化**，还是**创建独立的文本子节点**。

### 11.2 决策流程

```
beginWork 处理宿主组件时：

props.children 包含文字
        │
        ▼
shouldSetTextContent(parentType, parentProps)
        │
        ├── true  → 文字留在 props.children 里
        │           → 不为文字创建子 Fiber
        │           → createInstance(parentType, props) 时组件内部从 props.children 读取文字
        │           → 不会调用 createTextInstance
        │
        └── false → 文字作为独立子节点
                    → 创建 Fiber { tag: HostText, pendingProps: 'Hello World' }
                    → completeWork 时调用 createTextInstance('Hello World', ...)
                    → 再通过 appendInitialChild 挂到父节点
```

### 11.3 shouldSetTextContent = true 的完整流程

以 celeste 的 `<p>Hello World</p>` 为例：

```
beginWork(p 的 Fiber):
  │
  ├── props.children = 'Hello World'
  ├── shouldSetTextContent('p', props) → true
  ├── nextChildren = null（不创建任何子 Fiber）
  └── return null（没有子节点要向下处理）

completeWork(p 的 Fiber):
  │
  ├── createInstance('p', { children: 'Hello World' })
  │   → TextElementImpl 创建
  │   → parseSpecProps: this.text = props.children  // 'Hello World'
  │   → updateSpecProps: this.updateText()
  │   → Native: lightEngineComponentInstance.setText('Hello World')
  │
  └── 完成（没有子实例需要 appendInitialChild）

最终宿主树:
  p (内部持有文字 'Hello World')
```

**文字是通过 `props.children` 传入，由组件在 `createInstance` / `updateProps` 中自行读取处理的。**

### 11.4 shouldSetTextContent = false 的完整流程

假设对 `<div>Hello World</div>`（celeste 中 div 返回 false）：

```
beginWork(div 的 Fiber):
  │
  ├── props.children = 'Hello World'
  ├── shouldSetTextContent('div', props) → false
  ├── reconcileChildren: 为 'Hello World' 创建子 Fiber
  │   → Fiber { tag: HostText, pendingProps: 'Hello World' }
  └── return textFiber（继续向下处理文本 Fiber）

beginWork(文本 Fiber):
  │
  └── HostText 类型，没有子节点

completeWork(文本 Fiber):
  │
  └── createTextInstance('Hello World', rootContainer, hostContext, fiber)
      → 返回 TextInstance 对象

completeWork(div 的 Fiber):
  │
  ├── createInstance('div', props) → divInstance
  ├── appendInitialChild(divInstance, textInstance)
  └── 完成

最终宿主树:
  div
  └── TextInstance('Hello World')  ← 独立的文本子节点
```

### 11.5 混合内容的情况

```jsx
<div>
  Hello
  <img src="icon.png" />
  World
</div>
```

`shouldSetTextContent('div', props)` → `false`（children 是数组，不是纯字符串）：

```
reconcileChildren 产出:
  Fiber { tag: HostText, pendingProps: 'Hello' }
  Fiber { tag: HostComponent, type: 'img' }
  Fiber { tag: HostText, pendingProps: 'World' }

completeWork 顺序:
  createTextInstance('Hello')          → textInstance1
  createInstance('img', { src: ... })  → imgInstance
  createTextInstance('World')          → textInstance2
  createInstance('div', props)         → divInstance
    appendInitialChild(div, textInstance1)
    appendInitialChild(div, imgInstance)
    appendInitialChild(div, textInstance2)

最终宿主树:
  div
  ├── TextInstance('Hello')
  ├── img
  └── TextInstance('World')
```

### 11.6 更新时的区别

| | shouldSetTextContent = true | shouldSetTextContent = false |
|--|--|--|
| **文字变了** | 父组件 props 变化 → `commitUpdate(parent)` → 内部重读 children | `commitTextUpdate(textInstance, old, new)` |
| **文字被删除** | 父组件 props.children 变了，内部处理 | `removeChild(parent, textInstance)` |
| **文字被新增** | props.children 从无到有，commitUpdate 处理 | `createTextInstance` + `appendChild` |

### 11.7 shouldSetTextContent 返回 true 的注意事项

当返回 `true` 时，**React 会跳过所有子节点的 reconcile**：

```javascript
// React 内部逻辑
if (shouldSetTextContent(type, nextProps)) {
  nextChildren = null;  // ← 所有 children 都被忽略！
}
reconcileChildren(current, workInProgress, nextChildren);
```

所以**不能对包含混合内容的组件返回 true**：

```jsx
// 如果 shouldSetTextContent('div') 返回 true
<div>
  Hello           ← 被 div 消化
  <img src="x" /> ← 也被忽略了！img 丢失！❌
</div>
```

只有那些**子内容一定是纯文字**的组件（如 `<p>`、`<button>`、`<textarea>`）才应该返回 `true`。

### 11.8 是否需要实现 createTextInstance

| 情况 | 是否需要实现 |
|------|------------|
| **类型声明层面** | 必须提供（HostConfig 接口要求） |
| **运行时层面** | 如果能保证所有文字都在 `shouldSetTextContent=true` 的组件内，则不会被执行 |
| **最佳实践** | 实现为 `throw Error` 作为防御性兜底（celeste 的做法） |

```typescript
// celeste 的做法：防御性实现
createTextInstance(text, ...): TextElementImpl {
  throw new Error(`The text '${text}' must be wrapped in a lgt-text OR lgt-text-view element.`);
}
// 好处：开发者犯错时（如 <div>text</div>）立刻报错，明确告知修复方式
```

### 11.9 不同平台的策略对比

| 平台 | shouldSetTextContent 策略 | createTextInstance |
|------|--------------------------|-------------------|
| **ReactDOM** | `<textarea>` 或 children 是纯字符串/数字时返回 true | 创建真实的 `document.createTextNode()` |
| **React Native** | `<Text>` 返回 true | 创建 RCTRawText 实例 |
| **celeste** | `<p>` 和 `<button>` 返回 true | throw Error（禁止裸文本） |
| **Canvas 渲染器** | 通常全部返回 false | 创建文字绘制对象 |

---

## 十二、Hydration 模式（补充，本项目未使用）

当 `supportsHydration: true` 时，额外需要实现：

```typescript
interface HydrationHostConfig {
  // 是否可以复用已有的 DOM 节点
  canHydrateInstance(instance: any, type: Type, props: Props): Instance | null;
  canHydrateTextInstance(instance: any, text: string): TextInstance | null;

  // 获取下一个兄弟/第一个子节点（用于遍历已有 DOM）
  getNextHydratableSibling(instance: Instance): Instance | null;
  getFirstHydratableChild(parentInstance: Instance): Instance | null;

  // 复用时对比属性差异
  hydrateInstance(instance: Instance, type: Type, props: Props, ...): UpdatePayload | null;
  hydrateTextInstance(textInstance: TextInstance, text: string, ...): boolean;

  // 清理无法复用的节点
  didNotMatchHydratedContainerTextInstance(...): void;
  didNotMatchHydratedTextInstance(...): void;
}
```

用于 SSR（服务端渲染）+ 客户端接管的场景，对现有 DOM 进行 "认领" 而非重新创建。

---

## 十二、Fiber 与时间切片（Time Slicing）

### 12.1 核心机制：两种 Work Loop

React 内部有两种工作循环，区别在于**是否检查时间片**：

```javascript
// ========== 同步模式（不可中断）==========
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

// ========== 并发模式（可中断）==========
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

唯一的差别就是 `&& !shouldYield()` —— 每处理完一个 Fiber 节点，就检查一次是否该让出。

### 12.2 `shouldYield()` 的判断逻辑

来自 React 的 Scheduler 包：

```javascript
// packages/scheduler/src/forks/Scheduler.js
function shouldYieldToHost() {
  const timeElapsed = getCurrentTime() - startTime;
  if (timeElapsed < frameInterval) {
    // 还在时间片内，继续工作
    return false;
  }
  // 超出时间片了，该让出
  return true;
}
```

- `frameInterval` 默认是 **5ms**（不是 16ms 一帧，是更短的切片）
- `startTime` 是本轮工作开始的时间戳
- 每处理完**一个 Fiber 单元**就判断一次

### 12.3 让出后的恢复流程

```
时间线:
───────────────────────────────────────────────────────────────────►

│← 5ms →│  让出  │← 浏览器工作 →│← 5ms →│  让出  │← 浏览器 →│...
│ Work   │ yield │ paint/input  │ Work   │ yield │          │
│ A→B→C  │       │              │ D→E→F  │       │          │

Fiber 处理: A → B → C  暂停... 恢复 → D → E → F  暂停... 恢复 → ...
```

让出后：
1. Scheduler 通过 **MessageChannel**（非 setTimeout）发一个消息
2. 浏览器执行更高优先级的任务（用户输入、绘制、动画等）
3. 下一个宏任务轮次，Scheduler 回调被触发，从**上次暂停的 Fiber** 继续

### 12.4 为什么 Fiber 链表结构能暂停/恢复

传统递归（Stack Reconciler）无法暂停，因为调用栈信息丢了：

```javascript
// 旧架构：递归，无法中断
function reconcile(element) {
  reconcile(element.child);     // ← 调用栈里，暂停就丢失上下文
  reconcile(element.sibling);
}
```

Fiber 把递归改成了**链表迭代**，状态保存在 Fiber 节点上：

```javascript
// Fiber 架构：迭代，随时可停
function performUnitOfWork(fiber) {
  // 1. 处理当前节点（beginWork）
  const next = beginWork(fiber);

  if (next !== null) {
    // 有子节点，下一个处理子节点
    workInProgress = next;
  } else {
    // 没有子节点，completeWork 并找兄弟/回溯父节点
    completeUnitOfWork(fiber);
  }
}
```

Fiber 节点的三个指针构成可遍历链表：

```
      ┌─────┐
      │  A  │ ← return (父)
      └──┬──┘
   child │
      ┌──▼──┐  sibling  ┌─────┐  sibling  ┌─────┐
      │  B  │ ─────────► │  C  │ ─────────► │  D  │
      └─────┘            └──┬──┘            └─────┘
                      child │
                         ┌──▼──┐
                         │  E  │
                         └─────┘

遍历顺序: A → B → (B完成) → C → E → (E完成) → (C完成) → D → (D完成) → (A完成)
暂停点:   ^    ^              ^    ^               任何箭头处都可以暂停
```

`workInProgress` 就是"书签" —— 记录当前处理到哪个 Fiber，下次恢复直接从这里继续。

### 12.5 什么时候用哪种模式

| 触发方式 | Work Loop | 可中断 |
|----------|-----------|--------|
| `ReactDOM.render()` (Legacy) | `workLoopSync` | ❌ 不可中断 |
| `ReactDOM.createRoot().render()` (React 18+) | `workLoopConcurrent` | ✅ 可中断 |
| `flushSync(() => setState(...))` | `workLoopSync` | ❌ 强制同步 |
| `startTransition(() => setState(...))` | `workLoopConcurrent` | ✅ 低优先级，更容易被中断 |

### 12.6 优先级抢占

不只是"超时让出"，更高优先级的更新可以**打断**当前工作：

```
用户点击按钮 (DiscreteEvent, 高优先级)
        │
        ▼
正在处理一个 Transition 更新 (低优先级)
        │
        ▼
Scheduler 检测到高优先级任务进来
        │
        ▼
中断当前 Render → 丢弃未完成的 work-in-progress 树
        │
        ▼
开始处理高优先级更新 → 完成后再回来处理低优先级
```

这就是为什么 Render 阶段**必须无副作用** —— 因为它可能被丢弃重来。

---

## 十三、协作式调度的固有限制 —— 单个 Fiber 可以超时

### 13.1 核心问题

`shouldYield()` 只在**两个 Fiber 单元之间**检查，不会在一个单元**执行中途**打断：

```javascript
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    //                                 ↑ 只在这里检查
    performUnitOfWork(workInProgress);
    //     ↑ 这个函数执行期间，无论多久都不会被打断
  }
}
```

如果单个 `performUnitOfWork` 耗时很长，时间片就被"撑破"了：

```
时间线:
│← 5ms →│
│ A  B   │ C（耗时 20ms 的组件）                    │ D ...
│────────│──────────────────────────────────────────│────
         ↑                                          ↑
    shouldYield() = false                  shouldYield() = true
    继续执行 C                              才发现超时，让出

实际这一轮跑了 25ms，远超 5ms 预算
```

### 13.2 协作式 vs 抢占式

```
┌──────────────────────────────────────────────────────────────┐
│  协作式调度（React 当前方案）                                   │
│                                                              │
│  - 任务必须"主动"让出（执行完一个单元后检查）                    │
│  - 单个单元内不可中断                                          │
│  - 如果某个单元耗时过长，没有任何机制可以强制打断                 │
│  - 类比：你在开会，只有说完一句话后才看手机有没有消息              │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  抢占式调度（操作系统线程调度）                                  │
│                                                              │
│  - 系统强制中断正在执行的任务（时钟中断）                        │
│  - 任何时刻都可以被打断                                        │
│  - JS 单线程环境中无法实现真正的抢占式                           │
│  - 类比：老师随时可以打断你的发言让别人说                         │
└──────────────────────────────────────────────────────────────┘
```

**JS 是单线程的**，不可能像 OS 那样通过时钟中断强制切换，React 只能靠"每个单元结束后自觉检查"。

### 13.3 哪些情况会导致单个 Fiber 超时

| 场景 | 说明 |
|------|------|
| 组件 render 函数很重 | 函数组件体内有大量计算（循环、递归、大数组操作） |
| 同步的外部库调用 | 在 render 中调用了耗时的同步 API |
| `beginWork` 中的 diff | 子节点数量巨大时（如 1000 个列表项的 key diff） |
| `completeWork` 中的 `createInstance` | HostConfig 的 `createInstance` 涉及重度 Native 调用 |
| `prepareUpdate` 做深对比 | 对比大对象的 props diff |

示例：

```jsx
// 这个组件的 render 本身就要 50ms
function HeavyComponent({ data }) {
  // data 有 10 万条，在 render 里做聚合
  const result = data.reduce((acc, item) => {
    return expensiveComputation(acc, item);
  }, initialValue);

  return <div>{result}</div>;
}
```

React 处理 `HeavyComponent` 这个 Fiber 时，执行它的函数体就要 50ms，**这期间不会中断**。

### 13.4 耗时分布示意

```
performUnitOfWork 耗时分布:

普通组件:     |█|  (< 1ms, 大部分组件都很快)
中等组件:     |███|  (1-3ms)
重型组件:     |████████████████████|  (可能 20ms+)
                                    ↑
                               这一整段内无法中断
                               shouldYield 在它结束后才生效

时间片:  |← 5ms →|
              ↑
         如果一个单元就超了 5ms
         那这一轮实际执行时间 > 5ms
         但 React 也只能等它跑完
```

### 13.5 开发者侧的优化手段

#### 1. 拆分重组件（增加可中断点）

```jsx
// ❌ 一个巨大组件 = 一个巨大的工作单元
function GiantList({ items }) {
  return <div>{items.map(item => /* 复杂渲染 */)}</div>;
}

// ✅ 拆成多个子组件 = 多个小工作单元，每个之间可以让出
function GiantList({ items }) {
  return <div>{items.map(item => <ListItem key={item.id} data={item} />)}</div>;
}
function ListItem({ data }) {
  return /* 单个 item 的渲染 */;
}
```

每个 `<ListItem>` 是独立的 Fiber 节点，处理完一个就有机会 `shouldYield()`。

#### 2. `useMemo` / `useCallback` 减少单元内计算量

```jsx
function Component({ data }) {
  // ✅ 重计算被缓存，不会每次 render 都跑
  const processed = useMemo(() => heavyCompute(data), [data]);
  return <div>{processed}</div>;
}
```

#### 3. `useTransition` / `useDeferredValue` 降优先级

```jsx
function Search({ query }) {
  const [isPending, startTransition] = useTransition();
  const [results, setResults] = useState([]);

  function handleChange(e) {
    // 输入响应是高优先级（同步）
    setQuery(e.target.value);

    // 搜索结果计算是低优先级（可中断、可丢弃）
    startTransition(() => {
      setResults(computeResults(e.target.value));
    });
  }
}
```

#### 4. 虚拟列表 / 分页

从根本上减少 Fiber 节点数量，每个时间片内要处理的工作就少。

### 13.6 总结表

| 问题 | 回答 |
|------|------|
| 单次遍历能超过 5ms 时间片吗？ | ✅ 完全可以 |
| React 能强制打断吗？ | ❌ 不能，JS 单线程无法抢占 |
| 这是 bug 吗？ | 不是，是协作式调度的固有限制 |
| 开发者能做什么？ | 拆分组件粒度、缓存计算、降优先级、减少节点数 |
| 有没有更好的方案？ | Web 平台有 `scheduler.postTask()` 提案及 Worker 线程方案，但目前都不成熟 |

---

## 十四、参考资源

- [React Reconciler 源码 - ReactFiberHostConfig](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHostConfig.js)
- [React Scheduler 源码](https://github.com/facebook/react/tree/main/packages/scheduler)
- [Building a Custom React Renderer](https://agent-hunt.medium.com/hello-world-custom-react-renderer-9a95b7cd04bc)
- [react-reconciler README](https://www.npmjs.com/package/react-reconciler)
- [Celeste LightEngineHostConfig](../pcad-canvas-aggregation/celeste/src/core/host-config/LightEngineHostConfig.ts)
- [scheduler.postTask() 提案](https://github.com/nicolo-ribaudo/tc39-proposal-atomics-microwait)
