# Claw-Code Ink 组件架构与终端 UI 渲染流程

## 整体架构

基于 fork/深度定制版 Ink 框架，将 React 的声明式组件模型映射到终端字符界面：

```
React 组件树 → 自定义 Reconciler → 虚拟 DOM (ink-box/ink-text) → Yoga 布局引擎 → Screen Buffer → ANSI 差量输出 → 终端
```

---

## 1. 组件层 (`src/ink/components/`)

### 基础组件

| 组件 | 对标浏览器 | 作用 |
|------|-----------|------|
| `<Box>` | `<div style="display:flex">` | Flexbox 布局容器，渲染为 `<ink-box>` |
| `<Text>` | `<span>` | 文本样式（颜色、粗体、下划线等），渲染为 `<ink-text>` |
| `<ScrollBox>` | `<div style="overflow:scroll">` | 可滚动容器，带 imperative scroll API |
| `<AlternateScreen>` | 全屏模式 | 切换终端 alternate screen buffer + 鼠标追踪 |
| `<Spacer>` | `<div style="flex:1">` | 弹性占位 |
| `<Newline>` | `<br>` | 换行 |
| `<Link>` | `<a>` | OSC 8 超链接 |

### Context 提供

| Context | 作用 |
|---------|------|
| `AppContext` | 提供 exit 方法 |
| `StdinContext` | 提供 stdin、setRawMode、eventEmitter |
| `TerminalSizeContext` | 提供 columns/rows |
| `TerminalFocusContext` | 提供终端焦点状态 |
| `ClockContext` | 提供时钟 tick（动画用） |
| `CursorDeclarationContext` | 声明光标位置（IME 输入用） |

---

## 2. 虚拟 DOM (`src/ink/dom.ts`)

### 节点类型

```typescript
export type ElementNames =
  | 'ink-root'
  | 'ink-box'
  | 'ink-text'
  | 'ink-virtual-text'   // Text 嵌套 Text 时的内部类型
  | 'ink-link'
  | 'ink-progress'
  | 'ink-raw-ansi'
```

### DOMElement 结构

```typescript
export type DOMElement = {
  nodeName: ElementNames
  attributes: Record<string, DOMNodeAttribute>
  childNodes: DOMNode[]
  textStyles?: TextStyles            // color, bold, italic...

  // 内部状态
  dirty: boolean                     // 需要重渲染
  isHidden?: boolean                 // display: none
  _eventHandlers?: Record<string, unknown>  // onClick, onKeyDown...

  // Scroll 状态 (overflow: scroll 的 box)
  scrollTop?: number
  pendingScrollDelta?: number        // 累积的未消费滚动量
  scrollHeight?: number              // 内容总高度
  scrollViewportHeight?: number      // 可见区域高度
  stickyScroll?: boolean             // 自动跟随底部

  // 布局引擎
  parentNode: DOMElement | undefined
  yogaNode?: LayoutNode              // Yoga 布局节点
  style: Styles                      // flexbox 样式
}

export type TextNode = {
  nodeName: '#text'
  nodeValue: string                  // 纯文本内容
  parentNode: DOMElement | undefined
  yogaNode: undefined                // 文本节点没有 yoga 节点
  style: {}
}
```

### 关键操作

```typescript
createNode(nodeName)        // 创建节点 + Yoga 节点
appendChildNode(parent, child)  // 挂载 + Yoga insertChild + markDirty
removeChildNode(parent, child)  // 卸载 + Yoga removeChild + markDirty
setAttribute(node, key, val)    // 属性变更 + markDirty
setStyle(node, style)           // 样式变更 + markDirty（有 shallow equal 优化）
setTextStyles(node, styles)     // 文本样式变更 + markDirty
setTextNodeValue(node, text)    // 更新文字 + markDirty
markDirty(node)                 // dirty 向上冒泡到 root
```

---

## 3. Yoga 布局引擎

### 架构分层

```
src/ink/layout/engine.ts          → createLayoutNode() 入口
src/ink/layout/yoga.ts            → YogaLayoutNode 适配器（LayoutNode 接口 → Yoga API）
src/ink/layout/node.ts            → LayoutNode 接口定义
src/native-ts/yoga-layout/index.ts → 2578 行纯 TS 实现的 Flexbox 引擎
```

### 纯 TypeScript 实现

**不是 WASM，不是 C++ binding**，是 Meta Yoga C++ 引擎的简化 TS 移植：

