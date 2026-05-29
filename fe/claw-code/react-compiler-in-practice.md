# React Compiler 实践：从 Claude Code 到前端项目

> Claude Code 是目前已知最大规模使用 React Compiler 的生产项目（395 个文件全量启用），本文记录其原理、在 Claude Code 中的应用，以及如何在实际前端项目中接入。

---

## 目录

1. [React Compiler 是什么](#react-compiler-是什么)
2. [Claude Code 中的使用方式](#claude-code-中的使用方式)
3. [为什么终端 UI 场景收益更大](#为什么终端-ui-场景收益更大)
4. [前端项目接入指南](#前端项目接入指南)
5. [验证与渐进式引入](#验证与渐进式引入)
6. [注意事项](#注意事项)

---

## React Compiler 是什么

React Compiler（之前叫 React Forget）是 React 团队开发的**编译时优化工具**。它在构建阶段自动分析组件代码，插入细粒度的 memoization 逻辑，替代手动编写的 `useMemo`、`useCallback`、`React.memo`。

### 核心思想

```
以前: 开发者手动决定"哪里需要缓存" → 容易遗漏或过度优化
现在: 编译器分析数据依赖 → 自动在所有该缓存的地方插入缓存逻辑
```

### 编译前后对比

**开发者写的源码（不需要任何手动优化）：**

```tsx
function TodoList({ todos, filter }) {
  const [search, setSearch] = useState('')

  const filteredTodos = todos
    .filter(t => t.status === filter)
    .filter(t => t.title.includes(search))

  const handleSearch = (e) => {
    setSearch(e.target.value)
  }

  return (
    <div>
      <input value={search} onChange={handleSearch} />
      {filteredTodos.map(todo => (
        <TodoItem key={todo.id} todo={todo} />
      ))}
    </div>
  )
}
```

**编译器自动生成的输出：**

```javascript
import { c as _c } from "react/compiler-runtime";

function TodoList(t0) {
  const $ = _c(12);  // 分配 12 槽的缓存数组

  // 自动缓存 props 解构
  let filter, todos;
  if ($[0] !== t0) {
    ({ todos, filter } = t0);
    $[0] = t0; $[1] = todos; $[2] = filter;
  } else {
    todos = $[1]; filter = $[2];
  }

  const [search, setSearch] = useState('');

  // 自动缓存计算结果（等价于 useMemo(() => ..., [todos, filter, search])）
  let filteredTodos;
  if ($[3] !== todos || $[4] !== filter || $[5] !== search) {
    filteredTodos = todos
      .filter(t => t.status === filter)
      .filter(t => t.title.includes(search));
    $[3] = todos; $[4] = filter; $[5] = search; $[6] = filteredTodos;
  } else {
    filteredTodos = $[6];
  }

  // 自动缓存回调（等价于 useCallback）
  let handleSearch;
  if ($[7] !== setSearch) {
    handleSearch = (e) => setSearch(e.target.value);
    $[7] = setSearch; $[8] = handleSearch;
  } else {
    handleSearch = $[8];
  }

  // 自动缓存 JSX（等价于 React.memo 的效果）
  // ...
}
```

### `_c(n)` 运行时机制

```javascript
import { c as _c } from "react/compiler-runtime";
const $ = _c(30);
```

`_c(n)` 的作用：
- 分配一个**长度为 n 的持久化数组**（挂在组件的 fiber 上，跨渲染保持）
- 每个槽位存一个缓存值
- 每次重渲染时通过 `$[i] !== newValue` 判断是否需要重新计算

```
┌──────────────────────────────────────────────────────────────────┐
│  $[0]  = props 对象（引用对比）                                   │
│  $[1]  = 解构出的 prop a                                         │
│  $[2]  = 解构出的 prop b                                         │
│  $[3]  = 依赖项 1（用于判断是否重算）                              │
│  $[4]  = 依赖项 2                                                │
│  $[5]  = 缓存的计算结果                                           │
│  $[6]  = 缓存的回调函数                                           │
│  ...                                                             │
│  $[n-1] = 缓存的 JSX 子树                                        │
│                                                                  │
│  ↑ 等价于 n 个 useMemo/useCallback，但编译器自动决定位置和依赖    │
└──────────────────────────────────────────────────────────────────┘
```

---

## Claude Code 中的使用方式

### 规模

```
395 个 .tsx/.ts 文件导入了 react/compiler-runtime
552 个 .tsx 文件（组件文件）
= 几乎所有 React 组件都经过了 Compiler 优化
```

### 实际编译产物示例

来自 `src/ink/components/Button.tsx` 的编译输出：

```tsx
import { c as _c } from "react/compiler-runtime";

function Button(t0) {
  const $ = _c(30);  // Button 组件需要 30 个缓存槽

  let autoFocus, children, onAction, ref, style, t1;
  if ($[0] !== t0) {
    // props 变了 → 重新解构
    ({ onAction, tabIndex: t1, autoFocus, children, ref, ...style } = t0);
    $[0] = t0;
    $[1] = autoFocus; $[2] = children; $[3] = onAction;
    $[4] = ref; $[5] = style; $[6] = t1;
  } else {
    // props 没变 → 用缓存
    autoFocus = $[1]; children = $[2]; onAction = $[3];
    ref = $[4]; style = $[5]; t1 = $[6];
  }

  // ... 后续所有 useState, useCallback, 事件处理器都自动缓存
}
```

### 源码中还有 useMemo/useCallback 吗？

有少量存在（如 `useTerminalNotification.ts` 中），但这些是**与编译器共存**的——编译器不会重复优化已手动优化的代码，两者不冲突。

### 构建方式推断

Claude Code 使用 Bun 构建。从编译产物看，React Compiler 作为 Babel 插件在 Bun 的构建流程中运行：

```
源码 (.tsx) → babel-plugin-react-compiler → 编译产物 (.tsx with _c) → bun build --compile → 二进制
```

---

## 为什么终端 UI 场景收益更大

```
浏览器 DOM 更新:
  React diff → 改几个 DOM 属性 → 浏览器增量重绘
  代价: 低（浏览器优化了几十年）

终端 Ink 更新:
  React diff → Yoga Flexbox 重算 → 生成完整 ANSI 字符串 → write() 到 stdout
  代价: 远高于 DOM！
  • Yoga 布局计算是同步 CPU 密集操作
  • 终端没有"增量重绘"——每次都是全量字符串重写
  • stdout.write() 有 syscall 开销
```

所以避免不必要的重渲染在终端场景比浏览器更关键：

| | 浏览器 | 终端 (Ink) |
|---|---|---|
| 一次不必要的重渲染代价 | ~0.1ms (DOM patch) | ~2-5ms (Yoga + ANSI + write) |
| 每秒渲染次数 | 60fps 上限 | 不限（每次 state 变化都渲染） |
| Compiler 收益 | 中等 | **显著** |

---

## 前端项目接入指南

### 前提条件

| 条件 | 要求 |
|------|------|
| React 版本 | React 19（推荐）或 React 17/18 |
| Node.js | 18+ |
| 构建工具 | Vite / Webpack / Next.js 任一 |

### 安装

```bash
# React 19 项目
npm install -D babel-plugin-react-compiler@rc

# React 17/18 项目（需额外运行时包）
npm install -D babel-plugin-react-compiler@rc
npm install react-compiler-runtime
```

### Vite 接入

```javascript
// vite.config.js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [
    react({
      babel: {
        plugins: [
          ['babel-plugin-react-compiler']  // 放第一个
        ],
      },
    }),
  ],
})
```

### Next.js 15+ 接入

```javascript
// next.config.js
const nextConfig = {
  experimental: {
    reactCompiler: true,  // 一行搞定
  },
}

module.exports = nextConfig
```

### Webpack 接入

```javascript
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.(js|jsx|ts|tsx)$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            plugins: [
              ['babel-plugin-react-compiler'],  // 放第一个
              // ... 其他 babel 插件
            ],
          },
        },
      },
    ],
  },
}
```

### React 17/18 额外配置

```javascript
// babel 插件 options
{
  target: '18'  // 或 '17'
}
```

同时 `react-compiler-runtime` 包的 `package.json` 需指定 React 版本：

```json
{
  "dependencies": {
    "react": "^18.0.0"
  }
}
```

---

## 验证与渐进式引入

### 验证编译器是否生效

**方法 1：React DevTools**

安装 React DevTools 浏览器扩展，被优化的组件旁边会显示 ✨ 标记。

**方法 2：ESLint 插件**

```bash
npm install -D eslint-plugin-react-compiler
```

```javascript
// eslint.config.js (flat config)
import reactCompiler from 'eslint-plugin-react-compiler'

export default [
  {
    plugins: { 'react-compiler': reactCompiler },
    rules: {
      'react-compiler/react-compiler': 'error',
    },
  },
]
```

如果有组件不符合 React 规则（如条件中调 hook），ESLint 会报错。

**方法 3：检查构建产物**

```bash
grep "compiler-runtime" dist/assets/*.js
# 有输出 = 编译器生效
```

### 渐进式引入（大项目推荐）

**只对部分目录开启：**

```javascript
// babel-plugin-react-compiler options
{
  sources: (filename) => {
    return filename.includes('src/components')
  }
}
```

**对单个组件跳过：**

```tsx
function LegacyComponent() {
  "use no memo";  // 编译器跳过此组件
  // ... 不符合 React 规则的旧代码
}
```

---

## 注意事项

### 必须遵守 React 规则

编译器要求代码符合 [React Rules](https://react.dev/reference/rules)：

| 规则 | 说明 |
|------|------|
| 组件是纯函数 | 相同 props → 相同输出，不能有渲染期间副作用 |
| Hooks 不在条件中 | 不能 `if (x) { useState() }` |
| 不修改 props | 不能 `props.list.push(item)` |
| 副作用在 useEffect 中 | 不能在渲染函数体里直接修改外部状态 |

不符合规则的组件会被编译器**自动跳过**（不报错，只是不优化）。配合 ESLint 插件可以在开发时发现这些问题。

### 与现有代码的兼容性

| 现有代码 | 是否需要修改 |
|---------|-------------|
| 已有的 `useMemo` | ❌ 不需要删除，编译器会跳过已优化的部分 |
| 已有的 `useCallback` | ❌ 不需要删除，共存不冲突 |
| `React.memo` 包裹的组件 | ❌ 不需要删除，编译器额外优化内部 |
| Class 组件 | ⚠️ 不支持，会被跳过 |
| 第三方 hooks（react-query 等） | ✅ 正常工作 |

### 性能影响

| 维度 | 影响 |
|------|------|
| 构建时间 | 增加 10-20%（编译器需要额外分析） |
| 运行时包大小 | 增加极少（`react/compiler-runtime` 约 1KB） |
| 运行时性能 | 提升（减少不必要的重渲染和计算） |
| 内存 | 略增（每个组件多一个缓存数组） |

### 什么时候效果最明显

| 场景 | 收益 |
|------|------|
| 大列表 + 频繁过滤/排序 | ⭐⭐⭐ 避免重复计算 |
| 深层组件树 + 顶层状态变化 | ⭐⭐⭐ 避免瀑布式重渲染 |
| 传回调给子组件 | ⭐⭐ 自动稳定引用 |
| 简单展示组件 | ⭐ 收益小（本来就不重） |

---

## 总结

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  React Compiler 的核心价值：                                         │
│                                                                     │
│  把"性能优化"从一个需要经验判断的开发技能                            │
│  变成了编译器自动保证的事情                                          │
│                                                                     │
│  • 开发者：只管写清晰的逻辑代码                                     │
│  • 编译器：自动分析依赖，自动插入所有必要的缓存                      │
│  • Code Review：不再需要讨论"这里要不要加 useMemo"                  │
│  • 新人：不需要学习 memoization 最佳实践                            │
│                                                                     │
│  Claude Code 的 395 个文件全量启用证明：                             │
│  这个技术在大型生产项目中已经可靠可用。                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
