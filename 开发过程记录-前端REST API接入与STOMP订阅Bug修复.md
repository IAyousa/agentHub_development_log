# 开发过程记录 — 前端 REST API 接入与 STOMP 订阅 Bug 修复

> **日期**: 2026-06-05\
> **关联分支**: `dev`\
> **涉及文件**: ChatView.vue、chat.ts、wsClient.ts

---

## 一、背景

经过 PR #7 和 PR #6 的合并，后端 Java 的 REST Controller 和 WebSocket Controller 已全部就绪，Python Agent 服务也已完善。但前端存在三个缺口：

1. `ChatView.onMounted` 只初始化了 WebSocket，没有调用 `loadConversationList()` 和 `loadAgents()`，会话列表和 Agent 列表仍使用 mock 数据
2. 新建会话后切换过去时，STOMP 订阅可能静默失败，AI 回复无法实时到达
3. 刷新页面后消息顺序颠倒（AI 回复出现在用户消息上方）

---

## 二、任务 1：前端接入 REST API

### 2.1 现状分析

前端 API 层的代码早已写好：

| API 函数 | 对应端点 | Store 方法 | 是否被调用 |
|----------|---------|-----------|:--:|
| `getConversations()` | GET /conversations | `loadConversationList()` | ❌ 从未调用 |
| `getAgents()` | GET /agents | `loadAgents()` | ❌ 从未调用 |
| `getConversationMessages()` | GET /conversations/{id}/messages | `loadMessages()` | ✅ selectConversation 中调用 |

`loadConversationList()` 和 `loadAgents()` 内部都有 try/catch 静默降级——后端不可用时保留 mock 数据，UI 不受影响。

### 2.2 修复

**仅改 `ChatView.vue` 的 `onMounted`，加两行**：

```ts
onMounted(() => {
  chatStore.mobileView = 'list'
  chatStore.loadConversationList()  // 新增：GET /conversations
  chatStore.loadAgents()            // 新增：GET /agents
  chatStore.initWebSocket()
})
```

### 2.3 验证

启动 Java + Python → 启动前端 → DevTools Network 面板观察到：
- `GET /conversations?archived=false` → 200，返回 6 个真实会话
- `GET /agents` → 200，返回 Claude Code + Codex 两个 Agent
- ChatList 显示 6 个会话（之前 mock 只有 3 个）

---

## 三、任务 2：修复 STOMP 订阅与消息顺序 Bug

### 3.1 Bug 1：STOMP subscribe() 静默失败

#### 问题代码

`wsClient.ts` 的 `subscribe()` 方法没有返回值，调用方无法知道是否成功：

```ts
subscribe(conversationId: string): void {
    const topic = `/topic/conversation.${conversationId}`
    const subscription = this.client.subscribe(topic, ...)
    this.subscriptions.set(topic, subscription)
}
```

`chat.ts` 中用 try/catch 包裹但 catch 为空——失败时没有任何日志：

```ts
try {
    ws.subscribe(id)
} catch { /* ignore */ }
```

#### 修复

`wsClient.ts`：添加连接状态守卫，返回 boolean：

```ts
subscribe(conversationId: string): boolean {
    if (!this.client.connected) {
      console.warn('[wsClient] Cannot subscribe: STOMP not connected')
      return false
    }
    // ... 原有逻辑
    return true
}
```

`chat.ts`：用返回值判断，失败时打日志：

```ts
const ok = ws.subscribe(id)
if (!ok) {
    console.warn('[chat] subscribe failed for', id, '- will retry on reconnect')
}
```

即使订阅失败，`handleConnectionChange` 在 STOMP 重连后会自动重新订阅当前会话，不会永久丢失。

### 3.2 Bug 2：新建会话时 selectConversation 被跳过

#### 问题根因

`createConversation()` 提前设置了 `currentConversationId.value = id`，导致随后的路由导航 → watch → `selectConversation(id)` 因 `if (currentConversationId.value === id) return` 直接跳过，三个关键初始化都没执行：