```typescript
/**
 * Pure-TypeScript port of yoga-layout (Meta's flexbox engine).
 * Simplified single-pass flexbox implementation covering:
 *   - flex-direction (row/column + reverse)
 *   - flex-grow / flex-shrink / flex-basis
 *   - align-items / align-self / justify-content
 *   - margin / padding / border / gap
 *   - width / height / min / max (point, percent, auto)
 *   - position: relative / absolute
 *   - display: flex / none
 *   - overflow: visible / hidden / scroll
 *   - measure functions (for text nodes)
 *   - flex-wrap / align-content
 */
```

### Yoga Node 数据结构

```typescript
class Node {
  style: Style       // 输入：通过 setter 设置的 flexbox 属性
  layout: Layout     // 输出：calculateLayout() 计算后的结果
  parent: Node | null
  children: Node[]
  measureFunc: MeasureFunction | null  // 叶节点（ink-text）的测量函数
  isDirty_: boolean
  
  // 缓存优化字段（多级缓存，避免重复计算）
  _flexBasis: number
  _hasL: boolean       // 是否有 layout 缓存
  _cN: number          // multi-entry cache 条目数
  ...
}
```

### Layout 输出

```typescript
type Layout = {
  left: number      // 相对于父节点的 x 偏移
  top: number       // 相对于父节点的 y 偏移
  width: number     // 节点最终宽度（字符列数）
  height: number    // 节点最终高度（行数）
  border: [number, number, number, number]   // left, top, right, bottom
  padding: [number, number, number, number]
  margin: [number, number, number, number]
}
```

### calculateLayout 核心算法

```typescript
function layoutNode(node, availW, availH, wMode, hMode, ownerW, ownerH, performLayout) {
  // ❶ 缓存命中 → 跳过
  if (!node.isDirty_ && cacheHit(...)) return

  // ❷ 叶节点 + measureFunc（ink-text）
  if (node.measureFunc && node.children.length === 0) {
    const measured = node.measureFunc(innerW, wMode)
    node.layout.width = clamp(measured.width + paddingBorder, min, max)
    node.layout.height = clamp(measured.height + paddingBorder, min, max)
    return
  }

  // ❸ 容器节点 — Flexbox 算法
  // STEP 1: computeFlexBasis — 每个子节点的自然尺寸
  // STEP 2: resolveFlexibleLengths — 按 grow/shrink 分配剩余/超出空间
  // STEP 3: 确定容器自身尺寸
  // STEP 4: 定位子节点（justify-content + align-items）
  // STEP 5: 处理 position: absolute 子节点
}
```

---

## 4. 自定义 React Reconciler (`src/ink/reconciler.ts`)

基于 `react-reconciler` 库创建：

```typescript
const reconciler = createReconciler({
  // ─── 创建 ───
  createInstance(type, props, root, hostContext, fiber) {
    const node = createNode(type)           // 创建 DOMElement + Yoga.Node
    for (const [key, value] of Object.entries(props)) {
      applyProp(node, key, value)           // 设置 style/attributes/textStyles
    }
    return node
  },
  
  createTextInstance(text, root, hostContext) {
    return createTextNode(text)             // { nodeName: '#text', nodeValue: text }
  },

  // ─── 挂载 ───
  appendInitialChild: appendChildNode,      // parent.childNodes.push + yoga.insertChild
  appendChild: appendChildNode,
  insertBefore: insertBeforeNode,

  // ─── 更新 ───
  commitUpdate(node, type, oldProps, newProps) {
    const props = diff(oldProps, newProps)
    // 对变化的 props 调用 setStyle/setTextStyles/setAttribute
    if (style && node.yogaNode) applyStyles(node.yogaNode, style, newProps.style)
  },
  
  commitTextUpdate(node, oldText, newText) {
    setTextNodeValue(node, newText)         // nodeValue = newText + markDirty
  },

  // ─── 卸载 ───
  removeChild(node, removeNode) {
    removeChildNode(node, removeNode)       // 从 DOM 和 Yoga 树移除
    cleanupYogaNode(removeNode)             // 释放 Yoga 节点内存
  },

  // ─── 提交完成 ───
  resetAfterCommit(rootNode) {
    rootNode.onComputeLayout()              // ← Yoga calculateLayout
    rootNode.onRender()                     // ← 触发渲染
  },

  // ─── 焦点 ───
  finalizeInitialChildren(node, type, props) {
    return props.autoFocus === true
  },
  commitMount(node) {
    getFocusManager(node).handleAutoFocus(node)
  },

  // ─── 显示/隐藏 ───
  hideInstance(node) {
    node.isHidden = true
    node.yogaNode?.setDisplay(LayoutDisplay.None)
    markDirty(node)
  },
  unhideInstance(node) {
    node.isHidden = false
    node.yogaNode?.setDisplay(LayoutDisplay.Flex)
    markDirty(node)
  },
})
```

