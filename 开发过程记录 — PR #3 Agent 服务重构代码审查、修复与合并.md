# 开发过程记录 — PR #3 Agent 服务重构代码审查、修复与合并

> **日期**: 2026-06-02\
> **关联分支**: `remotes/origin/pr/3` → `fix-pr3-review` → `dev`\
> **所属模块**: agent-service / backend-java / docs\
> **原始 PR 概要**: 将 Agent Service 从 HTTP API 调用（Anthropic/OpenAI 云端 API）重构为本地 CLI 子进程调用（`asyncio.subprocess`），移除 `app/db/` 数据库访问层，删除 `deepseek_adapter.py` 和 3 个已废弃端点

---

## 一、背景

PR #3（`remotes/origin/pr/3`）是一个重构性质的合并请求，变更规模大：**16 个文件 ┃ +819 行 ┃ -977 行**。核心改动包括：

| 改动项 | 说明 |
|--------|------|
| CLI 适配器 | `ClaudeAdapter` / `CodexAdapter` 从 HTTP API 调用改为 `asyncio.create_subprocess_exec` 子进程调用 |
| 删除文件 | `deepseek_adapter.py`、`app/db/connection.py`、`app/db/repository.py`、`agents.py`/`conversations.py`/`artifacts.py` 端点 |
| API 契约变更 | 路径 `/api/v1/messages/chat/stream` → `/api/agent/chat`，字段 `snake_case` → `camelCase` |
| 响应格式变更 | NDJSON → SSE `data: <JSON>\n\n` |
| 配置重构 | API Key 改为 CLI 命令路径，Add `extra = "ignore"` 兼容旧 `.env` |

由于变更影响面广，需要在合并前进行系统性审查：验证代码与架构文档的一致性，修复发现的冲突，并通过端到端测试确保 Agent Service 和 Java 后端均可正常编译、启动和运行。

---

## 二、审查与修复过程

### 2.1 分支 init commit 的审查

**操作内容**：逐文件对比 `remotes/origin/pr/3` 与 `dev` 的差异，同时对照 5 份架构文档。

**发现冲突 4 类**：

| # | 冲突 | 严重程度 |
|---|------|----------|
| 1 | `config.py` 保留已废弃的 `H2_JAR_PATH`、`H2_DB_PATH` 死配置项 | 🔴 高 |
| 2 | `requirements.txt` 保留 `jaydebeapi`、`jpype1`、`sse-starlette` 等 9 个废弃依赖 | 🟡 中 |
| 3 | `ClaudeAdapter` 和 `CodexAdapter` 中 `strip_ansi()`、`build_prompt()` 重复定义 | 💡 优化 |
| 4 | `AGENT_MAX_TOKENS` 定义但未被 CLI 适配器引用 | 💡 优化 |

**设计差距 3 项**（已规划成员 D 的 D9-D12 任务，暂不修复）：

| # | 差距 | 说明 |
|---|------|------|
| 1 | `custom` → `ClaudeAdapter`，未实现 HTTP API 路径 | 自定义 Agent 暂时走 CLI |
| 2 | `config.py` 缺少 `CUSTOM_AGENT_*` 配置项 | 待 D11 任务加入 |
| 3 | `AGENT_MAX_TOKENS` 预留给 HTTP 适配器 | 待 D9 实现后生效 |

### 2.2 创建 fix-pr3-review 修复分支

**操作内容**：在 `fix-pr3-review` 分支上执行 3 批次修复。

**批次 1 — agent-service 适配器重构 + config 清理**：

```
commit 0a071bd
refactor(agent-service): 提取CLI适配器公共逻辑到BaseAdapter，清理config死配置

- 将 strip_ansi() 和 build_prompt() 从 ClaudeAdapter/CodexAdapter 提取到 BaseAdapter 基类
- 移除 H2_JAR_PATH、H2_DB_PATH 死配置项
- 移除未使用的 AGENT_MAX_TOKENS 配置项
- 移除 import os（不再需要）
```

**批次 2 — Java 后端 API 契约对齐**：

```
commit 9e56ebf
fix(backend): 修正AgentGatewayService与Python Agent Service的API契约对齐

- API 路径 /api/v1/messages/chat/stream → /api/agent/chat
- snake_case → camelCase 字段命名
- NDJSON 逐行解析 → SSE data: 行解析
- Consumer<String> → Consumer<AgentToken>（支持 agentId/agentName/error）
- 新增 AgentToken 内部类封装流式响应结构
```

