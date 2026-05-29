# Claude Code 缓存与索引架构

## 一、Anthropic API Prompt Cache（服务端缓存，最核心）

### 原理

由 Anthropic API 提供的服务端缓存。Claude Code 通过在请求中标记 `cache_control` 告诉 API "这部分内容可以缓存"，避免重复计费。

```
第1次请求:  [system prompt | tools | 历史消息 | 新消息]
            ├── 缓存创建 ──────────────┤├─ 新内容 ─┤
            
第2次请求:  [system prompt | tools | 历史消息 | 新消息1 | 新消息2]
            ├── 缓存命中（不重复计费）──────────────┤├─ 新 ─┤
```

### 代码实现

```javascript
// services/api/claude.ts:369
function getCacheControl({ querySource }) {
  return {
    type: 'ephemeral',                              // 临时缓存
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),  // 付费用户1小时TTL
  }
}

// 在消息的最后一个 block 上打标记:
{
  type: 'text',
  text: '...',
  cache_control: { type: 'ephemeral', ttl: '1h' }  // ← 缓存断点
}
```

### 两种 TTL

| TTL | 条件 | 效果 |
|-----|------|------|
| 5 分钟 | 默认（免费用户） | 5分钟内相同前缀命中缓存 |
| 1 小时 | 付费订阅用户 + 未超额 | 1小时内命中，大幅省钱 |

### 全局缓存（Global Cache Scope）

```javascript
// 所有用户共享同一份 system prompt + tools 的缓存
// 条件：没有 MCP 工具（MCP 是每用户不同的）
const globalCacheStrategy = useGlobalCacheFeature
  ? needsToolBasedCacheMarker ? 'none' : 'system_prompt'
  : 'none'
```

工具列表的顺序必须保持稳定（`tools.ts` 注释强调要和 Statsig 配置同步），否则字节级不同就会打破缓存。

### Cache Break 检测

`promptCacheBreakDetection.ts` 跟踪每次请求的 hash，当检测到缓存断裂时记录原因：

```javascript
type PreviousState = {
  systemHash: number,        // system prompt 的 hash
  toolsHash: number,         // 工具定义的 hash
  cacheControlHash: number,  // cache_control 标记的 hash
  toolNames: string[],       // 工具名列表
  model: string,             // 模型名
  // ...
}

// 检测到变化 → 记录事件
logEvent('tengu_prompt_cache_break', {
  systemPromptChanged, toolSchemasChanged, modelChanged, ...
})
```

---

## 二、Tool Schema Cache（工具描述缓存）

```javascript
// utils/toolSchemaCache.ts
// 整个会话期间，每个工具的 API schema 只渲染一次

const TOOL_SCHEMA_CACHE = new Map<string, CachedSchema>()

// 第一次调用 toolToAPISchema(BashTool) → 生成 schema → 存入 Map
// 后续调用 → 直接从 Map 取

// 好处：GrowthBook 特性标志切换不会导致工具描述字节变化 → 不打破 prompt cache
```

---

## 三、文件读取缓存（两层）

### 3A. FileReadCache — 编辑操作用

```javascript
// utils/fileReadCache.ts
class FileReadCache {
  private cache = new Map<string, CachedFileData>()  // 最多 1000 个文件
  
  readFile(filePath) {
    const stats = fs.statSync(filePath)
    const cached = this.cache.get(filePath)
    
    // mtime 没变 → 直接返回缓存
    if (cached && cached.mtime === stats.mtimeMs) {
      return { content: cached.content, encoding: cached.encoding }
    }
    
    // mtime 变了 → 重新读取
    const content = fs.readFileSync(filePath)
    this.cache.set(filePath, { content, encoding, mtime: stats.mtimeMs })
    return { content, encoding }
  }
}
```

### 3B. FileStateCache — LRU 缓存，记录模型"见过"的文件

```javascript
// utils/fileStateCache.ts
class FileStateCache {
  private cache: LRUCache<string, FileState>  // LRU, 25MB 上限, 100 条上限
  
  // 用于：
  // 1. 检测文件是否被模型读过（Edit 前必须先 Read）
  // 2. getChangedFiles() 比对哪些文件改了
  // 3. 判断是否是 partial view（如 CLAUDE.md 注入时裁剪过）
}

type FileState = {
  content: string
  timestamp: number
  offset: number | undefined
  limit: number | undefined
  isPartialView?: boolean  // 是否只看到了部分内容
}
```