### `applyProp` 分发逻辑

```typescript
function applyProp(node, key, value) {
  if (key === 'children') return
  if (key === 'style') {
    setStyle(node, value)                   // 存到 DOM 节点
    applyStyles(node.yogaNode, value)       // 写入 Yoga 节点的 style
    return
  }
  if (key === 'textStyles') {
    node.textStyles = value                 // 终端文本样式
    return
  }
  if (EVENT_HANDLER_PROPS.has(key)) {
    setEventHandler(node, key, value)       // onClick/onKeyDown 等
    return
  }
  setAttribute(node, key, value)            // 其他属性
}
```

---

## 5. Ink 主类 (`src/ink/ink.tsx`)

### 初始化

```typescript
class Ink {
  constructor(options) {
    // 终端尺寸
    this.terminalColumns = options.stdout.columns || 80
    this.terminalRows = options.stdout.rows || 24
    
    // 渲染管线
    this.rootNode = dom.createNode('ink-root')
    this.renderer = createRenderer(this.rootNode, this.stylePool)
    
    // 注册回调
    this.rootNode.onComputeLayout = () => {
      this.rootNode.yogaNode.setWidth(this.terminalColumns)
      this.rootNode.yogaNode.calculateLayout(this.terminalColumns)
    }
    this.rootNode.onRender = this.scheduleRender  // throttled
    
    // 监听 resize
    options.stdout.on('resize', this.handleResize)
    
    // 创建 React container
    this.container = reconciler.createContainer(this.rootNode, ConcurrentRoot, ...)
  }
}
```

### 终端宽高获取

```typescript
// 来源：Node.js 的 stdout API
// 底层：操作系统 ioctl(TIOCGWINSZ) 系统调用
this.terminalColumns = options.stdout.columns || 80
this.terminalRows = options.stdout.rows || 24

// 监听变化：内核 SIGWINCH → Node.js 更新 → 'resize' 事件
private handleResize = () => {
  const cols = this.options.stdout.columns || 80
  const rows = this.options.stdout.rows || 24
  if (cols === this.terminalColumns && rows === this.terminalRows) return
  this.terminalColumns = cols
  this.terminalRows = rows
  this.render(this.currentNode)  // 触发 React 重新渲染
}
```

### render 入口

```typescript
render(node: ReactNode): void {
  const tree = <App
    terminalColumns={this.terminalColumns}
    terminalRows={this.terminalRows}
    ...
  >
    {node}
  </App>
  
  reconciler.updateContainerSync(tree, this.container, null, noop)
  reconciler.flushSyncWork()
}
```

### onRender — 帧渲染

```typescript
onRender() {
  // ① 调用 renderer 生成 Frame (包含 Screen buffer)
  const frame = this.renderer({
    frontFrame: this.frontFrame,    // 上一帧
    backFrame: this.backFrame,      // 空白缓冲
    terminalWidth, terminalRows,
    altScreen: this.altScreenActive,
    prevFrameContaminated: this.prevFrameContaminated,
  })
  
  // ② 计算 diff（对比新旧 Screen）
  const diff = this.log.render(prevFrame, frame, ...)
  
  // ③ 优化（合并相邻写入）
  const optimized = optimize(diff)
  
  // ④ 写到终端
  writeDiffToTerminal(this.terminal, optimized)
  
  // ⑤ swap buffers
  this.backFrame = this.frontFrame
  this.frontFrame = frame
}
```

---

## 6. Renderer (`src/ink/renderer.ts`)