```text
createConversation()
  → currentConversationId.value = "abc-123"    // 提前设置
  → router.push('/chat/abc-123')
    → watch → selectConversation("abc-123")
      → if (currentConversationId === "abc-123") return  // 跳过！
      ✗ ws.unsubscribe(oldId)    没执行
      ✗ ws.subscribe("abc-123")  没执行  
      ✗ loadMessages("abc-123")  没执行
```

#### 修复

删除 `createConversation()` 中多余的 `currentConversationId.value = id`，让 router → watch → selectConversation 完成完整初始化：

```ts
// 修改前
conversations.value[id] = { id, title: res.data.title, messages: [] }
currentConversationId.value = id   // ← 删除这行

// 修改后
conversations.value[id] = { id, title: res.data.title, messages: [] }
// currentConversationId is set by selectConversation() via router watch
```

### 3.3 Bug 3：消息顺序颠倒

#### 问题根因

Java 的 `MessageRepository` 使用 `OrderByCreatedAtDesc`（最新在前），分页查询的最新 50 条消息中 AI 回复排在用户消息前面。前端直接渲染，导致新消息出现在顶部。

#### 修复

前端 `loadMessages()` 收到数据后 `.reverse()` 反转：

```ts
const messages = res.data.messages.map(mapApiMessage).reverse()
// API returns newest-first (DESC); reverse to display oldest-first in chat UI
```

Java 侧保持 DESC（正确的分页语义——第一页应返回最新消息），前端负责 UI 展示的排序。

### 3.4 验证

| # | 测试场景 | 结果 |
|---|---------|:--:|
| 3.4a | Console 无 subscribe failed 警告 | ✅ |
| 3.4b | 默认会话发消息 → 实时收到 AI 回复 | ✅ |
| 3.4c | 新建会话 → 发消息 → 实时收到回复 | ✅ |
| 3.4d | 刷新后消息顺序正确（用户消息在上，AI 回复在下） | ✅ |

---

## 四、最终产物

### 4.1 代码变更

| 文件 | 改动 | 说明 |
|------|------|------|
| `ChatView.vue` | +2 行 | onMounted 新增 loadConversationList + loadAgents |
| `wsClient.ts` | +8/-1 行 | subscribe() 加连接守卫 + 返回 boolean |
| `chat.ts` | 3 处修改 | 去静默吞错、删多余 currentConversationId、消息数组反转 |

### 4.2 提交记录

```
c5c0dde docs: 同步文档对齐前端REST API接入+STOMP bug修复后的项目状态
318d819 fix(frontend): 前端接入REST API并修复STOMP订阅与会话初始化两个bug
```

---

## 五、经验总结

### 5.1 "已写好的代码"要检查调用链

`loadConversationList()` 和 `loadAgents()` 的方法体、API 调用、错误处理全部就绪，但没有任何组件调用它们。这是因为 C7 开发时为了避免重连风暴移除了这两个调用，但后续新增了 ConversationController 和 MessageController 后没有补回去。

**教训**：功能代码写好后，要追踪完整的调用链——从 UI 事件或生命周期钩子一路跟到 API 调用。

### 5.2 静默 catch 掩盖问题

`try { ... } catch { /* ignore */ }` 在生产代码中零日志的代价是巨大的——STOMP 订阅失败导致用户看不到 AI 回复，但开发者完全无法从 Console 发现，只能靠用户报告。

**教训**：即使是 fallback 逻辑，catch 块至少应该 `console.warn` 或 `log.warn`。

### 5.3 注意函数间的"职责越界"

`createConversation()` 设置 `currentConversationId` 越过了自己的职责边界。会话切换的初始化（取消旧订阅、建立新订阅、加载消息）应该由专门的 `selectConversation()` 统一完成。一个函数做多件事的"便利"最终会变成调试的噩梦。
