# 开发过程记录 — ChatList 改为展示会话列表

> **日期**: 2026-05-28\
> **任务编号**: C8\
> **所属模块**: frontend\
> **前置依赖**: C6 MessageInput 组件提取 ✅（已完成）

---

## 一、背景

当前 `ChatList.vue` 中间栏展示的是 **Agent 列表**（Orchestrator / Coder / Designer），与团队设计决策 [2.2 中间栏展示内容](团队任务分配.md#22-中间栏展示内容) 不一致。

根据设计文档，中间栏应展示**用户的会话列表**，而非 Agent 列表。每个会话可通过 `conv_agents` 关联表绑定多个 Agent，用户点击会话后根据 `conversation.type` 自动路由到 ChatView（direct）或 OfficeView（group）。

**当前状态的问题**：

| 方面 | 当前（错误） | 目标（正确） |
|------|-------------|-------------|
| 展示内容 | Agent 列表 | 用户会话列表 |
| 数据字段 | `name`, `lastMsg`, `avatar` | `title`, `lastMessage`, `type`, `agentNames` |
| 头像 | DiceBear Agent 风格头像 | 会话首字母（渐变色方块） |
| 新建入口 | 无 | 内联输入框 + 确认/取消按钮 |

**C8 完成的后续价值**：为 C9（Vue Router 按 `conversation.type` 分发路由）提供真实的数据模型，同时为 C5（切换到真实 API）预留了接口对齐的 `ConversationSummary` 类型定义。

---

## 二、任务执行过程

### 2.1 现状分析

改造前的 ChatList.vue 结构（77 行）：

```
ChatList.vue
├── Search Bar          — 搜索框
└── Agent List（v-for="item in mockChats"）
    ├── Avatar           — DiceBear 外部图片
    ├── Name             — Agent 名称
    └── LastMsg          — 最后一条消息预览
```

问题：
- `mockChats` 数据硬编码在组件内部，不在 store 中管理
- 数据结构与 API 契约不匹配，后续切换真实接口需要大量改动
- 点击 Agent 直接设置 `currentConversationId` 为 Agent ID，但实际上 store 中 `conversations` 的 key 也是 Agent ID（`orchestrator`、`coder`），恰好能工作，但架构上是错误的

### 2.2 Store 类型层：新增 ConversationSummary

在 `stores/chat.ts` 中新增 `ConversationSummary` 接口，字段完全对齐 [API 文档 2.2.1 节](API%20契约定义与通信协议规范.md#221-获取会话列表) 的 `GET /conversations` 响应格式：

```typescript
export interface ConversationSummary {
  id: string
  title: string
  type: 'direct' | 'group'     // 联合类型：P0 阶段只有 direct
  lastMessage: string
  updatedAt: string
  agentNames: string[]          // 参与 Agent 名称数组
  isArchived?: boolean          // 可选：默认不归档
}
```

**设计要点**：
- `type: 'direct' | 'group'` 是 TypeScript 的**字面量联合类型**，编译期即可检查非法值，为 C9 的路由分发提供类型安全保障
- 字段名使用 `camelCase`，与 API 响应一致（Spring Boot 默认 Jackson 驼峰序列化）
- `isArchived` 设为可选字段，目前默认不归档，为后续会话管理功能预留

### 2.3 Store 数据层：conversationList 与 conversations 分离

```typescript
const conversationList = ref<ConversationSummary[]>([...])  // 列表元数据
const conversations = ref<Record<string, Conversation>>({...})  // 消息内容
```

**为什么分成两个数据结构？**

| 结构 | 用途 | 读写频率 | 使用者 |
|------|------|----------|--------|
| `conversationList` | 列表视图元信息（标题、摘要、时间） | 读多写少 | ChatList |
| `conversations` | 单会话的消息数组 | 高频写（每条消息追加） | ChatWindow |

这是**关注点分离**的体现：列表渲染不需要遍历所有消息，消息操作不需要更新列表元数据。两者通过 **ID 桥接**（`conversationList[i].id === conversations 的 key`）关联。

### 2.4 Store Action 层：createConversation

```typescript
const createConversation = (title: string) => {
  const id = `conv_${Date.now()}`           // 时间戳生成唯一 ID
  conversationList.value.unshift(newConv)    // ① 列表元数据：头部插入
  conversations.value[id] = { ... }          // ② 消息容器：初始化 + 欢迎语
  currentConversationId.value = id           // ③ 自动选中新会话
  mobileView.value = 'chat'                  // ④ 移动端自适应
}
```

**关键技术点**：
- `unshift()` 而非 `push()`：新会话出现在列表顶部，符合用户预期
- 两步写入必须**同步完成**：如果只写 `conversationList` 不写 `conversations`，选中新会话时 `currentMessages` computed 返回空数组，ChatWindow 白屏
- `mobileView = 'chat'`：移动端新建后自动展示聊天面板而非停留在列表

### 2.5 ChatList.vue 全量重写

改造后的组件结构（87 行）：

```
ChatList.vue
├── Top Bar（v-if / v-else 状态切换）
│   ├── 正常模式：搜索框 + "+" 按钮
│   └── 创建模式：输入框 + 确认按钮 + 取消按钮（X）
└── Conversation List（v-for="item in conversationList"）
    ├── Avatar — 首字母（gradient block）
    ├── Title — 会话标题
    ├── UpdatedAt — formatTime 相对时间
    ├── Agent Tags — 最多 2 个，超出 +N
    └── LastMessage — 最后消息预览
```

**与改造前的关键差异**：

| 改造点 | 之前 | 之后 |
|--------|------|------|
| 数据源 | `mockChats`（组件内硬编码） | `conversationList`（store 响应式数据） |
| 响应式获取 | 直接变量 | `storeToRefs(chatStore)` |
| 头像 | `<img :src="item.avatar">` | 首字母 `<span>{{ item.title.charAt(0) }}</span>` |
| 时间格式 | 静态字符串 `"16:40"` | `formatTime()` 动态相对时间 |
| Agent 信息 | 无 | 标签 `<span>` 最多 2 个 + "+N" |
| 新建入口 | 无 | 内联输入框 + Enter/Esc 键盘支持 |

### 2.6 内联创建模式的交互设计

最初实现使用 `prompt()` 原生弹窗，但在 Trae 预览浏览器中报错 `Error: prompt() is not supported`，改为**内联输入模式**：

```
点击 "+" → showCreateInput = true
         → 搜索栏切换为输入框（紫色边框）
         → 输入名称 → Enter 或点确认 → confirmCreate()
         → Esc 或点 X → cancelCreate()
```

**状态管理**：

| 状态变量 | 类型 | 作用 |
|----------|------|------|
| `showCreateInput` | `ref<boolean>` | 控制 v-if/v-else 切换两种模式 |
| `createTitle` | `ref<string>` | v-model 双向绑定输入内容 |

**键盘快捷键**：`@keydown.enter="confirmCreate"` / `@keydown.escape="cancelCreate"`

**禁用联动**：`createTitle.trim()` 为空时确认按钮置灰（`disabled:opacity-40 disabled:cursor-not-allowed`）

### 2.7 辅助函数：formatTime 相对时间

```typescript
const formatTime = (iso: string) => {
  const diffMin = Math.floor(diffMs / 60000)
  if (diffMin < 1) return '刚刚'
  if (diffMin < 60) return `${diffMin}分钟前`
  // ...
  if (diffDay < 7) return `${diffDay}天前`
  return date.toLocaleDateString('zh-CN', { month: 'short', day: 'numeric' })
}
```

采用**相对时间衰减策略**：近期的用相对时间（"3分钟前"），7 天以上的用绝对日期（"5月28日"）。这是会话列表 UI 的常见模式（参考微信、Slack）。

---

## 三、技术要点详解

### 3.1 ConversationSummary 与 Conversation 的分离设计

```
conversationList（Array）               conversations（Record）
┌──────────────────────────┐           ┌─────────────────────────┐
│ { id: "conv_frontend_001",│───ID───→ │ "conv_frontend_001": {  │
│   title: "前端博客开发",    │  桥接    │   title: "前端博客开发",   │
│   lastMessage: "...",     │           │   messages: [...]      │
│   agentNames: [...] }    │           │ }                      │
└──────────────────────────┘           └─────────────────────────┘
```

- ChatList 只依赖 `conversationList`，渲染复杂度 O(n)，n 为会话数量（通常 < 20）
- ChatWindow 通过 `currentConversationId` 从 `conversations` 中 O(1) 取值
- C5 改造为真实 API 时，`conversationList` 由 `GET /conversations` 填充，`conversations` 由 `GET /conversations/{id}/messages` 懒加载

### 3.2 storeToRefs 的响应性保持

```typescript
// ✅ 正确：保留响应性
const { currentConversationId, conversationList } = storeToRefs(chatStore)

// ❌ 错误：直接解构会丢失 Vue 的响应式代理
const { currentConversationId } = chatStore
```

Pinia store 本质上是 `reactive()` 包装的对象。直接解构会得到原始值（非响应式），`storeToRefs()` 将每个属性包装成独立的 `Ref`，保持模板中的响应式更新。

### 3.3 首字母头像替代外部图片

```html
<div class="bg-gradient-to-br from-indigo-400 to-purple-500">
  <span>{{ item.title.charAt(0) }}</span>
</div>
```

**原因**：
- 不再依赖 DiceBear 外部 API（减少网络请求，离线可用）
- 会话数量增长后不需要为每个会话设计头像
- `charAt(0)` 取中文首字（如"前"、"后"、"代"），简洁直观

### 3.4 Agent 标签的截断策略

```html
<span v-for="name in item.agentNames.slice(0, 2)" :key="name">
  {{ name }}
</span>
<span v-if="item.agentNames.length > 2">+{{ item.agentNames.length - 2 }}</span>
```

在 64px 宽度的侧边栏中，最多显示 2 个 Agent 标签（`max-w-[80px] truncate`），超出部分显示 "+N"。参考 Slack 频道成员预览的设计模式。

---

## 四、验证测试

共设计 6 个测试用例，全部通过：

| # | 测试项 | 操作 | 预期结果 | 结果 |
|---|--------|------|----------|------|
| 1 | 会话列表展示 | 观察中间栏 | 3 个会话（前端博客开发/后端接口重构/代码审查），非 Agent 列表 | ✅ |
| 2 | 选中态高亮 | 点击"后端接口重构" | 高亮切换，ChatWindow 显示 Codex 欢迎语 | ✅ |
| 3 | 切换回第一项 | 点击"前端博客开发" | 高亮恢复，消息内容正确 | ✅ |
| 4 | 新建会话 | 点击 + → 输入"测试新会话" → 确认 | 新会话出现于顶部、自动选中、有欢迎语 | ✅ |
| 5 | 新建后切回 | 点击已有会话 | 原有数据不受影响 | ✅ |
| 6 | 多 Agent 标签 | 观察"代码审查"会话 | 显示 Claude Code + Codex 两个标签，无溢出 | ✅ |

**类型检查**：`vue-tsc -b --noEmit` 通过，仅 OfficeView 预存的 `onOwnerChairClick` 未使用报错（与 C8 无关）。

---

## 五、关键决策与取舍

| 决策 | 选择 | 理由 |
|------|------|------|
| conversationList 数据位置 | 放在 store 中 | 为 C5 真实 API 切换预留，数据源集中管理 |
| 新建会话输入方式 | 内联输入框 | prompt() 在 Trae 预览浏览器不支持；内联模式样式可控、支持键盘操作 |
| 会话 ID 生成方式 | `conv_${Date.now()}` | mock 阶段时间戳即可；C5 改造时改为后端返回的 UUID |
| conversations 默认值 | `conv_frontend_001` | 保证页面加载时有默认选中会话，ChatWindow 不白屏 |
| 首字母 vs 外部头像 | 首字母 | 离线可用、无网络依赖、与 API 头像字段不冲突 |

---

## 六、产物清单

| 文件 | 变更类型 | 行数 | 说明 |
|------|----------|------|------|
| `frontend/src/stores/chat.ts` | 修改 | +59 行 | 新增 `ConversationSummary` 类型、`conversationList` mock 数据、`createConversation` action；`conversations` 重构为会话维度 |
| `frontend/src/components/chat/ChatList.vue` | 重写 | 87 行 | 从 Agent 列表改为会话列表，新增内联创建模式、相对时间、Agent 标签 |

---

## 七、架构影响

```
改造前：
  ChatList → 硬编码 Agent 列表 → 点击设置 currentConversationId = Agent ID
                                  ↓
                            ChatWindow 取 conversations[Agent ID].messages

改造后：
  ChatList → store.conversationList → 点击设置 currentConversationId = conversation.id
                                        ↓
                                  ChatWindow 取 conversations[conversation.id].messages
```

**关键变化**：`currentConversationId` 从 Agent ID 改为 Conversation ID，两者在 store 中通过 ID 桥接。ChatWindow 的取数逻辑（`conversations[currentConversationId]`）保持不变，无需修改。

---

## 八、后续依赖

```
C8（ChatList 展示会话）✅
  │
  ├──→ C9（Vue Router）
  │      conversation.type === 'direct' → ChatView
  │      conversation.type === 'group'  → OfficeView
  │
  └──→ C5（chatStore 改造）
         替换 conversationList 数据源为 GET /conversations
         替换 conversations 数据源为 GET /conversations/{id}/messages
         类型定义零改动
```