```typescript
export default function createRenderer(node, stylePool): Renderer {
  let output: Output | undefined
  
  return (options) => {
    const width = Math.floor(node.yogaNode.getComputedWidth())
    const height = Math.floor(node.yogaNode.getComputedHeight())
    const screen = options.backFrame.screen
    
    output.reset(width, height, screen)
    
    // 核心：遍历虚拟 DOM 树，写入 Screen buffer
    renderNodeToOutput(node, output, {
      prevScreen: options.prevFrameContaminated ? undefined : prevScreen
    })
    
    const renderedScreen = output.get()
    
    return {
      screen: renderedScreen,
      viewport: { width: terminalWidth, height: terminalRows },
      cursor: { x: 0, y: screen.height, visible: !isTTY || height === 0 },
    }
  }
}
```

---

## 7. render-node-to-output (`src/ink/render-node-to-output.ts`)

遍历虚拟 DOM 树，读取 Yoga 布局结果，写入 Output/Screen：

```typescript
function renderNode(node, offsetX, offsetY) {
  const yogaNode = node.yogaNode
  const x = offsetX + yogaNode.getComputedLeft()   // 累加父偏移得到绝对坐标
  const y = offsetY + yogaNode.getComputedTop()
  const width = yogaNode.getComputedWidth()
  const height = yogaNode.getComputedHeight()

  // ── Blit 优化：clean + 位置未变 → 直接从上一帧拷贝 ──
  if (!node.dirty && cached && samePosition && prevScreen) {
    output.blit(prevScreen, x, y, width, height)
    return
  }

  // ── ink-text: 文本渲染 ──
  if (node.nodeName === 'ink-text') {
    const segments = squashTextNodesToSegments(node)    // 收集子文本
    const text = applyTextStyles(segments)              // 应用 ANSI 样式
    const wrapped = wrapText(text, maxWidth)            // 换行
    output.write(x, y, wrapped)
  }
  
  // ── ink-box: 容器递归 ──
  else if (node.nodeName === 'ink-box') {
    const padLeft = yogaNode.getComputedPadding('left')
    const padTop = yogaNode.getComputedPadding('top')
    
    // overflow: scroll 处理
    if (style.overflowY === 'scroll') {
      // viewport culling — 只渲染可见区域子节点
      for (child of children) {
        if (childTop + childHeight < scrollTop) continue
        if (childTop > scrollTop + viewportHeight) continue
        renderNode(child, contentX, contentY - scrollTop)
      }
    } else {
      for (child of children) {
        renderNode(child, x + padLeft, y + padTop)
      }
    }
    
    // 画 border
    if (node.style.borderStyle) renderBorder(x, y, width, height)
  }
}
```

---

## 8. Screen Buffer 与 Diff

### Screen 结构 (`src/ink/screen.ts`)

```typescript
// 字符格网 rows × cols
// 每个 cell 存储：
{
  char: string           // 字符内容
  width: CellWidth       // 1(半角) 或 2(全角)
  styleId: number        // StylePool 中 intern 的样式 ID
  hyperlink: number      // HyperlinkPool 中 intern 的链接 ID
}
```

### Output 操作 (`src/ink/output.ts`)

```typescript
class Output {
  write(x, y, text, softWrap?)   // 写入文本
  blit(prevScreen, x, y, w, h)   // 从上一帧拷贝区域
  clear(rect, isAbsolute)        // 清除区域
  noSelect(rect)                 // 标记不可选中
  get(): Screen                  // 应用所有操作到 Screen buffer
}
```

### Diff + 终端输出 (`src/ink/log-update.ts`)

逐 cell 对比当前帧与上一帧的 Screen：
- 只对变化的 cell 生成 ANSI 转义序列
- cursor move + set style + write char
- 支持 DECSTBM 硬件滚动优化

---

## 9. 输入处理 (`App.tsx`)

```
stdin raw mode → stdin 'readable' 事件 → 读取字节
  → parse-keypress.ts 解析：
    - 普通按键: { key: 'a', ctrl: false, ... }
    - 特殊键: { key: 'return', name: 'return', ... }
    - 鼠标 SGR: { type: 'mouse', button: 0, x: 5, y: 10, ... }
    - 粘贴: { type: 'paste', data: '...' }
  → EventEmitter 广播（legacy path）
  → DOM 事件分发（capture/bubble，类似 W3C）
    → dispatchKeyboardEvent(parsedKey)  // 键盘事件
    → onClickAt(col, row)              // 鼠标点击
    → onHoverAt(col, row)              // 鼠标悬停
```

### 终端协议支持