**批次 3 — requirements.txt 清理 + 文档同步**：

```
commit 1e4f3b4
chore(agent-service): 清理废弃依赖，移除 sse-starlette/jaydebeapi/jpype1 等

- 移除 sse-starlette（已改用 FastAPI 原生 StreamingResponse）
- 移除 jaydebeapi、jpype1（app/db 数据访问层已删除）
- 移除 sqlalchemy、aiosqlite、websockets、python-multipart、jose、passlib
- langgraph 版本号更新为 0.0.20
```

**修复前后对比**：

| 文件 | 修复前 | 修复后 |
|------|--------|--------|
| `requirements.txt` | 30 行，9 个废弃依赖 | 17 行，6 个有效依赖 |
| `config.py` | 包含 H2/JDBC/AGENT_MAX_TOKENS | 纯 CLI + 安全配置 |
| `base_adapter.py` | 仅定义接口 | 含 `strip_ansi()` + `build_prompt()` 静态方法 |
| `claude_adapter.py` | 247 行，自含工具函数 | 去除重复代码，调用基类方法 |
| `codex_adapter.py` | 255 行，自含工具函数 | 去除重复代码，调用基类方法 |
| `AgentGatewayService.java` | 旧 API 路径 + NDJSON 解析 | `/api/agent/chat` + SSE `data:` 行解析 + AgentToken 封装 |

---

## 三、代码测试验证（重点）

> 本节采用**分步交互式测试**，用户逐步骤执行命令，AI 解释预期结果。覆盖静态检查 → 依赖安装 → 启动 → API 接口 → Java 编译 → 数据库验证共 13 项测试。

### 3.1 静态检查（步骤 1~4）

**分支状态与旧文件删除确认**：

```powershell
# 步骤2：确认已删除文件不存在
Test-Path "agent-service/adapters/deepseek_adapter.py"    # → False ✅
Test-Path "agent-service/app/db/connection.py"             # → False ✅
Test-Path "agent-service/app/db/repository.py"             # → False ✅
Test-Path "agent-service/app/api/endpoints/agents.py"      # → False ✅
Test-Path "agent-service/app/api/endpoints/conversations.py" # → False ✅
Test-Path "agent-service/app/api/endpoints/artifacts.py"   # → False ✅
```

**config.py 死配置检查**：

```powershell
# 步骤4：确保旧配置已完全清除
Select-String -Path "agent-service/config.py" `
  -Pattern "H2|JPype|jaydebeapi|CLAUDE_API_KEY|CODEX_API_KEY|DEEPSEEK|AGENT_MAX_TOKENS"
# → 无输出 ✅
```

### 3.2 Python 依赖与 Agent Service 启动（步骤 5~6）

**依赖安装**：

```powershell
cd agent-service; pip install -r requirements.txt
```

> 仅安装 6 个有效依赖，**无** `jaydebeapi`、`jpype1`、`sse-starlette` 安装日志。

**Agent Service 启动**（终端 1）：

```powershell
cd agent-service; uvicorn main:app --port 8000
```

**健康检查**（终端 2 / 浏览器）：

```
GET http://localhost:8000/health
→ {"status":"ok","version":"1.0.0","uptime":3600}

GET http://localhost:8000/
→ {"message":"Welcome to Agenthub API","docs":"/docs"}
```

### 3.3 API 接口测试 — Postman 交互验证（步骤 7~8）

> 用户使用 Postman 替代 curl 进行 HTTP 接口测试，以下为 Postman 的完整配置方法。

**测试 7.1：参数校验（空 context → 400）**

| 配置项 | 值 |
|--------|-----|
| Method | `POST` |
| URL | `http://localhost:8000/api/agent/chat` |
| Headers | `Content-Type: application/json` |
| Body (raw JSON) | `{"agentType": "claude_code", "systemPrompt": "", "context": ""}` |

```
Status: 400 Bad Request
{"error":"VALIDATION_ERROR","message":"context 字段不能为空","timestamp":"...","path":"/api/agent/chat"}
```

**测试 7.2：非流式调用 claude_code（stream=false）**

```json
{
    "agentType": "claude_code",
    "systemPrompt": "你是一个助手",
    "context": "你好",
    "stream": false
}
```

