# 开发过程记录 — Agent 选择/切换 UI 与适配器 Bug 修复

> **日期**: 2026-06-06\
> **关联分支**: `dev`\
> **涉及范围**: Agent 下拉选择器、Codex 适配器 Bug 修复、错误消息人性化、工作区隔离强化

---

## 一、背景

### 1.1 现状

项目在完成产物预览全链路后，聊天功能已完整跑通。但所有消息都硬编码使用 `agent_claude_001`（Claude Code）：

- Java `WebSocketController` 第 50 行：`agentRepository.findById("agent_claude_001")`
- 前端 `sendMessage()` 只传 `{conversationId, content}`
- `SendMessageRequest.agentId` 字段在 Java DTO 中存在但从未被读取
- ChatWindow header 只显示会话标题，没有任何 Agent 选择 UI

Codex 适配器（`codex_adapter.py`）自 PR #3 创建以来从未被调用（因为只硬编码用 Claude Code），存在两个遗留 Bug：

- 第 85 行：`self._build_prompt()` 应为 `BaseAdapter.build_prompt()`（方法名拼写错误）
- 错误后同时 yield `msg_end`，导致 Java 端收到重复 finish 帧

### 1.2 目标

用户可以在聊天界面选择用哪个 Agent 回复消息，切换流程完整打通。

---

## 二、方案设计

### 2.1 数据流

```
用户点击 Agent 选择器 → 选 Codex
  ↓
store.selectedAgentId = "agent_codex_001"
  ↓
用户发送消息
  ↓
前端 sendMessage() → STOMP { conversationId, content, agentId: "agent_codex_001" }
  ↓
Java WebSocketController
  → 读取 request.getAgentId()
  → agentRepository.findById(agentId)
  → agentType = "codex", systemPrompt = agent.getSystemPrompt()
  → 调用 Python CodexAdapter
  ↓
Agent 回复 → 前端显示
```

### 2.2 UI 设计

ChatWindow header 右侧新增 Agent 下拉选择器：

```
┌─ ChatWindow header ──────────────────────────────────────┐
│ 🏠 New Chat                          [Claude Code ▾] [⋮] │
└──────────────────────────────────────────────────────────┘
```

### 2.3 改动文件

| # | 文件 | 改动 |
|---|------|------|
| 1 | `chat.ts` | 新增 `selectedAgentId` ref，`sendMessage` 传 `agentId` |
| 2 | `wsClient.ts` | `SendMessagePayload` 接口加可选 `agentId` |
| 3 | `ChatWindow.vue` | header 加 `<select>` Agent 下拉选择器 |
| 4 | `WebSocketController.java` | 读 `request.getAgentId()` 替代硬编码 |
| 5 | `SendMessageRequest.java` | 字段已存在，无需修改 |

---

## 三、测试与 Bug 修复

### 3.1 第一轮：Codex 适配器 `_build_prompt` 错误

**现象**：选择 Claude Code 正常回复，选择 Codex 时 Python 崩出异常。

```
AttributeError: 'CodexAdapter' object has no attribute '_build_prompt'
```

**根因**：`codex_adapter.py` 第 85 行调用 `self._build_prompt()`，但基类方法名为 `BaseAdapter.build_prompt()`（静态方法，无下划线前缀）。该 Bug 从 PR #3 以来一直潜伏，因从未调用 Codex 适配器而未被发现。

**修复**：`self._build_prompt` → `BaseAdapter.build_prompt`

### 3.2 第二轮：Codex 失败后 fallback 消息重复

**现象**：Codex CLI 未认证时，前端出现重复的 fallback 消息。

**追踪根因**：Python 适配器在错误后同时 yield `error` 和 `msg_end` 两个 chunk，messages.py 将两者都转为 SSE finish 帧。Java 收到两帧 finish，第一帧触发 fallback，第二帧再次触发 fallback。

```
Python: yield error → yield msg_end
  ↓
Messages.py: SSE finish #1 (含error) → SSE finish #2
  ↓
Java: fallback #1 + fallback #2（重复）
```

**修复**：两个适配器均加 `has_error` 标志，错误后跳过 `msg_end` yield。

### 3.3 第三轮：错误后再发空消息

**现象**：Codex 失败时，错误信息 + 一条空消息同时出现。

**根因**：WebSocketController 的回调中，error 检查在 finish 之后。`AgentToken.error(msg)` 同时设置了 `error` 和 `finish=true`，先进入 finish 分支（receivedTokens=false → 发空 fallback），再进入 error 分支（发 errorChunk）。