| 能力 | 转义序列 |
|------|---------|
| 隐藏光标 | DECTCEM `\x1b[?25l` |
| 备用屏幕 | DEC 1049 `\x1b[?1049h/l` |
| 鼠标追踪 | SGR 1006 模式 |
| 括号粘贴 | DEC 2004 `\x1b[?2004h/l` |
| 焦点报告 | DEC 1004 `\x1b[?1004h/l` |
| 扩展键盘 | Kitty keyboard protocol + modifyOtherKeys |
| 超链接 | OSC 8 `\x1b]8;;url\x07text\x1b]8;;\x07` |
| 终端探测 | XTVERSION 查询 |
| 剪贴板 | OSC 52 |

---

## 10. 完整渲染流程实例

### 假设渲染如下 JSX：

```tsx
<Box flexDirection="column" paddingLeft={1}>
  <Text bold>user:</Text>
  <Text color="ansi:cyan"> hello world</Text>
</Box>
```

### 完整调用链

```
ink.render(userJSX)
  │
  ▼
reconciler.updateContainerSync(tree, container)
  │
  ├── [Render Phase] ─────────────────────────────────────────
  │   createInstance('ink-box', { style: { flexDirection:'column', paddingLeft:1, ... } })
  │     → createNode('ink-box')         → 空 DOMElement + Yoga.Node.create()
  │     → applyProp('style', {...})     → yogaNode.setFlexDirection('column')
  │                                       yogaNode.setPadding('left', 1)
  │                                       yogaNode.setFlexShrink(1)
  │   createInstance('ink-text', { style: {...}, textStyles: { bold: true } })
  │     → createNode('ink-text')        → DOMElement + Yoga.Node.create()
  │     → yogaNode.setMeasureFunc(measureTextNode.bind(null, node))
  │     → node.textStyles = { bold: true }
  │   createTextInstance("user:")
  │     → { nodeName:'#text', nodeValue:'user:', yogaNode: undefined }
  │   appendInitialChild(inkText, textNode)
  │     → inkText.childNodes.push(textNode)
  │     → markDirty(inkText)
  │   appendInitialChild(inkBox, inkText)
  │     → inkBox.childNodes.push(inkText)
  │     → inkBox.yogaNode.insertChild(inkText.yogaNode, 0)  // Yoga 树成型
  │     → markDirty(inkBox)
  │
  ├── [Commit Phase] ─────────────────────────────────────────
  │   resetAfterCommit(rootNode)
  │     │
  │     ├── rootNode.onComputeLayout()
  │     │     yogaRoot.setWidth(80)           // 终端宽度
  │     │     yogaRoot.calculateLayout(80)    // ← Yoga 递归计算
  │     │       │
  │     │       └── layoutNode(root, 80, NaN, Exactly, Undefined)
  │     │             └── layoutNode(inkBox, 80, NaN, ...)
  │     │                   │ innerWidth = 80 - paddingLeft(1) = 79
  │     │                   ├── layoutNode(inkText0, 79, NaN, AtMost, ...)
  │     │                   │     → measureFunc("user:") → { w:5, h:1 }
  │     │                   │     → layout = { left:0, top:0, w:5, h:1 }
  │     │                   ├── layoutNode(inkText1, 79, NaN, AtMost, ...)
  │     │                   │     → measureFunc(" hello world") → { w:11, h:1 }
  │     │                   │     → layout = { left:0, top:1, w:11, h:1 }
  │     │                   └── inkBox.layout = { left:0, top:0, w:80, h:2 }
  │     │
  │     │     计算结果（每个 Node.layout）:
  │     │       root:      { left:0, top:0, width:80, height:2 }
  │     │       inkBox:    { left:0, top:0, width:80, height:2, padding:[1,0,0,0] }
  │     │       inkText0:  { left:0, top:0, width:5,  height:1 }
  │     │       inkText1:  { left:0, top:1, width:11, height:1 }
  │     │
  │     └── rootNode.onRender() → scheduleRender (throttled)
  │           → queueMicrotask(this.onRender)
  │
  ├── [Layout Effects] ──────────────────────────────────────
  │   useLayoutEffect 可读取 yogaNode.getComputedXxx()
  │
  └── [Microtask] ───────────────────────────────────────────
      this.onRender()
        │
        ├── renderer({ frontFrame, backFrame, ... })
        │     │
        │     ├── renderNodeToOutput(root, output, { prevScreen })
        │     │     renderNode(root, offsetX=0, offsetY=0)
        │     │       renderNode(inkBox, offsetX=0, offsetY=0)
        │     │         │ x=0, y=0, padLeft=1
        │     │         │
        │     │         renderNode(inkText0, offsetX=1, offsetY=0)
        │     │           x = 1 + 0 = 1, y = 0 + 0 = 0
        │     │           squashTextNodes → "user:"
        │     │           applyTextStyles("user:", { bold:true }) → "\x1b[1muser:\x1b[22m"
        │     │           output.write(1, 0, "\x1b[1muser:\x1b[22m")
        │     │         
        │     │         renderNode(inkText1, offsetX=1, offsetY=0)
        │     │           x = 1 + 0 = 1, y = 0 + 1 = 1
        │     │           squashTextNodes → " hello world"
        │     │           applyTextStyles → "\x1b[36m hello world\x1b[39m"
        │     │           output.write(1, 1, "\x1b[36m hello world\x1b[39m")
        │     │
        │     └── output.get() → Screen buffer:
        │           Row 0: [ ][u][s][e][r][:][  ]...  (bold, col 1~5)
        │           Row 1: [ ][ ][h][e][l][l][o][ ][w][o][r][l][d]...  (cyan, col 1~12)
        │
        ├── this.log.render(prevFrame, newFrame)
        │     逐 cell 对比 → 生成 diff patches
        │
        ├── optimize(diff) → 合并相邻写入
        │
        └── stdout.write(patches):
              \x1b[1;2H\x1b[1muser:\x1b[22m     ← row1 col2, bold
              \x1b[2;2H\x1b[36m hello world\x1b[39m  ← row2 col2, cyan
```