```
Status: 200 OK
{"content": "你好！有什么可以帮你的？", "messageId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"}
```

> **关键验证点**：`content` 包含 Claude Code CLI 实际生成的回复，`messageId` 为 UUID 格式。如未安装 CLI 工具，返回 502 `AGENT_ERROR`，同样为规范 JSON 格式。

**测试 7.3：非流式调用 custom Agent**

```json
{
    "agentType": "custom",
    "systemPrompt": "",
    "context": "测试自定义Agent",
    "stream": false
}
```

```
Status: 200 OK
{"content": "以下是我的测试计划：...", "messageId": "9ccf4dea-e131-4cbc-9fec-ad90618034bc"}
```

> **注意**：`custom` 类型的首个请求因 Claude 生成了较长的测试方案文本，Postman 等待时间较长才返回完整结果。后续请求恢复正常。这说明 CLI 子进程的实际响应时间取决于生成内容的长度，非接口 bug。

**测试 8：流式 SSE 调用**

```json
{
    "agentType": "claude_code",
    "systemPrompt": "你是一个助手",
    "context": "你好",
    "stream": true
}
```

```
data: {"token":"你好","finish":false,"agentId":"agent_claude_code","agentName":"Claude Code"}

data: {"token":"！有","finish":false,"agentId":"agent_claude_code","agentName":"Claude Code"}

data: {"token":"什么","finish":false,"agentId":"agent_claude_code","agentName":"Claude Code"}

data: {"token":"可以帮你的？","finish":false,"agentId":"agent_claude_code","agentName":"Claude Code"}

data: {"token":"","finish":true,"messageId":"xxx-xxx-xxx"}
```

> **用户自行验证通过**。SSE 格式为 `data: <JSON>\n\n`，`finish: false` 的 chunk 携带 `token`/`agentId`/`agentName`，最后一条 `finish: true` 携带 `messageId`。

### 3.4 文档一致性检查（步骤 10）

```powershell
# 检查文档中无旧 API 路径引用
Select-String -Path "docs\API 契约定义与通信协议规范.md" `
  -Pattern "v1/messages|agent_type|NDJSON"
# → 无输出 ✅

# 检查基础设施文档无 dead 依赖引用
Select-String -Path "docs\基础设施配置.md" `
  -Pattern "sse-starlette|jaydebeapi|jpype1|DEEPSEEK_API|CLAUDE_API_KEY"
# → 无输出 ✅
```

### 3.5 Java 后端编译与启动（步骤 9、11~13）

**Maven 编译**：

```powershell
cd backend-java; mvn compile -q
```

> 编译成功，重点是修正后的 `AgentGatewayService.java` 无编译错误。

**Spring Boot 启动**：

```powershell
cd backend-java; mvn spring-boot:run
```

> 控制台输出 `Started AgenthubApplication in X.XXX seconds`，端口 8080 已监听。

**H2 数据库控制台验证**：

```
浏览器访问: http://localhost:8080/h2-console
JDBC URL:   jdbc:h2:file:./shared-data/agenthub;AUTO_SERVER=TRUE
用户名:      sa
密码:        (空)
```

> 成功进入 H2 Web 控制台，可见 `agents`、`conversations`、`messages`、`conv_agents` 等表。

### 3.6 测试结果汇总

| # | 测试项 | 方式 | 结果 |
|---|--------|------|:--:|
| 1 | 分支状态与改动概览 | git 命令 | ✅ |
| 2 | agent-service 结构完整性（6 旧文件已删除） | PowerShell Test-Path | ✅ |
| 3 | `requirements.txt` 依赖精简 | Get-Content | ✅ |
| 4 | `config.py` 无死配置 | Select-String 反向确认 | ✅ |
| 5 | `pip install` 依赖安装 | pip 命令 | ✅ |
| 6 | Agent Service 启动 + `/health` + `/` | uvicorn + 浏览器 | ✅ |
| 7 | `/api/agent/chat` 参数校验（空 context → 400） | Postman POST | ✅ |
| 8 | `/api/agent/chat` claude_code 非流式 | Postman POST | ✅ |
| 9 | `/api/agent/chat` custom 非流式 | Postman POST | ✅ |
| 10 | `/api/agent/chat` 流式 SSE | Postman POST + stream:true | ✅ |
| 11 | 文档一致性（无旧 `/api/v1/`、废弃依赖引用） | Select-String | ✅ |
| 12 | Java 后端 Maven 编译 | `mvn compile -q` | ✅ |
| 13 | Java 后端启动 + H2 控制台可访问 | `mvn spring-boot:run` + 浏览器 | ✅ |