**修复**：error 检查移到 finish 之前，error 后 `return` 跳过 finish 逻辑。

### 3.4 第四轮：错误消息在前端以空气泡显示

**现象**：Codex 错误时前端只显示空白消息气泡，无文字。

**根因**：`errorChunk` 的 `messageType` 为 `"error"`，但 `ChatMessage.vue` 的模板只处理 `text`、`code`、`artifact_preview` 三种类型。`"error"` 类型走不到任何渲染分支。

**修复**：`errorChunk` 的 `messageType` 改为 `"text"`，错误消息作为普通文本气泡渲染。

### 3.5 第五轮：错误信息不适合用户阅读

**用户反馈**：不要显示 "OpenAI Codex CLI 异常退出（code=1）"，用户看不懂。

**修复**：新增 `friendlyErrorMessage()` 方法，根据错误关键词映射为用户友好的提示：

| 原始错误 | 映射后 |
|---------|--------|
| "未找到" | "XXX 未安装，请联系管理员配置该 Agent 的 CLI 工具。" |
| "超时" | "XXX 响应超时，请稍后重试。" |
| "异常退出" | "XXX 暂时无法使用，请检查 Agent 配置或稍后重试。" |

---

## 四、附带修复

### 4.1 系统提示词管理混乱

**发现**：`data.sql` 中 Agent 的 `system_prompt` 有值，Java 从 DB 读出传给 Python。但 Python 的 `SYSTEM_PROMPTS` 才是增强版（含角色定义、输出规范、工作区边界约束）。DB 有值时 Python 的 fallback 永远不会触发。

**修复**：`data.sql` 内置 Agent 的 `system_prompt` 置空，由 Python `SYSTEM_PROMPTS` 统一管理。增强提示词新增工作区边界约束："工作区是你唯一可以读写的目录，禁止访问工作区之外的任何文件"。

### 4.2 Agent 可跨越工作区读项目文件

**发现**：用户观察到 Agent 仍然能读取 `CLAUDE.md` 和 git 历史。

**分析**：`cwd` 只设置子进程的初始工作目录。Claude Code CLI 作为完整 Agent，可以使用文件系统工具导航到任意路径。工作区隔离是软约束（系统提示词 + cwd），不是硬沙箱（需 Docker/chroot）。

**结论**：MVP 阶段接受此已知限制，硬隔离留到 P1。

---

## 五、最终产物

### 5.1 代码变更

| 操作 | 文件 | 说明 |
|:--:|------|------|
| 修改 | `chat.ts` | `selectedAgentId` + `sendMessage` 传 `agentId` |
| 修改 | `wsClient.ts` | `SendMessagePayload.agentId` 字段 |
| 修改 | `ChatWindow.vue` | header Agent 下拉 `<select>` |
| 修改 | `WebSocketController.java` | 读 `request.agentId` + error/finish 顺序 + 友好错误 |
| 修改 | `codex_adapter.py` | `_build_prompt` typo + `has_error` 去重 |
| 修改 | `claude_adapter.py` | `has_error` 去重 |
| 修改 | `system_prompts.py` | 工作区边界约束 |
| 修改 | `data.sql` | `system_prompt` 置空 |

### 5.2 提交记录

```
096f494 feat(frontend): 实现Agent选择/切换UI，修复Codex适配器遗留bug
```

---

## 六、经验总结

### 6.1 未调用过的代码一定有 Bug

CodexAdapter 自 PR #3 创建后从未被调用（硬编码只用 Claude Code），两个 Bug（`_build_prompt` typo、重复 msg_end）潜伏数周。ClaudeAdapter 的第一个 Bug 也相同。

**教训**：代码写好后的第一步验证永远是"跑一遍"。即使是测试简单切换也能立即暴露问题。

### 6.2 技术错误信息要分层

"OpenAI Codex CLI 异常退出（code=1）"对开发者有用（日志），对用户没用（界面）。技术错误 → 用户友好描述的映射应该在最靠近用户的那层做。本次在 Java WebSocketController（消息路由的最后一步）做了这个映射。

### 6.3 配置分散导致维护困难

Agent 系统提示词同时存在于 `data.sql`（DB）和 `system_prompts.py`（Python 模板），两份数据内容不同，开发者不确定哪份生效。本次统一为 DB 置空、Python 模板兜底。

**教训**：每份配置数据应该只有一个权威来源。如果迫不得已需要两份，必须明确标注优先级。
