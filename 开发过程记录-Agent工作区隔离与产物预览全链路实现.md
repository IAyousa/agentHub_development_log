# 开发过程记录 — Agent 工作区隔离与产物预览全链路实现

> **日期**: 2026-06-05 ~ 2026-06-06\
> **关联分支**: `dev`\
> **涉及范围**: Agent 工作区隔离、`/internal/artifacts` 端点、Python 产物自动检测上传、前端预览卡片渲染

---

## 一、背景

### 1.1 现状缺口

在 PR #6 和 PR #7 合并后，聊天链路（发消息→Agent 流式回复→前端打字机渲染）已完整跑通。但产物管线完全断裂：

- Agent 生成的代码只以纯文本形式出现在聊天流中
- 无法像 ChatGPT Canvas / Claude Artifacts 那样点击预览
- Agent 的工作目录是 agent-service 启动目录，可能意外读取 AgentHub 项目源码

### 1.2 三个已有能力

在本次开发之前，以下模块已经就绪但未接通：

| 已有能力 | 位置 | 状态 |
|----------|------|:--:|
| Java ArtifactService/Controller（CRUD + 文件存储） | backend-java | ✅ 端点可用 |
| 前端 ArtifactWindow + ArtifactSandbox（iframe 预览） | frontend | ✅ UI 就绪 |
| Python ClaudeAdapter（Agent 生成代码） | agent-service | ✅ 文本输出 |

### 1.3 断裂的链路

```
Agent 生成代码 → ??? → 文件存储 → WebSocket 推 preview_card → 前端 iframe 预览
    ↑            ↑          ↑              ↑                    ↑
  已有能力     完全空白   已有能力       没有代码              已有UI
```

---

## 二、架构讨论：Agent 工作区隔离

### 2.1 问题发现

在开始产物开发前，用户提出了一个关键安全问题：

**"Agent 接收到用户请求后，现在是不是会把我们正在实现的 AgentHub 项目整体读取一遍后再生成内容？会不会导致 Agent 输出的代码污染项目？"**

追踪代码发现：ClaudeAdapter 启动子进程时 `cwd` 为 `"."`（`AGENT_WORKING_DIRECTORY` 默认值），即 agent-service 目录。Agent 确实可能读取项目源码作为上下文，存在以下风险：

- **浪费 token**：Agent 读取无关的 AgentHub 源码后再回复用户请求
- **污染风险**：如果 Agent 被赋予写文件权限，可能修改项目源码
- **会话间无隔离**：不同会话共享同一工作目录

### 2.2 设计方案

每个会话独立工作区，与 AgentHub 项目物理隔离：

```
agent-service/
├── agent_workspaces/                    ← 新建根目录
│   ├── conv_frontend_001/               ← 会话级隔离
│   │   └── (Agent 在此自由读写)
│   └── {其他会话ID}/
├── adapters/
└── config.py
```

Agent 子进程的 `cwd` 指向 `./agent_workspaces/{conversationId}/`，对 AgentHub 源码完全不可见。

### 2.3 实施（commit `46e0713`）

| 文件 | 改动 |
|------|------|
| `config.py` | 新增 `AGENT_WORKSPACE_ROOT = "./agent_workspaces"` |
| `AgentGatewayService.java` | `sendToAgent()` 签名新增 `workingDirectory` 参数 |
| `WebSocketController.java` | 传入 `"./agent_workspaces/" + conversationId` |
| `messages.py` | 收到 workingDirectory 时 `os.makedirs()` 按需创建 |
| `.gitignore` | 排除 `agent_workspaces/` |

---

## 三、`/internal/artifacts` 端点实现

### 3.1 设计方案

这是产物管线的核心桥梁——Python Agent 服务通过 HTTP 调此端点，将生成的代码文件存入 Java 后台。

**请求格式**（JSON）：
```json
{
  "conversationId": "conv_abc",
  "messageId": "msg_xyz",
  "filename": "index.html",
  "content": "<html>...</html>",
  "contentType": "text/html"
}
```

**端点行为**：
1. 接收 JSON → 写磁盘文件 → DB 记录
2. 通过 `SimpMessagingTemplate` 推 `preview_card` 到 `/topic/conversation.{id}`
3. 前端收到后渲染可点击预览卡片

### 3.2 实施（commit `0f23a46`）

| 文件 | 操作 | 说明 |
|------|:--:|------|
| `InternalArtifactRequest.java` | 新建 | DTO：conversationId / messageId / filename / content / contentType |
| `ArtifactService.java` | 修改 | 新增 `saveFromContent()`：文本内容直接 `Files.writeString()` 写盘 |
| `ArtifactController.java` | 修改 | 新增 `POST /internal/artifacts`：存文件 → SimpMessagingTemplate 推 preview_card |

### 3.3 调试：前端收到卡片但无法预览

**现象**：curl 请求后前端出现预览卡片，但点击后 iframe 显示 "hello.html" 而非渲染的 HTML 页面。

**根因**：WebSocket 推送的 `content` 字段是文件名，前端 `showArtifact()` 将 `content` 当作待渲染代码传入 ArtifactSandbox。

**修复**：`content` 改为实际文件内容，`metadata` 补充 `title` 和 `language`：

```java
previewCard.put("content", req.getContent());  // 实际 HTML 内容，非文件名
metadata.put("title", dto.getFilename());
metadata.put("language", inferLanguage(dto.getFilename()));
```

`inferLanguage()` 根据文件扩展名推断（.html→html, .js→javascript, .css→css 等）。

---

## 四、Python 产物自动检测上传