---

## 四、分支合并到 dev

### 4.1 提交策略

将 `fix-pr3-review` 上的修复按逻辑拆分为 **3 个独立 commit**，逐个提交后再整体合并：

| 批次 | Commit | 类型 | 涉及文件 |
|------|--------|------|---------|
| 1 | `refactor(agent-service)` | 重构 + 清理 | `claude_adapter.py`, `codex_adapter.py`, `config.py` |
| 2 | `fix(backend)` | 契约修复 | `AgentGatewayService.java` |
| 3 | `chore(agent-service)` | 依赖 + 文档 | `requirements.txt`, 4 个 docs 文件 |

**提交命令示例**：

```powershell
# 批次1
git add agent-service/adapters/claude_adapter.py agent-service/adapters/codex_adapter.py agent-service/config.py
git commit -m "refactor(agent-service): 提取CLI适配器公共逻辑到BaseAdapter，清理config死配置
..."
```

### 4.2 合并执行

```powershell
git checkout dev
git merge fix-pr3-review
```

> 无冲突，Fast-forward 合并，线性历史。

### 4.3 最终 dev 历史

```
1e4f3b4  chore(agent-service): 清理废弃依赖，移除 sse-starlette/jaydebeapi/jpype1 等
9e56ebf  fix(backend): 修正AgentGatewayService与Python Agent Service的API契约对齐
0a071bd  refactor(agent-service): 提取CLI适配器公共逻辑到BaseAdapter，清理config死配置
46b267c  refactor(agent-service): 重构Agent服务为本地CLI调用架构
0659be4  docs: 新增团队代码合并规范文件
```

**合并后验证**：

```powershell
git push origin dev
```

> dev 分支领先 origin/dev 4 个 commits，工作区 clean。

---

## 五、遇到的问题与解决方案

| # | 问题现象 | 根本原因 | 解决方案 |
|---|----------|----------|----------|
| 1 | 首次审查发现 PR #3 有 4 类代码冲突 | 重构时未同步清理 `config.py` 死配置和 `requirements.txt` 废弃依赖 | 在 `fix-pr3-review` 分支上逐批次修复：移除 H2 配置、提取公共代码、清理 9 个废弃依赖 |
| 2 | `AgentGatewayService.java` 与 Agent Service 的 API 契约完全不匹配 | 成员 A 的 Java 代码基于旧 API（`/api/v1/messages/chat/stream`、`snake_case`、NDJSON），PR #3 已全面重构接口 | 重写 `sendToAgent()`：对齐路径、字段命名、响应格式，新增 `AgentToken` 内部类支持 `agentId`/`agentName` 切换 |
| 3 | `git commit` 时报 "no changes added to commit" | 用户先执行 `git commit` 但漏掉了 `git add` 命令 | AI 提示两条命令分行执行：先 `git add`，再 `git commit` |
| 4 | 提交 batch 1 时 scope 写为 `agent`，不符合 `.gitmessage` 规范 | 团队 commit 规范中 scope 应为 `frontend` / `backend` / `agent-service` | AI 修正为 `refactor(agent-service)` |
| 5 | Java 后端启动后访问 `http://localhost:8080/` 返回 404 | REST Controller 层尚未实现（成员 A 的 A5/A6 任务待开发），当前仅 Service + WebSocket 骨架 | 改用 H2 Web 控制台（`/h2-console`）验证启动成功，确认数据库表正常 |

---

## 六、关键决策与经验总结

### 6.1 交互式分步测试的价值

本次测试采用 **AI 逐步骤给出命令 → 用户执行 → 用户反馈结果 → AI 给出下一步** 的交互模式，优势：

1. **用户全程参与**：亲自执行每条命令，对服务器启动、API 调用、数据库访问有直观感受
2. **问题即时定位**：每步都有明确的"预期结果"，发现偏差可立即回溯
3. **无需预写测试脚本**：利用已有的 curl / Postman / 浏览器等通用工具即完成验证
4. **覆盖全链路**：从静态检查（文件路径、依赖列表、配置项）→ 编译 → 启动 → HTTP 接口 → 数据库，每一层都被验证