---

## 四、Memoize 缓存（函数结果缓存）

用 lodash 的 `memoize` 把昂贵的计算/IO 结果缓存起来，同样输入只执行一次：

| 函数 | 缓存什么 | 失效条件 |
|------|---------|---------|
| `getSystemContext()` | git status/branch/log | 整个会话只算一次 |
| `getUserContext()` | CLAUDE.md 内容 | 整个会话只算一次 |
| `getCommands(cwd)` | 命令列表 | 手动 clearCommandsCache() |
| `getSkillDirCommands(cwd)` | skill 文件扫描结果 | 手动 clear 或文件变化 |
| `loadAllCommands(cwd)` | 合并后的全部命令 | 手动 clear |
| `getAgentDefinitionsWithOverrides(cwd)` | agent 定义 | 整个会话只算一次 |
| `getPrompt(cwd)` | SkillTool 的 system prompt | 会话内缓存 |
| `loadMarkdownFilesForSubdir()` | .claude/ 下的 md 文件 | 手动 clear |

---

## 五、Speculation（推测执行缓存）

当用户还在输入时，Claude Code 预测用户可能的下一步操作，提前执行：

```javascript
// services/PromptSuggestion/speculation.ts
const MAX_SPECULATION_TURNS = 20
const MAX_SPECULATION_MESSAGES = 100

// 流程：
// 1. 模型回复完一轮后，预测 "用户接下来可能说什么"
// 2. 在后台 fork 一个子 agent，用预测的 prompt 提前执行
// 3. 用户真的输入了匹配的内容 → 直接用缓存的结果（几乎 0 延迟）
// 4. 用户输入不匹配 → 丢弃推测结果
```

```
用户提交 "fix the bug"
    ↓
模型回复：修好了，改了 foo.ts
    ↓
后台推测：用户可能会说 "run the tests"
    ↓
提前 fork agent 执行 "run the tests"
    ↓
用户真的输入 "run tests" → 匹配！直接显示缓存的结果
    ↓
节省的时间显示在 UI: "Saved 3.2s"
```

---

## 六、UI/渲染缓存

### Fuse.js 搜索索引缓存

```javascript
// commands 引用不变就不重建 Fuse 索引
let fuseCache = { commands, fuse }
```

### 文件索引缓存（FileIndex）

```javascript
// hooks/fileSuggestions.ts
// git ls-files 结果的签名：文件列表没变就不重建 nucleo 索引
let loadedTrackedSignature: string | null = null
// 5秒刷新间隔
let lastRefreshMs = 0
```

### Ink 渲染帧缓存

```javascript
// ink/ink.tsx
// 双缓冲：frontFrame + backFrame
// 只输出两帧之间的 diff（不是每帧全刷）
private frontFrame: Frame
private backFrame: Frame
```

### Settings 缓存

```javascript
// utils/settings/settingsCache.ts
// 配置文件解析结果缓存，避免每次访问都重新读磁盘 + 解析 JSON
```

---

## 七、缓存层级全景图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          缓存层级全景                                     │
│                                                                          │
│  ┌── API 层 ──────────────────────────────────────────────────────────┐  │
│  │  Prompt Cache (服务端)  → system+tools+history 不变的部分不重复计费   │  │
│  │  Tool Schema Cache     → 工具定义整个会话只序列化一次                │  │
│  │  Cache Break Detection → 监控缓存命中率，定位断裂原因               │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌── 推测层 ──────────────────────────────────────────────────────────┐  │
│  │  Speculation           → 预测用户下一步，提前执行并缓存结果          │  │
│  │  Speculative Bash      → 提前跑权限分类器，命令执行时直接用结果      │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌── 数据层 ──────────────────────────────────────────────────────────┐  │
│  │  FileReadCache         → 文件内容 (mtime 失效, max 1000)           │  │
│  │  FileStateCache        → 模型见过的文件 (LRU, 25MB 上限)           │  │
│  │  Memoize (20+处)       → 昂贵计算结果 (会话级或手动失效)           │  │
│  │  Settings Cache        → 配置文件解析结果                          │  │
│  │  GrowthBook Cache      → 特性标志磁盘缓存                         │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌── UI 层 ──────────────────────────────────────────────────────────┐  │
│  │  Fuse.js Index Cache   → 命令搜索索引 (引用相等性)                 │  │
│  │  FileIndex (nucleo)    → 文件路径索引 (签名 + 5s TTL)             │  │
│  │  Ink Frame Buffer      → 双缓冲 + diff 输出                      │  │
│  │  Top Level Cache       → 空 query 时前 100 条文件结果              │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 八、模糊搜索索引详解