### 终端显示结果

```
 user:              ← bold, paddingLeft=1 产生前导空格
  hello world       ← cyan
```

---

## 11. 增量更新流程

当文本从 `"hello world"` 变成 `"hello world!"` 时：

```
React setState → reconciler.commitTextUpdate(textNode, old, new)
  → setTextNodeValue(node, "hello world!")
    → node.nodeValue = "hello world!"
    → markDirty(parent ink-text)
      → ink-text.yogaNode.markDirty()  // measureFunc 需要重新测量
      → dirty 冒泡到 root

  → resetAfterCommit(root)
    → onComputeLayout → calculateLayout(80)
      → root/inkBox 命中缓存（clean + 同输入）→ 跳过
      → inkText1 dirty → 重调 measureFunc
        → "hello world!" → { w:12, h:1 }
      → inkText1.layout.width = 12 (多了 1 列)
    
    → onRender → microtask
      → renderNodeToOutput: inkText1.dirty → 重新 write
      → Screen diff: 只有 row 1 col 13 有变化（新增 '!'）
      → stdout.write("\x1b[2;14H!")  ← 只写 1 个字符！
```

---

## 12. 性能优化要点

| 层级 | 优化 | 效果 |
|------|------|------|
| Yoga | dirty flag + multi-entry cache | 增量更新只重算脏子树 |
| Yoga | measureFunc 结果缓存 | 文本未变不重新测量 |
| render-node-to-output | blit 优化 | clean + 同位置 → 直接从 prevScreen 拷贝 |
| render-node-to-output | viewport culling | ScrollBox 只渲染可见区域 |
| Screen diff | cell-level diff | 只生成变化 cell 的 ANSI |
| Screen diff | damage rect | 限制 diff 扫描范围 |
| log-update | DECSTBM 硬件滚动 | 滚动时用终端硬件移动行 |
| onRender | throttle 16ms | 高频事件合并为一帧 |
| ScrollBox | 绕过 React | scrollBy 直接改 DOM + microtask 合并 |

---

## 13. useLayoutEffect 与光标声明机制

### 背景：终端物理光标

终端里那个闪烁的竖线/方块就是物理光标。位置通过 ANSI 转义序列控制：

```typescript
process.stdout.write('\x1b[10;5H')  // 移到 row 10, col 5
```

光标位置的意义（即使 Ink 平时把光标隐藏了）：
- **IME 输入法**在光标位置弹出候选框
- **屏幕放大镜/读屏器**跟随光标位置

### 光标声明（Cursor Declaration）

组件通过 `useDeclaredCursor` hook **声明式地**告诉 Ink 物理光标应该停在哪：