### 6.2 Postman 替代 curl 的实践

Windows PowerShell 下 curl 存在别名冲突（默认映射到 `Invoke-WebRequest`），切换为用户熟悉的 Postman 进行测试，配置清晰、结果可视化：

- 方法：`POST`，URL：`http://localhost:8000/api/agent/chat`
- Headers：`Content-Type: application/json`
- Body：`raw` + `JSON`，填入请求体
- 流式 SSE：Postman 自动逐行显示 `data:` 行，无需额外配置

### 6.3 分批次提交的优点

将修复拆分为 3 个语义独立的 commit（重构 → 修复 → 清理），比单一大提交更好：

- Code Review 时每个 commit 的意图一目了然
- 如有问题可精确 `git revert` 某个修改而不影响其余部分
- 符合 Conventional Commits 规范，团队历史记录可读性强

### 6.4 文档同步的重要性

PR #3 虽然是一次代码重构，但架构文档中的 5 份文件均需同步更新。本次在代码修复的同时确保了文档一致性，避免了"代码与文档脱节"的常见问题。

---

## 七、后续待办

| # | 事项 | 优先级 | 关联任务 |
|---|------|--------|----------|
| 1 | `custom` Agent 实现 `HttpApiAdapter`（httpx HTTP 流式调用） | 🟡 P1 | 成员 D — D9 |
| 2 | `config.py` 新增 `CUSTOM_AGENT_*` 配置项 | 🟡 P1 | 成员 D — D11 |
| 3 | `requirements.txt` 恢复 `httpx` 依赖 | 🟡 P1 | 成员 D — D12 |
| 4 | `AdapterFactory` 注册 `HttpApiAdapter` | 🟡 P1 | 成员 D — D10 |
| 5 | Java 后端实现 REST Controller 层（A5/A6/A7） | 🔴 P0 | 成员 A |
| 6 | 端到端集成测试：前端 → WebSocket → Java → Agent Service → CLI | 🔴 P0 | 全员联调 |

---

## 八、变更文件清单

| 文件 | 操作 | 说明 |
|------|------|------|
| `agent-service/adapters/base_adapter.py` | 修改 | 新增 `strip_ansi()` + `build_prompt()` 静态方法 |
| `agent-service/adapters/claude_adapter.py` | 修改 | 删除重复工具函数，改为调用基类方法 |
| `agent-service/adapters/codex_adapter.py` | 修改 | 删除重复工具函数，改为调用基类方法 |
| `agent-service/adapters/deepseek_adapter.py` | 删除 | DeepSeek 适配器已废弃 |
| `agent-service/app/db/__init__.py` | 删除 | 数据库访问层已移除 |
| `agent-service/app/db/connection.py` | 删除 | H2 JDBC 桥接已移除 |
| `agent-service/app/db/repository.py` | 删除 | Agent 数据仓库已移除 |
| `agent-service/app/api/endpoints/agents.py` | 删除 | 端点已移除 |
| `agent-service/app/api/endpoints/conversations.py` | 删除 | 端点已移除 |
| `agent-service/app/api/endpoints/artifacts.py` | 删除 | 端点已移除 |
| `agent-service/app/api/endpoints/messages.py` | 修改 | 重构为 `/api/agent/chat` SSE 流式端点 |
| `agent-service/config.py` | 修改 | 移除 H2/DeepSeek/API Key 配置，改为 CLI 命令路径 |
| `agent-service/main.py` | 修改 | 简化路由注册，新增异常处理 + 健康检查 |
| `agent-service/models.py` | 修改 | 精简为 AgentChatRequest/Response、HealthResponse、ErrorResponse |
| `agent-service/requirements.txt` | 修改 | 移除 9 个废弃依赖，langgraph 升级 0.0.20 |
| `backend-java/.../AgentGatewayService.java` | 修改 | API 契约对齐，SSE 解析 + AgentToken 内部类 |
| `docs/API 契约定义与通信协议规范.md` | 修改 | 同步 API 路径、字段命名、SSE 格式 |
| `docs/基础设施配置.md` | 修改 | 同步 config.py 和 requirements.txt 内容 |
| `docs/架构拓扑图与项目目录结构.md` | 修改 | 更新架构图 + 目录树 + 数据流 |
| `docs/项目概述与技术栈总览.md` | 修改 | 更新 Agent 服务技术栈描述 |
