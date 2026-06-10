# 开发过程记录 — C3 Agent 管理 API 封装

> **日期**: 2026-06-03\
> **关联分支**: `dev`\
> **所属模块**: frontend\
> **Commit**: `93820b6`

## 一、背景

C2 完成了会话管理 API 封装，前端还需要 Agent 维度的 REST 接口——前端需要展示可用 Agent 列表（ChatList 的 Agent 标签）、创建会话时选择 Agent。

## 二、技术实现

### 2.1 新增文件

`frontend/src/api/agent.ts` — 封装 3 个 Agent REST 接口：

| 接口 | 方法 | 端点 | 说明 |
|------|------|------|------|
| `getAgents` | GET | `/agents` | 获取所有 Agent 列表（含 capabilities、avatarUrl、isBuiltin） |
| `getAgent` | GET | `/agents/{id}` | 获取单个 Agent 详情（额外含 systemPrompt、createdAt） |
| `createAgent` | POST | `/agents` | 创建自定义 Agent，含 type 校验 |

### 2.2 TypeScript 类型定义

```ts
type AgentType = 'claude_code' | 'codex' | 'custom'

interface AgentListItem {
  id: string; name: string; type: AgentType;
  avatarUrl?: string; capabilities?: string[];
  isBuiltin: boolean;
}

interface AgentDetail extends AgentListItem {
  systemPrompt: string; createdAt: string;
}

interface CreateAgentRequest {
  name: string; type: AgentType;
  systemPrompt?: string; avatarUrl?: string;
  capabilities?: string[];
}
```

### 2.3 关键设计

- `AgentDetail` 通过 `extends AgentListItem` 复用字段，列表接口不含 `systemPrompt`（对齐 API 契约 2.3 节的安全设计——详情接口才返回系统提示词）
- `type` 使用字符串字面量联合类型而非 `string`，编译期即可捕获非法值
- `capabilities` 数组从后端 JSON 字符串自动反序列化为 `string[]`

## 三、产物清单

| 文件 | 操作 | 行数 |
|------|------|------|
| `frontend/src/api/agent.ts` | 新增 | +99 |
