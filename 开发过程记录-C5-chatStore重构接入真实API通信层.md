# 开发过程记录 — C5 chatStore 重构接入真实 API 通信层

> **日期**: 2026-06-03\
> **关联分支**: `dev`\
> **所属模块**: frontend\
> **Commit**: `db7e840`

## 一、背景

C2-C4 分别建立了会话 API、Agent API 和 WebSocket 客户端三个隔离的模块，但 Pinia Store（`chat.ts`）仍是纯 mock 数据 + 本地操作。C5 是通信层搭建的最后一步——将这些模块集成到 Store 中，让状态管理与真实后端建立联系。

## 二、技术实现

### 2.1 架构概览

```
chatStore (Pinia)
  ├── C2 conversation.ts → REST API → 会话 CRUD + 历史消息
  ├── C3 agent.ts        → REST API → Agent 列表/详情/创建
  ├── C4 wsClient.ts     → STOMP   → 消息收发 + 流式 tokens
  └── 本地 mock 数据     → 降级策略 → API 不可用时保底
```

### 2.2 核心方法

**loadConversationList / loadAgents / loadMessages** — REST API 加载：

```ts
async function loadConversationList() {
    try {
      const res = await getConversations(false)
      conversationList.value = res.data.conversations.map(c => ({...}))
    } catch {
      // API 不可用时保留 mock 数据，不报错
    }
}
```

设计原则：API 成功则覆盖 mock 数据，失败则静默降级——前端始终有数据可渲染。

**initWebSocket** — 建立 STOMP 连接：

```ts
function initWebSocket() {
    const ws = getWsClient(wsCallbacks)  // 传入4个回调
    ws.connect()
}
```

**sendMessage** — WebSocket 发送消息：

```ts
function sendMessage(content: string) {
    const userMsg: Message = { ... }
    conversations.value[convId].messages.push(userMsg)

    isLoading.value = true
    const ws = getWsClient(wsCallbacks)
    ws.sendMessage({ conversationId: convId, content: content.trim() })
}
```

**createConversation** — 异步创建会话：

```ts
async function createConversation(title: string): Promise<string> {
    try {
      const res = await apiCreateConversation({ title, type: 'direct', agentIds: [] })
      return res.data.id
    } catch {
      const id = `conv_${Date.now()}`
      // 本地创建降级
      return id
    }
}
```

### 2.3 流式 token 处理

**handleToken** — 打字机渲染核心：

```ts
const handleToken = (chunk: MessageChunk) => {
    const conv = conversations.value[currentConversationId.value]
    if (!conv) return

    if (chunk.isComplete) {
      // 1. 移除之前的临时流式消息
      conv.messages = conv.messages.filter(m => m.id !== streamingMessageId.value)
      // 2. 添加固化的完整消息
      conv.messages.push({ id: chunk.messageId, ... })
      // 3. 清理状态
      streamingMessageId.value = null
      isLoading.value = false
    } else {
      if (!streamingMessageId.value) {
        // 首个 chunk：创建占位消息
        streamingMessageId.value = `stream_${Date.now()}`
        conv.messages.push({ id: streamingMessageId.value, role: 'agent', content: '' })
      }
      // 追加 token 到占位消息
      const msg = conv.messages.find(m => m.id === streamingMessageId.value)
      if (msg) msg.content += chunk.token
    }
}
```

**handleAgentSwitch** — 发言人切换：

```ts
const handleAgentSwitch = (agentId: string, agentName: string) => {
    currentAgentId.value = agentId
    currentAgentName.value = agentName
}
```

### 2.4 Store 暴露的状态与内部状态

**外部可读（通过 storeToRefs）**：
- `conversations`、`conversationList`、`agents`、`isLoading`、`error`
- `currentConversationId`、`currentAgentId`、`currentAgentName`
- `isArtifactVisible`、`currentArtifact`、`offices`
- Computed: `currentMessages`、`currentTitle`

**内部专用**（不暴露给组件）：
- `streamingMessageId` — 流式输出中临时消息的 ID
- `wsConnected` — WebSocket 连接状态
- `wsCallbacks` — 注入 wsClient 的 4 个回调闭包

### 2.5 Vite 配置修复

`sockjs-client` 在浏览器环境下依赖 `global` 对象，Vite 不自动提供。在 `vite.config.ts` 中添加：

```ts
define: {
  global: 'globalThis',
}
```

## 三、产物清单

| 文件 | 操作 | 行数变化 |
|------|------|---------|
| `frontend/src/stores/chat.ts` | 重写 | +524/-159 |
| `frontend/vite.config.ts` | 修改 | +5 |

### 代码统计

| 指标 | 数值 |
|------|------|
| 新增代码行 | 524 |
| 删除代码行 | 159 |
| 净增加代码行 | 365 |
