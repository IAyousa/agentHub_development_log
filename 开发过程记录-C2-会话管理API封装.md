# 开发过程记录 — C2 会话管理 API 封装

> **日期**: 2026-06-03\
> **关联分支**: `dev`\
> **所属模块**: frontend\
> **Commit**: `135ce37`

## 一、背景

前端此前完全使用 mock 数据，ChatList 组件展示的是硬编码的假会话。需要建立前端与后端 REST API 的通信基础。C2 是 C2-C5 前端通信层搭建系列的第一步，负责会话管理相关的 API 封装。

## 二、技术实现

### 2.1 新增文件

`frontend/src/api/conversation.ts` — 封装 8 个会话 REST 接口：

| 接口 | 方法 | 端点 | 说明 |
|------|------|------|------|
| `getConversations` | GET | `/conversations` | 获取会话列表，支持 `archived` 筛选 |
| `getConversation` | GET | `/conversations/{id}` | 获取单个会话详情（含 Agent 关联） |
| `createConversation` | POST | `/conversations` | 创建新会话，含类型和 Agent 绑定 |
| `updateConversation` | PUT | `/conversations/{id}` | 更新会话标题/归档状态 |
| `deleteConversation` | DELETE | `/conversations/{id}` | 删除会话 |
| `getConversationAgents` | GET | `/conversations/{id}/agents` | 获取会话关联的 Agent 列表 |
| `addAgentToConversation` | POST | `/conversations/{id}/agents` | 向会话添加 Agent |
| `removeAgentFromConversation` | DELETE | `/conversations/{id}/agents/{agentId}` | 从会话移除 Agent |
| `getConversationMessages` | GET | `/conversations/{id}/messages` | 分页获取历史消息 |
| `pinMessage` | PUT | `/conversations/{id}/messages/{msgId}/pin` | 置顶/取消置顶消息 |

### 2.2 TypeScript 类型定义

```ts
// API 层类型（与 Store 层解耦，便于替换后端）
interface ConversationListItem {
  id: string; title: string; type: 'direct' | 'group';
  agentIds: string[]; lastMessageAt?: string;
  isArchived?: boolean; createdAt?: string;
}

interface MessageItem {
  id: string; conversationId: string; role: 'user' | 'agent';
  type: 'text' | 'code' | 'artifact_preview';
  content: string; agentId?: string; metadata?: Record<string, unknown>;
  pinned?: boolean; createdAt: string;
}

interface CreateConversationRequest {
  title: string; type: 'direct' | 'group'; agentIds: string[];
}
```

### 2.3 技术要点

- 基于已有的 `api/index.ts` Axios 实例（baseURL `localhost:8080`，30s 超时）
- 使用 TypeScript 泛型 `apiClient.get<T>(url)` 提供编译期类型安全
- API 层与 Pinia Store 层的类型独立定义，后端接口变更时只需更新 API 层
- 所有端点对齐 API 契约文档 v1.0 第 2.2 节

## 三、同步更新

同步更新 `CLAUDE.md`：
- 从中文改为全英文
- 修正 8 处与设计文档/实际代码的差异项
- 更新前端完成度评估

## 四、产物清单

| 文件 | 操作 | 行数 |
|------|------|------|
| `frontend/src/api/conversation.ts` | 新增 | +260 |
| `CLAUDE.md` | 重写 | +230/-130 |
