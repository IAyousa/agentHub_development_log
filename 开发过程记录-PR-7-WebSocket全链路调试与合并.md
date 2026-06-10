# 开发过程记录 — PR #7 代码提取、WebSocket 全链路调试与合并

> **日期**: 2026-06-05\
> **关联分支**: `review/pr-7-liu-weizhi`（审查） → `dev`（提取后直接创建）\
> **PR 作者**: 刘伟智（Liu-Weizhi1219），fork 仓库 → `IAyousa/Agenthub_demo`\
> **PR 概要**: A4-A7 任务合集，50 个文件变更（+2465/-5112 行）

---

## 一、背景

### 1.1 PR #7 是什么

成员刘伟智从 fork 仓库 `Liu-Weizhi1219/Agenthub_demo:dev` 向主仓库 `IAyousa/Agenthub_demo:dev` 提交了 PR #7。变更范围覆盖全栈 50 个文件：Java 后端新增 Controller/Service、Python Agent 服务修改、前端修改、文档修改。

### 1.2 当前项目状态

在审查 PR #7 之前，`dev` 分支已完成：
- **前端 C2-C7**：REST API 封装（conversation.ts / agent.ts）、STOMP WebSocket 客户端（wsClient.ts）、chatStore 重构、ChatWindow 真实通信接入
- **后端 PR #4/#5**：Artifact 管理（Entity + Service + Controller）+ Agent 管理 REST 控制器
- **后端 Service 层**：ConversationService、MessageService、AgentGatewayService、ArtifactService 全部就绪
- **文档**：5 份技术框架文档 + CLAUDE.md 全面更新

**MVP 缺失模块**：ConversationController、MessageController、WebSocketSessionManager、WebSocketController handler 实现

---

## 二、PR #7 代码审查

### 2.1 审查方法

不从 PR 分支直接合并（避免覆盖已有代码），而是：

1. 添加 PR 作者的 fork 为远程仓库
2. 拉取 PR 分支到本地，创建审查分支 `review/pr-7-liu-weizhi`
3. 逐文件审查，将有用代码与有害变更分离
4. 提取有价值模块，在当前 `dev` 分支上重新创建（修复后）

```bash
# 添加远程 + 拉取 PR 分支
git remote add Liu-Weizhi1219 https://gitee.com/Liu-Weizhi1219/Agenthub_demo.git
git fetch Liu-Weizhi1219 dev
git checkout -b review/pr-7-liu-weizhi Liu-Weizhi1219/dev

# 审查完成后切回 dev，直接在 dev 上创建修复版
git checkout dev
```

### 2.2 审查结果：四分类

| 分类 | 内容 | 决策 |
|------|------|:--:|
| ✅ 可提取 | ConversationController（257 行）、MessageController（148 行）、WebSocketSessionManager（105 行）、WebSocketController 扩展 | 修复后直接创建 |
| ❌ 架构冲突 | Python `agent-service/app/db/` — 给无状态网关加数据库 | 拒绝 |
| ❌ 误删除 | AgentController、ArtifactController、ArtifactService、前端 API 层（agent.ts/conversation.ts/wsClient.ts） | 拒绝（PR 基于旧代码快照） |
| ➖ 无关 | DeepSeekAdapter、前端 UI 修改、文档修改 | 拒绝（非 MVP 需求） |

### 2.3 四个提取模块的修复清单

**ConversationController.java** — 3 处修复：
1. `@RequestMapping("/api/conversations")` → `@RequestMapping("/conversations")`（去掉 `/api` 前缀，对齐 API 契约）
2. `new ResponseEntity<>(body, status)` → `ResponseEntity.status(status).body(body)`（风格统一）
3. `HashMap` → `LinkedHashMap`（字段顺序稳定）

**MessageController.java** — 4 处修复：
1. `@RequestMapping("/api")` → 去掉 `/api` 前缀
2. `PUT /messages/{id}/pin` → `PUT /conversations/{conversationId}/messages/{messageId}/pin`（层级路径 + 归属校验）
3. Content 硬编码 `language: "javascript"` → ObjectMapper 动态解析 JSON
4. `HashMap` → `LinkedHashMap`

**WebSocketSessionManager.java** — 完全重写：
- 原始版本基于裸 WebSocket API（`WebSocketSession` + `TextMessage`）
- 项目使用 STOMP over WebSocket（`SimpMessagingTemplate`）
- 重写为：`ConcurrentHashMap<String, String>`（userId → sessionId）+ `SimpMessagingTemplate.convertAndSend()`

**WebSocketController.java** — 方法论正确但编译不过：
- 消息处理流程设计正确：保存消息 → 路由决策 → 构建上下文 → 调用 Agent → 流式推送
- 但 5 处编译错误：Service 方法签名不匹配、`SimpMessagingTemplate` 未使用、`WebSocketSessionManager` 接口错误
- 修复：对齐已有 Service 的方法签名 + 使用 STOMP 消息协议格式

### 2.4 为什么 PR 包含了大量误删除