### 4.1 问题

`/internal/artifacts` 端点就绪后，需要有人主动调用它。手动 curl 测试通过，但需实现自动化——Agent 生成代码后自动检测并上传。

### 4.2 关键发现：Python 没有 conversationId

Java 发给 Python 的请求体只有 `{agentType, systemPrompt, context, stream, workingDirectory}`。Python 无法知道当前请求属于哪个会话，也就无法调 `/internal/artifacts`（需要 `conversationId` 参数）。

**修复**：在 Java 请求体、Python 模型、函数签名中全线补齐 `conversationId`。

### 4.3 实施（commit `1528d9b`）

**新建 `artifact_uploader.py`**：

| 函数 | 职责 |
|------|------|
| `detect_code_blocks()` | 正则 `"`(\w*)\n(.*?)"` 匹配 Markdown 代码块，去重 |
| `_LANGUAGE_MAP` | 18 种语言标签 → 文件扩展名 + MIME 类型映射 |
| `upload_artifact()` | `httpx.AsyncClient` POST `/internal/artifacts` |
| `detect_and_upload()` | 主入口：检测 → 逐个上传 |

**修改 `messages.py`**：

```python
# 在 event_generator 中积累完整回复
full_text: list[str] = []
# ... 每条 msg_chunk: full_text.append(delta)

# Agent 结束后，后台异步上传
if conversation_id and message_id:
    asyncio.ensure_future(
        detect_and_upload("".join(full_text), conversation_id, message_id)
    )
```

### 4.4 测试验证

用户发送："帮我写一个HTML页面，用代码块```html包裹，标题是Hello AgentHub，背景色深蓝，文字白色居中显示"

结果：
- 聊天区显示 Markdown 代码块
- **同时生成可点击的预览卡片**
- 点击后 ArtifactSandbox iframe 渲染出深蓝背景 + 白色文字的 HTML 页面

**关于同时显示源码和预览卡片的讨论**：用户认为 Agent 先输出 Markdown 代码再生成预览卡片是否有必要。结论是主流 AI 工具均采用这种模式（源码可读可复制 + 预览卡片一键渲染），MVP 阶段保持现状。

---

## 五、最终产物

### 5.1 全链路架构

```
用户发消息 "帮我写一个HTML页面"
  ↓
前端 STOMP → Java WebSocketController
  → AgentGatewayService.POST /api/agent/chat
  → {agentType, systemPrompt, context, conversationId, workingDirectory}
  ↓
Python messages.py
  → os.makedirs("./agent_workspaces/conv_xxx", exist_ok=True)
  → ClaudeAdapter("claude -p", cwd="./agent_workspaces/conv_xxx")
  → Agent 在隔离工作区中生成代码（stdout）
  ↓
SSE 流式返回 token + 积累 full_text[]
  ↓ (Agent 结束后)
Python artifact_uploader.detect_and_upload()
  → 正则提取代码块（```html, ```css, ```js 等）
  → httpx POST /internal/artifacts × N
  ↓
Java ArtifactController.internalUpload()
  → Files.writeString() 写磁盘
  → DB 记录
  → SimpMessagingTemplate.convertAndSend() push preview_card
  ↓
前端 ChatMessage 渲染预览卡片 → 点击 → ArtifactWindow
  → ArtifactSandbox <iframe srcdoc="...">
```

### 5.2 提交记录

```
1528d9b feat(artifact): Python端Agent产物自动检测上传，接线完整预览链路
0f23a46 feat(backend): 新增/internal/artifacts内部端点，接线Python产物上传→存储→前端预览链路
46e0713 feat(agent): 实现Agent会话级工作区隔离，防止项目源码污染
```

### 5.3 代码统计

| 指标 | 数值 |
|------|------|
| 新建文件 | 3（InternalArtifactRequest.java / artifact_uploader.py / __init__.py） |
| 修改文件 | 7（Java 3 + Python 3 + .gitignore） |
| 新增代码 | ~280 行 |
| 修改代码 | ~15 行 |

---

## 六、经验总结

### 6.1 链路调试要逐段验证

产物管线的开发采用了"分段建造、逐段测试"策略：

1. 先做 Java 端点 → curl 测试
2. 再做 Python 上传模块 → curl 模拟测试
3. 最后接前端预览 → 全栈测试

每段独立可验证，出问题时能快速定位故障层。

### 6.2 元数据传递要全线补齐

Python 没有 `conversationId` 是在实施产物上传时才发现的问题。如果最初设计 Java→Python 请求体时就把会话上下文信息（conversationId、userId 等）作为标准字段传入，就不会出现这个"想上传但不知道属于哪个会话"的尴尬。

**教训**：跨服务通信的字段设计要向前看——当前功能可能不需要某个字段，但后续功能很可能需要。

### 6.3 安全考量前置

用户提出的"Agent 会不会读取项目源码"是一个典型的安全问题。虽然 Claude CLI 的 `-p` 模式不主动修改文件，但工作目录隔离是一个低成本高收益的防御措施：

- 隔离成本：4 个文件 +15 行代码
- 防御价值：消除项目源码被意外读取/修改的全部风险

这种防御措施应该尽早做，而非等到问题发生后再补救。

### 6.4 源码展示 vs 预览卡片不是二选一

用户质疑"Agent 先输出 Markdown 代码再生成预览卡片"是否有必要。讨论后确认：两者服务不同场景——源码用于阅读、复制、审查，预览卡片用于快速渲染。主流工具（ChatGPT Code Interpreter、Claude Artifacts）均采用这种双轨展示。