```typescript
// src/ink/hooks/use-declared-cursor.ts
export function useDeclaredCursor({ line, column, active }) {
  const setCursorDeclaration = useContext(CursorDeclarationContext)
  const nodeRef = useRef<DOMElement | null>(null)

  useLayoutEffect(() => {
    const node = nodeRef.current
    if (active && node) {
      setCursorDeclaration({ relativeX: column, relativeY: line, node })
    } else {
      setCursorDeclaration(null, node)  // 条件清除，不覆盖其他组件的声明
    }
  })

  return setNode  // ref callback，attach 到包含输入框的 Box
}
```

使用示例（`BaseTextInput.tsx`）：

```typescript
const cursorRef = useDeclaredCursor({
  line: 0,
  column: cursorOffset,  // 比如已输入 "hello" → column=5
  active: isFocused,
})
return <Box ref={cursorRef}>...</Box>
```

### CursorDeclaration 数据结构

```typescript
type CursorDeclaration = {
  relativeX: number       // 光标在声明节点内的列偏移
  relativeY: number       // 光标在声明节点内的行偏移
  node: DOMElement        // 声明节点（其 Yoga 布局提供绝对坐标原点）
}
```

### onRender 中如何落地

```typescript
// ink.tsx → onRender() 末尾
const decl = this.cursorDeclaration
const rect = nodeCache.get(decl.node)  // 该节点在屏幕上的绝对位置

const target = {
  x: rect.x + decl.relativeX,   // 绝对列 = 节点左上角 + 相对偏移
  y: rect.y + decl.relativeY,   // 绝对行
}

// 输出 ANSI 移动光标
optimized.push({
  type: 'stdout',
  content: cursorPosition(target.y + 1, target.x + 1)  // ANSI 1-indexed
})
```

### 为什么必须用 useLayoutEffect

```
commit phase:
  resetAfterCommit
    ├── onComputeLayout()  → Yoga 算完布局
    └── queueMicrotask(onRender)  → 放入队列，不立即执行
  ↓
  useLayoutEffect 执行  ← 此时 Yoga 已算完，且在 onRender 之前
    → setCursorDeclaration({ relativeX: 5, relativeY: 0, node })
  ↓
  microtask: onRender()  ← 读到 fresh 的 cursorDeclaration，光标位置正确
```

如果用 `useEffect`（在 onRender 之后异步执行）：
- onRender 先执行 → 读到的是旧的光标位置 → 这一帧光标画在错误位置
- useEffect 随后更新声明 → 下一帧才修正
- 用户看到：**光标延迟一次按键才跟到正确位置**（IME 候选框闪跳）

### 终端操作的本质

所有终端操作都是"往 stdout 写特定字节序列"，没有 API 调用：