**根因**：Fork 分支严重落后于上游 `dev`。

Git fork 不会自动同步——fork 者需要手动执行 `git fetch upstream && git merge upstream/dev`。两位 PR 作者（PR #6 的 zsy125 和 PR #7 的刘伟智）都没有在提交前同步上游，导致 PR diff 中包含大量"删除上游已有文件"的误操作。

这是团队协作流程中需要改进的点——应在 PR 提交前要求作者先 rebase/merge 上游最新代码。

---

## 三、端到端 WebSocket 消息链路调试

这是本次开发中最有价值的技术经验。从"消息发出无回复"到"Claude 流式回复正常显示"，经历了 8 轮调试。

### 3.1 链路架构回顾

```
浏览器 STOMP → WebSocketController → MessageService(DB) → AgentGatewayService(HTTP)
    → Python FastAPI → ClaudeAdapter(asyncio.subprocess) → claude CLI → stdout
    → Python SSE 流 → AgentGatewayService(SSE parser) → SimpMessagingTemplate(STOMP)
    → 浏览器 MESSAGE 帧 → chatStore.handleToken → 打字机渲染
```

### 3.2 调试轮次

**第 1 轮：会话不存在**

- 现象：`IllegalArgumentException: 会话不存在: conv_1780652629021`
- 根因：前端用本地生成的 ID（`conv_${Date.now()}`）发消息，但 ID 不在 H2 数据库中
- 修复：ConversationController 接受 `agentIds: []` 空数组（自动分配默认 Agent）+ WebSocketController 自动补建不存在的会话

**第 2 轮：Hibernate 懒加载异常**

- 现象：`LazyInitializationException: failed to lazily initialize a collection`
- 根因：STOMP handler 不在 Spring MVC 的 OpenEntityManagerInViewFilter 范围内，`Conversation.agents`（`@ManyToMany` 懒加载集合）在事务外访问失败
- 尝试修复：加 `@Transactional` 到 `handleUserMessage()`
- **发现**：`@Transactional` 在 STOMP `@MessageMapping` handler 上会导致方法完全不执行——Spring AOP 代理与 STOMP 消息分发机制冲突

**第 3 轮：STOMP 订阅话题不匹配**

- 现象：浏览器 WS 帧显示订阅了 `/topic/conversation.conv_frontend_001`，但发送消息用了新创建的 UUID `60f2a100-...`
- 根因：前端 `createConversation` 返回 REST API 的 UUID，但 `selectConversation(UUID)` 切换会话时 `ws.subscribe(UUID)` 可能被 try-catch 静默吞掉
- 验证方法：直接在默认 mock 会话（`conv_frontend_001`）发送消息——话题一致
- **教训**：调试时先从最简单的场景开始，排除变量

**第 4 轮：后端 handler 未被调用**

- 现象：Java 日志没有任何 "Processing message" 输出
- 根因：`@Transactional` 注解阻止了 STOMP handler 的方法调用（见第 2 轮）
- 修复：移除 `@Transactional`，`resolveAgentType` 简化为 MVP 硬编码返回 `"claude_code"`

**第 5 轮：handler 被调用了，但 Agent SSE 回调不触发**

- 现象："Processing message" + "User message saved" 出现，但无 "Pushing token" 日志
- 根因：**`bodyToFlux(String.class)` 按网络 buffer 分块，不是按行分**——Python 的 SSE 响应被 Netty 一次读完，多个 `data:` 行合并成一个字符串
- 修复：`chunk.split("\n")` 先按行拆分，再逐行解析

**第 6 轮：SSE 数据到了，但 `data:` 前缀丢失**

- 现象：SSE chunk 内容是 `{"token":"你好！...","finish":false,...}` ——没有 `data: ` 前缀
- 根因：Python `StreamingResponse` 传输中 `data: ` 前缀丢失（Starlette/FastAPI 的 SSE 处理机制）
- 修复：Java 解析器改为兼容模式——同时支持 `data: {...}` 和纯 JSON `{...}` 两种格式

**第 7 轮：Claude CLI 在 Python 子进程中的执行差异**

- 现象：终端直接运行 `claude -p "hello"` 正常，但 Python adapter 子进程无输出
- 根因：adapter 的 stderr 被后台任务静默读取后丢弃，无法看到 CLI 的认证错误
- 修复：stderr 内容改为 log 输出（而非静默丢弃），便于排查认证/PATH 问题
- 最终确认：`claude` CLI 在子进程中正常工作，输出内容经 SSE → STOMP 完整到达前端

**第 8 轮：测试代码清理**

- 移除 `[STOMP delivery OK]` 即时推送测试
- 移除 Python/Java 调试 print/log
- 保留关键节点 INFO 日志：消息到达 → DB 写入 → Agent 调用完成 → 回复长度汇总

---

## 四、关键 Bug 与技术经验

### 4.1 `@Transactional` 与 STOMP handler 不兼容

