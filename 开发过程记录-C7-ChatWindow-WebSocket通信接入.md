# 开发过程记录 — C7 ChatWindow 接入真实 WebSocket 通信

> **日期**: 2026-06-03\
> **关联分支**: `dev`\
> **所属模块**: frontend\
> **任务概要**: ChatWindow 从 `setTimeout` mock 模拟切换为通过 STOMP/WebSocket 真实发送消息，接入已完成的 chatStore.sendMessage() 通信层

---

## 一、背景

C5 重构已在 chatStore 中实现了完整的 `sendMessage()`（WebSocket 发送 + 流式 token 接收）、`initWebSocket()`（STOMP 连接建立）和 `handleToken`（逐 token 打字机渲染），但 ChatWindow 组件从未接入——它仍使用 `simulateResponse()` 通过 `setTimeout` + 关键词匹配返回假的 HTML/CSS/JS artifact。

C7 的目标是打通前端 UI → Store → WebSocket 这条链路，让用户输入真正通过 STOMP 协议发送到后端。

### 变更涉及

| 文件 | 说明 |
|------|------|
| `ChatWindow.vue` | 删除 `simulateResponse`（131 行 mock）、改为 `sendMessage()`、新增浮动 toast、MessageInput 加 key |
| `ChatView.vue` | `onMounted` 中初始化 WebSocket 连接 |
| `chatStore (chat.ts)` | 重构 `sendMessage`、修复 `initWebSocket`、补 `handleWsError`、`selectConversation` 状态清理 |
| `App.vue` | RouterView 迁移到 Vue Router 4 slot props 语法 |

---

## 二、核心实现

### 2.1 ChatWindow — 删除 Mock，接入真实发送

**修改前**：
```ts
const handleSend = (content: string) => {
  chatStore.addMessage({...})   // 手动添加用户消息
  simulateResponse(content)      // setTimeout → 关键词匹配 → 假回复
}
```

`simulateResponse` 共 131 行，根据关键词（counter/animation/dashboard/code/preview）返回硬编码的 HTML artifact 或 JavaScript 代码片段。

**修改后**：
```ts
const handleSend = (content: string) => {
  chatStore.sendMessage(content)
}
```

`sendMessage()` 内部一次性完成三件事：添加用户消息 → 设置 `isLoading` → WebSocket 发布到 `/app/chat.send`。

### 2.2 chatStore — sendMessage 重构

```ts
function sendMessage(content: string) {
    // 1. 先添加用户消息（确保 UI 始终有反馈，不受后端状态影响）
    conversations.value[convId].messages.push(userMsg)

    // 2. 检查 WebSocket 连接状态
    const ws = getWsClient(wsCallbacks)
    if (!ws.connected) {
      error.value = '服务端连接出现问题，请稍后重试'
      return
    }

    // 3. 已连接 → 标记加载中 → 发送
    isLoading.value = true
    error.value = null
    ws.sendMessage({ conversationId: convId, content: content.trim() })
}
```

**设计原则**：用户消息优先添加到本地（无论后端状态如何），再检查连接状态决定是否通过 WebSocket 发送。这样后端离线时用户输入不会丢失，且能收到明确的错误提示。

### 2.3 ChatView — 页面加载时初始化 WebSocket

```ts
onMounted(() => {
  chatStore.mobileView = 'list'
  chatStore.initWebSocket()
})
```

`initWebSocket()` 获取 wsClient 单例 → `connect()` → `client.activate()` → SockJS 异步握手 → STOMP CONNECTED 帧到达 → `onConnect` 回调 → `handleConnectionChange(true)` → 订阅当前会话 topic。

---

## 三、踩坑与修复

### 3.1 错误 1：STOMP v7 不兼容的过早 subscribe()

**现象**：
```
stomp.umd.js:1668 Uncaught (in promise) TypeError: There is no underlying STOMP connection
    at Client.subscribe (stomp.umd.js:1729:18)
```

**根因**：`initWebSocket()` 中 `ws.connect()` 是异步的（SockJS 需要时间完成握手），但紧接着同步调用了 `ws.subscribe()`。`@stomp/stompjs` v7 要求 subscribe 必须在 CONNECTED 帧之后，否则抛出异常。该异常导致 `initWebSocket()` 中断，WebSocket 连接从未成功激活，后续 `ws.connected` 永远是 `false`。

**修复**：从 `initWebSocket()` 中移除 `subscribe()` 调用。订阅由 `handleConnectionChange` 回调在连接就绪后自动完成。

