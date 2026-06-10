# 开发过程记录 — C4 WebSocket STOMP 客户端封装

> **日期**: 2026-06-03\
> **关联分支**: `dev`\
> **所属模块**: frontend\
> **Commit**: `9b196e4`

## 一、背景

AgentHub 的实时通信基于 STOMP over WebSocket 协议（API 契约 §3）。前端需要一个封装的客户端来管理连接生命周期、消息收发、流式 token 分发和自动重连，同时保持传输层与 Pinia Store 解耦。

## 二、技术实现

### 2.1 新增文件

`frontend/src/websocket/wsClient.ts` — 327 行，基于 `@stomp/stompjs` v7 + `sockjs-client` 的单例客户端。

### 2.2 架构设计：回调注入模式

```ts
interface WsCallbacks {
  onToken: (chunk: MessageChunk) => void
  onAgentSwitch: (agentId: string, agentName: string) => void
  onConnectionChange: (connected: boolean) => void
  onError: (error: string) => void
}

// 全局单例
let instance: WsClient | null = null

export function getWsClient(callbacks: WsCallbacks): WsClient {
  if (!instance) instance = new WsClient(callbacks)
  return instance
}
```

Store 通过传入回调函数与 WebSocket 客户端交互，传输层不依赖 Pinia、不依赖 Vue——纯 TypeScript 类。

### 2.3 核心功能

**连接生命周期管理**：
```ts
connect(): void {
    if (this.client.active) return
    this.client.activate()  // SockJS → STOMP 异步握手
}

disconnect(): void {
    this.client.deactivate()
}
```

**消息发送**：
```ts
sendMessage(payload: SendMessagePayload): void {
    if (!this.client.connected) {
      this.callbacks.onError('未连接到服务器，请稍后重试')
      return
    }
    this.client.publish({
      destination: '/app/chat.send',
      body: JSON.stringify(payload),
    })
}
```

**STOMP 订阅管理**：
```ts
subscribe(conversationId: string): void {
    const topic = `/topic/conversation.${conversationId}`
    const subscription = this.client.subscribe(topic, (message) => {
      const body = JSON.parse(message.body)
      if (body.type === 'agent_switch') {
        this.callbacks.onAgentSwitch(body.agentId, body.agentName)
      } else {
        this.callbacks.onToken(body)
      }
    })
}
```

**自动重连与心跳**：
```ts
this.client = new Client({
    webSocketFactory: () => new SockJS(WS_ENDPOINT),
    reconnectDelay: 5000,           // 断线 5 秒后重连
    heartbeatIncoming: 10000,        // 服务端心跳 10s
    heartbeatOutgoing: 10000,        // 客户端心跳 10s
    onConnect: () => { ... },        // 重连后自动恢复订阅
    onDisconnect: () => { ... },
    onStompError: (frame) => { ... },
})
```

### 2.4 技术要点

- **SockJS** 提供 WebSocket 降级能力——浏览器不支持 WS 时自动退至 XHR 长轮询
- **全局单例** 确保 ChatView 和 OfficeView 共享同一连接，避免多连接资源浪费
- **STOMP 目的地**：`/app/chat.send`（发送）、`/topic/conversation.{id}`（订阅）
- **`agent_switch` 事件**：后端可能在流式回复中切换发言人 Agent，客户端通过 message body 的 `type` 字段区分 token 数据和切换事件

## 三、产物清单

| 文件 | 操作 | 行数 |
|------|------|------|
| `frontend/src/websocket/wsClient.ts` | 新增 | +327 |