```java
// ❌ 在 @MessageMapping 方法上加 @Transactional 会导致方法不被调用
@Transactional
@MessageMapping("/chat.send")
public void handleUserMessage(@Payload SendMessageRequest request) { ... }

// ✅ 正确做法：不加 @Transactional，在需要事务的方法内部调用 Service 层
@MessageMapping("/chat.send")
public void handleUserMessage(@Payload SendMessageRequest request) {
    messageService.saveMessage(...);  // Service 方法自带 @Transactional
}
```

### 4.2 WebFlux `bodyToFlux(String.class)` 的陷阱

```java
// ❌ bodyToFlux(String.class) 按网络 buffer 分块，不是按行。
// 小响应（<MTU）会被一次读完，多个 SSE 行合并成一个 String。
.bodyToFlux(String.class)
.doOnNext(line -> {
    if (!line.startsWith("data: ")) return;  // 只匹配第一行！
})

// ✅ 正确：先按 \n 拆行，再逐行处理
.bodyToFlux(String.class)
.doOnNext(chunk -> {
    for (String rawLine : chunk.split("\n")) {
        String line = rawLine.trim();
        // 需兼容有/无 "data: " 前缀两种格式
    }
})
```

### 4.3 SSE `data:` 前缀的跨框架差异

Python FastAPI `StreamingResponse` 传输 SSE 时，`data: ` 前缀可能在传输层丢失。Java 解析器不应严格要求 `"data: "` 前缀，而应直接尝试解析 JSON 内容。

### 4.4 调试策略

| 策略 | 说明 |
|------|------|
| 分而治之 | 先验证 STOMP 推送能否到达（即时 ping），再验证 SSE 解析，最后串联完整链路 |
| 逐层打桩 | 在每一层加日志/即时响应，快速定位故障层 |
| 最小场景 | 先用默认 mock 会话（ID 固定）测试，排除前端状态变化 |
| 原始数据 | 加 `log.info` 打印实际收到的字节内容，而非猜测格式 |

---

## 五、最终产物

### 5.1 新建文件（3 个）

| 文件 | 行数 | 说明 |
|------|------|------|
| `backend-java/.../controller/ConversationController.java` | 189 | 7 个 REST 端点：CRUD + Agent 关联 + 历史消息 |
| `backend-java/.../controller/MessageController.java` | 100 | 分页消息列表 + 置顶（含会话归属校验） |
| `backend-java/.../service/WebSocketSessionManager.java` | 49 | STOMP 版会话管理 |

### 5.2 修改文件（2 个）

| 文件 | 变化 | 说明 |
|------|------|------|
| `WebSocketController.java` | +75 行 | 从空壳到 4 步完整 handler |
| `ConversationController.java` | agentIds 改为可选 | 空数组自动默认 claude_code |

### 5.3 调试中修复的已有 Bug

| 文件 | Bug | 修复 |
|------|-----|------|
| `AgentGatewayService.java` | SSE 按网络 buffer 分块导致多行合并 | `split("\n")` + 兼容无前缀 JSON |
| `claude_adapter.py` | stderr 被静默丢弃，认证错误无法排查 | stderr 改为 log 输出（保留最后 500 字符） |

### 5.4 代码统计

| 指标 | 数值 |
|------|------|
| 新建文件 | 3 |
| 修改文件 | 4 |
| 新增代码行 | ~420 |
| 删除代码行 | ~20 |

---

## 六、提交与推送

```
79a1046 feat(backend): 实现会话/消息REST控制器并填充WebSocket消息处理链路（PR #7）

- 新增 ConversationController（7端点：CRUD + Agent关联 + 历史消息）
- 新增 MessageController（分页消息列表 + 置顶/取消置顶含会话归属校验）
- 新增 WebSocketSessionManager（STOMP版，基于 SimpMessagingTemplate）
- WebSocketController 填充 handler：保存消息→路由决策→构建上下文→Agent调用→STOMP推送
- 修复 AgentGatewayService SSE 解析：bodyToFlux 分块拆分 + 兼容无 data: 前缀 JSON
- 修复 ClaudeAdapter stderr 被静默丢弃问题
```

6 个文件，+558/-36 行。推送至 `origin/dev`。

---

## 七、后续任务

| 任务 | 状态 | 说明 |
|------|:--:|------|
| ConversationController + MessageController | ✅ 完成 | REST API 全部就绪 |
| WebSocketSessionManager | ✅ 完成 | STOMP 会话管理 |
| WebSocketController handler | ✅ 完成 | 4 步完整链路可跑通 |
| 前端接入 Conversation REST API | ⬜ 待做 | loadConversationList/loadMessages 需在 onMounted 调用 |
| 前端修复 STOMP 订阅不匹配 | ⬜ 待做 | selectConversation 中 ws.subscribe 可能静默失败的 bug |
| agentType 动态路由 | ⬜ P2 | MVP 硬编码 claude_code，后续从 conversation.type 查 |
| /internal/artifacts 端点 | ⬜ P1 | Python Agent 生成代码后直接上传产物 |
| 群聊 Orchestrator | ⬜ P2 | 多 Agent 调度 |