```ts
// 修复前
function initWebSocket() {
    const ws = getWsClient(wsCallbacks)
    ws.connect()
    if (currentConversationId.value) {
      ws.subscribe(currentConversationId.value)  // ← 这里抛异常
    }
}

// 修复后
function initWebSocket() {
    const ws = getWsClient(wsCallbacks)
    ws.connect()
    // 订阅由 handleConnectionChange 在 STOMP 连接就绪后自动完成
}
```

### 3.2 错误 2：CORS 拦截预检请求

**尝试**：在 `onMounted` 中用 `fetch('http://localhost:8080', { method: 'HEAD' })` 检测后端是否可达，再决定是否初始化 WebSocket。

**现象**：
```
Access to fetch at 'http://localhost:8080/' from origin 'http://localhost:5173'
has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header
```

**根因**：浏览器的 CORS 策略对 `fetch()` 的跨域请求强制要求服务端返回 `Access-Control-Allow-Origin` 头。后端 `CorsConfig` 未给根路径配置 CORS 头，预检始终失败，WebSocket 从未初始化。

**修复**：移除 HTTP 预检。SockJS/WebSocket 握手不受 CORS 限制（WebSocket 升级请求使用独立的协议机制），直接调用 `initWebSocket()` 即可。

### 3.3 错误 3：Loading 动画全局泄漏

**现象**：在会话 A 发送消息后切换到会话 B，B 也显示"Agent 正在思考..."动画。

**根因**：`isLoading` 是 store 级别的全局状态，不是按会话隔离的。A 中设为 `true` 后切换会话，B 也读取到 `true`。

**修复**：在 `selectConversation()` 切换会话时清理三个全局状态：

```ts
isLoading.value = false
streamingMessageId.value = null
error.value = null
```

### 3.4 错误 4：handleWsError 未重置 isLoading

**现象**：后端离线时发送消息，WebSocket 未连接，`wsClient.sendMessage()` 调用 `onError` 回调，但 `isLoading` 未被清除。

**修复**：
```ts
const handleWsError = (err: string) => {
    error.value = err
    isLoading.value = false  // 新增
}
```

### 3.5 改进：MessageInput 切换会话清空

**现象**：在会话 A 输入了文字但未发送，切换到会话 B 后输入框仍残留旧文本。

**根因**：`MessageInput` 的 `input` 是组件本地 `ref`，不会因路由切换自动重建。

**修复**：给 `MessageInput` 添加 `:key="chatStore.currentConversationId"`，Vue 在会话 ID 改变时销毁旧组件、创建新组件，`input` 自然为空。

---

## 四、Toast 错误提示设计演进

### 4.1 第一版：固定红色横幅

位于 header 和消息区之间的静态错误条，需手动点击 X 关闭。

**用户反馈**：希望设计成浮动弹窗，停留一会自动消失。

### 4.2 第二版：浮动 Toast + 弹性滑入动画

`absolute` 定位浮在页面上方，带 `cubic-bezier` 弹性滑入动画。通过 Vue `watch(error)` 触发。

**问题 1**：同一错误值重复赋值（"服务端连接出现问题，请稍后重试"），Vue `watch` 不触发后续回调。

**问题 2**：用户反馈动画太花哨，希望直接显示。

### 4.3 最终版：纯淡入淡出 + 命令式触发

```ts
// ChatWindow.vue
const toastMessage = ref('')
const showToast = (msg: string) => {
  dismissToast()
  toastMessage.value = msg
  toastTimer = setTimeout(() => { toastMessage.value = '' }, 4000)
}

const handleSend = (content: string) => {
  chatStore.sendMessage(content)
  if (!chatStore.wsConnected) {
    showToast('服务端连接出现问题，请稍后重试')
  }
}
```

**关键改进**：
- 改用命令式调用（`handleSend` 中直接检查 `wsConnected`），不依赖 Vue watch，每次发送都触发
- 动画改为纯 `opacity` 过渡（0.2s），直接显示/消失
- 4 秒自动消失 + 手动 X 关闭 + 切换会话自动消失

---

## 五、手动测试验证

> 用户在前端浏览器中交互操作，观察 DevTools Console 和 Network 面板。

### 5.1 后端在线测试

| # | 测试项 | 预期 | 结果 |
|---|--------|------|:--:|
| 1 | 页面加载后 WebSocket 连接建立 | Network WS 标签出现 `/ws-chat` 连接 | ✅ |
| 2 | 发送消息 — 消息显示 | 用户消息立即出现在聊天区 | ✅ |
| 3 | 发送消息 — 加载动画 | "Agent 正在思考..."动画出现 | ✅ |
| 4 | 切换会话 — 输入框清空 | 新会话输入框为空 | ✅ |
| 5 | 切换会话 — 动画不残留 | 新会话无加载动画 | ✅ |