```typescript
// src/ink/termio/csi.ts
export function cursorPosition(row: number, col: number): string {
  return `\x1b[${row};${col}H`
}

// 其他常用操作
'\x1b[?25l'      // 隐藏光标
'\x1b[?25h'      // 显示光标
'\x1b[38;2;r;g;bm'  // 设置 24-bit 前景色
'\x1b[1m'        // 粗体
'\x1b[0m'        // 重置样式
'\x1b[2J'        // 清屏
'\x1b[?1049h'    // 进入备用屏幕
```

---

## 14. 动画机制（Spinner / Glimmer）

### 架构：共享时钟 + React re-render

```
Clock (setInterval 16ms)
  → 每 tick 通知所有 subscribers
    → useAnimationFrame(50ms) 检查间隔 ≥50ms?
      → setTime(now) → 触发 React re-render
        → 组件用 time 值计算当前动画帧
```

### Clock（`src/ink/components/ClockContext.tsx`）

全局单例 setInterval，所有动画共享：

```typescript
function createClock(tickIntervalMs) {
  const subscribers = new Map()
  let interval = setInterval(tick, tickIntervalMs)  // 16ms
  let tickTime = 0

  function tick() {
    tickTime = Date.now() - startTime
    for (const onChange of subscribers.keys()) onChange()
  }

  return {
    subscribe(onChange, keepAlive) { ... },
    now() { return tickTime },  // 同一 tick 内所有人看到同一个值
    setTickInterval(ms) { ... },
  }
}
```

优化：
- 终端失焦时降为 32ms（省 CPU）
- 无 keepAlive 订阅者时停止 interval

### useAnimationFrame（`src/ink/hooks/use-animation-frame.ts`）

```typescript
export function useAnimationFrame(intervalMs = 16) {
  const clock = useContext(ClockContext)
  const [viewportRef, { isVisible }] = useTerminalViewport()
  const [time, setTime] = useState(() => clock.now())

  useEffect(() => {
    if (!clock || !isVisible || intervalMs === null) return
    let lastUpdate = clock.now()
    const onChange = () => {
      const now = clock.now()
      if (now - lastUpdate >= intervalMs) {
        lastUpdate = now
        setTime(now)  // 触发 re-render
      }
    }
    return clock.subscribe(onChange, true)  // keepAlive: true → 驱动 clock
  }, [clock, intervalMs, isVisible])

  return [viewportRef, time]
}
```

关键设计：
- `isVisible = false`（滚出视口）→ 不订阅 → 动画暂停 → 零消耗
- `intervalMs = null` → 主动暂停（如 stalled 状态）

### 图标旋转（`SpinnerGlyph.tsx`）

```typescript
const SPINNER_FRAMES = ['·', '✢', '✳', '✶', '✻', '✽', '✻', '✶', '✳', '✢', '·']

// 每 80ms 换一个字符
const spinnerChar = SPINNER_FRAMES[Math.floor(time / 80) % SPINNER_FRAMES.length]
return <Text color={messageColor}>{spinnerChar}</Text>
```

### 文字 Glimmer 效果（`GlimmerMessage.tsx`）

一个 3 字符宽的亮色窗口从左滑到右：

```typescript
// 高亮窗口位置：每 50ms 移一格
const glimmerIndex = (Math.floor(time / 50) % cycleLength) - 10

// 切分文字成三段
const shimmerStart = glimmerIndex - 1
const shimmerEnd = glimmerIndex + 1
before = 高亮左侧字符
shim = 高亮窗口内字符（3 字符宽）
after = 高亮右侧字符

// 三段不同颜色
<Text color={messageColor}>{before}</Text>   // 暗色
<Text color={shimmerColor}>{shim}</Text>     // 亮色
<Text color={messageColor}>{after}</Text>    // 暗色
```

视觉效果：

```
T h i n k i n g . . .
□ □ □ □ □ □ □ □ □ □ □   全暗
■ ■ ■ □ □ □ □ □ □ □ □   亮色窗口在左
□ ■ ■ ■ □ □ □ □ □ □ □   向右滑动
□ □ ■ ■ ■ □ □ □ □ □ □
□ □ □ □ □ □ □ □ ■ ■ ■   滑到右边
□ □ □ □ □ □ □ □ □ □ □   滑出，间隔后从左重新进入
```

### 两个动画共用同一次 render

```typescript
const [ref, time] = useAnimationFrame(50)  // 50ms 触发一次 re-render

// 在同一次 render 里，用同一个 time 算不同的东西：
const frame = Math.floor(time / 80)           // 图标：每 80ms 换帧
const glimmerIndex = Math.floor(time / 50)    // Glimmer：每 50ms 移一格
```

时间线：
```
time:   0    50   100   150   200   250
图标:   0    0    1     1     2     3     ← 每 80ms 变一次
glimmer: 0   1    2     3     4     5     ← 每 50ms 变一次
render:  ✓    ✓    ✓     ✓     ✓     ✓    ← 每 50ms render 一次
```

### 颜色控制

使用 24-bit true color（RGB 精确控制）：

```typescript
// utils.ts
interpolateColor({ r:153, g:153, b:153 }, { r:185, g:185, b:185 }, t)
toRGBColor(color)  // → "rgb(153,153,153)"

// 写到终端时变成：
// \x1b[38;2;153;153;153m  (24-bit 前景色)
```

---

## 15. 关键设计决策

| 决策 | 原因 |
|------|------|
| Yoga 用纯 TS 重写 | 无 WASM 加载延迟，同步可用，可深度优化缓存 |
| onComputeLayout 在 commit 中同步 | useLayoutEffect 需要读 fresh 布局值 |
| onRender 用 microtask 延迟 | 等 layout effects（光标位置声明）执行完 |
| Screen 用 cell 数组而非字符串 | 支持 cell-level diff，全角字符宽度处理 |
| 双缓冲 (frontFrame/backFrame) | diff 需要对比前后两帧 |
| styleId intern 到 StylePool | 减少 cell 存储体积，加速比较 |
| ScrollBox 绕过 React state | 滚轮每帧 60+ 事件，不能走 reconciler |