### 斜杠命令搜索：Fuse.js（第三方库）

用于 `/com` → 匹配 `/compact`, `/commit`, `/commands`

#### Fuse.js 简介

[Fuse.js](https://www.fusejs.io/) 是一个轻量级（~30KB）、零依赖的 JavaScript 模糊搜索库。基于 Bitap 算法（近似字符串匹配），能容忍拼写错误、字符遗漏。

#### 索引数据结构

每个命令被预处理为搜索项 `CommandSearchItem`：

```javascript
// 原始 Command:
{ name: "commit-push-pr", description: "Commit, push and create a PR", aliases: ["cpp"] }

// 转换为索引项:
{
  commandName: "commit-push-pr",              // 原名
  partKey: ["commit", "push", "pr"],          // 按 : - _ 分割的片段
  aliasKey: ["cpp"],                          // 别名数组
  descriptionKey: ["commit", "push", "create", "pr"],  // 描述拆词、清洗后
  command: { /* 原始 Command 引用 */ }
}
```

#### Fuse.js 配置

```javascript
const fuse = new Fuse(commandData, {
  includeScore: true,        // 返回匹配分数
  threshold: 0.3,            // 0=精确匹配, 1=匹配一切; 0.3 比较严格
  location: 0,               // 期望匹配出现在字符串开头
  distance: 100,             // 允许匹配离 location 多远
  keys: [
    { name: 'commandName',    weight: 3 },   // 命令名权重最高
    { name: 'partKey',        weight: 2 },   // 命令名片段
    { name: 'aliasKey',       weight: 2 },   // 别名
    { name: 'descriptionKey', weight: 0.5 }, // 描述权重最低
  ],
})
```

#### 搜索结果二次排序

Fuse.js 返回结果后，Claude Code 做了更精细的排序：

```
排序优先级:
1. 精确名称匹配     — "compact" 查 "compact" → 最高
2. 精确别名匹配     — "cpp" 查 "commit-push-pr" → 次高
3. 前缀名称匹配     — "com" 查 "compact" → 高（更短的名字优先）
4. 前缀别名匹配     — "c" 查 "cpp" → 中
5. Fuse 模糊分数    — 相似度
6. 使用频率 tiebreaker — 分数接近时常用的排前面
```

#### 缓存策略

```javascript
let fuseCache = null

function getCommandFuse(commands) {
  // 同一个数组引用 → 复用索引，不重建（每次按键都走这里）
  if (fuseCache?.commands === commands) {
    return fuseCache.fuse
  }
  // commands 变了（极少发生）→ 重建索引
  const fuse = new Fuse(commandData, options)
  fuseCache = { commands, fuse }
  return fuse
}
```

---

### 文件搜索：自研 FileIndex（nucleo 风格）

用于 `@src/com` → 匹配项目中的文件路径

位于 `src/native-ts/file-index/index.ts`，是 Rust [nucleo](https://github.com/helix-editor/nucleo)（Helix 编辑器的模糊搜索引擎）的 TypeScript 移植。

#### 核心算法：多阶段过滤 + 打分

```
输入: query = "comsu"
文件列表: 270,000 个路径

Stage 1: Bitmap 快速剔除 (O(1) per path)
  ↓ 每个文件只有一个 26-bit 整数 (a-z 字母存在位图)
  ↓ 如果文件路径缺少 query 中任何字母 → 立即跳过

Stage 2: indexOf 贪心扫描 + Gap 惩罚预计算
  ↓ 逐字符在路径中找 needle 的每个字符位置
  ↓ 同时计算 gap penalty（字符间距越大扣分越多）

Stage 3: Gap-bound 提前剔除
  ↓ 如果 "最佳可能分数 - gap惩罚" 都打不过当前 top-k 的最低分 → 跳过

Stage 4: Boundary/CamelCase 打分
  ↓ 匹配在路径分隔符后 → +8 (BONUS_BOUNDARY)
  ↓ 匹配在 camelCase 大写处 → +6 (BONUS_CAMEL)
  ↓ 连续匹配 → +4 (BONUS_CONSECUTIVE)
  ↓ 首字符匹配 → +8 (BONUS_FIRST_CHAR)
  ↓ 短路径奖励 → +(32 - 路径长度/4)

Stage 5: Top-K 维护
  ↓ 用排序数组 + 二分插入维护 top-k
  ↓ 只保留最好的 N 个结果
```

#### 打分常量

```javascript
const SCORE_MATCH = 16           // 每个匹配字符的基础分
const BONUS_BOUNDARY = 8         // 在路径分隔符后匹配
const BONUS_CAMEL = 6            // 在驼峰大写处匹配
const BONUS_CONSECUTIVE = 4      // 连续匹配奖励
const BONUS_FIRST_CHAR = 8       // 首字符匹配
const PENALTY_GAP_START = 3      // 间隔开始惩罚
const PENALTY_GAP_EXTENSION = 1  // 间隔距离惩罚
```

#### 性能优化技术

| 技术 | 效果 |
|------|------|
| Bitmap 快速剔除 | O(1) 跳过 ~10-90% 路径 |
| indexOf 扫描 | 利用 V8/JSC 的 SIMD 优化 |
| Gap-bound 提前退出 | 分数不可能进 top-k 就跳过 |
| Top-K 二分插入 | 避免对所有结果排序 |
| 异步分块构建 | 每 4ms yield 一次事件循环 |
| Smart Case | 全小写→忽略大小写；有大写→区分 |
| `readyCount` | 索引构建中也能搜已就绪的部分 |
| 缓存 `topLevelCache` | 空 query 直接返回缓存的 top 100 |
| 路径签名去重 | git ls-files 结果没变就不重建索引 |

#### 打分示例

搜索 `"comsu"` 匹配 `"src/components/Summary.tsx"`：

```
s r c / c o m p o n e n t s / S u m m a r y . t s x
        ↑ ↑ ↑               ↑ ↑
        c o m               s u

位置: [4, 5, 6, 15, 16]
gap:  (4→5: 连续) (5→6: 连续) (6→15: gap=8) (15→16: 连续)

score = 5 * 16                        // 80 基础
      + 3 * 4                          // 12 连续奖励
      + 8                              // 8 boundary ("/" 后的 "c")
      + 8                              // 8 boundary ("/" 后的 "S")
      - 3 - 8*1                        // -11 gap惩罚
      + max(0, 32 - 26/4)             // +25 短路径偏好
      = 122
```

---

### 两种搜索引擎的分工

```
用户输入        搜索引擎           搜索目标
─────────────────────────────────────────────────
/com         → Fuse.js          → Command[] (命令/skill 列表，~100个)
@src/com     → FileIndex        → 文件路径 (git ls-files，~270k个)
@agent-name  → Fuse.js          → AgentDefinition[] (代理列表)
#channel     → 简单 includes()  → Slack 频道名
!cmd         → shell completion → shell 命令/变量
```

| | Fuse.js | 自研 FileIndex |
|---|---|---|
| 数据量 | <1000 条 | 10万~50万条 |
| 搜索字段 | 多字段（名称+描述+别名） | 单字段（路径） |
| 特殊需求 | 多权重字段模糊匹配 | 路径分隔符感知、camelCase、极致性能 |
| 算法 | Bitap（通用模糊） | nucleo 风格（路径专用打分） |
| 延迟要求 | 宽松（几百条秒搜完） | 严格（27万条要 <5ms） |

---

## 九、设计哲学

每一层都在问同一个问题——"这个东西变了吗？没变就别重新算"。

- API 层：prompt cache 省 90% token 费用
- 数据层：memoize / LRU 避免重复 IO
- 索引层：引用相等性 / 签名比对避免重建
- 渲染层：双缓冲 + diff 只输出变化的字符