### 5.2 后端离线测试

| # | 测试项 | 预期 | 结果 |
|---|--------|------|:--:|
| 1 | 页面加载无 WS 重试风暴 | Console 无持续 WS 报错、页面导航正常 | ✅ |
| 2 | 发送消息 — 消息仍显示 | 消息出现在聊天区（本地保留） | ✅ |
| 3 | 发送消息 — toast 提示 | 浮动弹窗"服务端连接出现问题，请稍后重试" | ✅ |
| 4 | 每次发送都弹出 toast | 连续发送 3 次，每次都有 toast | ✅ |
| 5 | toast 4 秒自动消失 | 计时消失 | ✅ |
| 6 | toast 手动关闭 | 点击 X 立即消失 | ✅ |
| 7 | 切换会话 toast 消失 | 不再显示上一条会话的错误 | ✅ |

### 5.3 控制台警告清理

| # | 检查项 | 结果 |
|---|--------|:--:|
| 1 | `[Vue Router warn] <router-view> can no longer be used...` | ✅ 已消除 |

---

## 六、关键决策记录

| # | 决策 | 结论 | 理由 |
|---|------|------|------|
| 1 | WebSocket 初始化时机 | 页面加载时（`onMounted`） | 避免第一条消息因连接未就绪而失败 |
| 2 | 后端离线时的消息处理 | 本地保留 + toast 提示 | 不丢失用户输入，同时让用户感知到服务端异常 |
| 3 | Toast 触发机制 | 命令式（`handleSend` 中检查 `wsConnected`） | 避免 Vue watch 同值不触发的坑 |
| 4 | subscribe() 调用位置 | 仅在 `handleConnectionChange` 中 | STOMP v7 不允许在连接建立前调用 subscribe |
| 5 | 会话切换状态清理 | 在 `selectConversation` 中集中清理 | `isLoading`/`streamingMessageId`/`error` 是全局状态，切换时必须重置 |
| 6 | MessageInput 清空策略 | 添加 `:key="conversationId"` | 组件重建方式最简洁，无需修改 MessageInput 内部逻辑 |
| 7 | 预检方案取舍 | 放弃 HTTP 预检，直接连接 WebSocket | CORS 策略阻止跨域 fetch，SockJS 不受此限制 |

---

## 七、技术要点

### STOMP v7 与之前版本的关键差异

`@stomp/stompjs` v7 中 `subscribe()` 必须在 CONNECTED 帧之后调用，不再支持预排队。正确的初始化流程：

```
client.activate()
  → SockJS 异步握手
    → onConnect 回调
      → handleConnectionChange(true)
        → ws.subscribe(topic)  ← 必须在此处订阅
```

### SockJS 与 CORS

SockJS 使用 WebSocket 升级请求（HTTP 101 Switching Protocols），不受浏览器 CORS 策略限制。但 `fetch()`/`XMLHttpRequest` 到不同端口仍受 CORS 约束。这意味着：
- WebSocket 连接后端：不需要 CORS 配置
- HTTP API 调用后端（REST）：需要 CORS 配置

---

## 八、产物清单

| 文件 | 操作 | 行数变化 | 说明 |
|------|------|---------|------|
| `frontend/src/components/chat/ChatWindow.vue` | 修改 | -130 | 删除 simulateResponse、新增 toast、MessageInput key |
| `frontend/src/views/ChatView.vue` | 修改 | +1 | onMounted 中调用 initWebSocket |
| `frontend/src/stores/chat.ts` | 修改 | +17 | sendMessage 重构、initWebSocket 修复、状态清理 |
| `frontend/src/App.vue` | 修改 | +6 | RouterView slot props 语法 |

### 代码统计

| 指标 | 数值 |
|------|------|
| 修改文件 | 4 |
| 新增代码行 | 71 |
| 删除代码行 | 153 |
| 净减少代码行 | 82 |

---

## 九、后续任务依赖

| 任务 | 前置条件 | 关联 |
|------|----------|------|
| 端到端 AI 回复流（加载动画→逐字渲染） | 后端 WebSocketController + AgentGatewayService（A5/A6） | 消息已发出，后端处理器为空，无回复 |
| 会话列表、Agent 列表从 REST API 加载 | 后端 ConversationController、完善 AgentController（A5） | API 调用会因后端缺失端点而静默降级 |
| ChatList 创建新会话 | 后端 POST /conversations 端点 | 当前返回 404，走本地 fallback |
