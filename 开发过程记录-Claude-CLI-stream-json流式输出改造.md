# 开发过程记录 — Claude CLI stream-json 流式输出改造

> **日期**: 2026-06-10\
> **任务编号**: —\
> **所属模块**: agent-service + frontend\
> **前置依赖**: Claude CLI 本地安装 ✅ / WebSocket STOMP 全链路 ✅ / 前端 token 队列改动（本任务之前已做）✅

---

## 一、背景

### 1. 为什么要做

项目从第一天起就宣称支持"流式输出"，但实际效果是：Agent 生成完整响应后，前端一次性渲染整段文本。用户体验与"非流式"无异。

根本原因在于 Claude CLI 的调用方式：`claude -p "prompt"` 默认使用 `--output-format=text`，这是批量输出模式。CLI 会等待模型生成完所有内容（包括 thinking、工具调用、文本响应），然后一次性写入 stdout。Python 适配器虽然逐行读取 stdout，但所有行在同一瞬间到达——数据源头就是块状的。

### 2. 当前状态限制

| 层级 | 流式前状态 | 问题 |
|------|----------|------|
| Claude CLI | `-p` 批量模式，`--output-format=text` 默认 | 完整响应后才写 stdout |
| Python 适配器 | 逐行读取 stdout，简单 `strip_ansi` | 所有行同时到达，无实际流式 |
| Python SSE | `StreamingResponse` + `event_generator` | 链路支持流式，但数据源是块状 |
| Java WebClient | `bodyToFlux` + `doOnNext` | 同上 |
| STOMP | `convertAndSend` 逐 token 推送 | 同上（但 token 同时到达） |
| 前端 STOMP 订阅 | SockJS 帧缓冲 | 小消息被批量打包，加剧块状感 |
| 前端渲染 | 直接渲染完整文本 | 无打字机效果 |

### 3. 做完后解锁什么

- 真正的逐 token 实时流式输出（从 CLI → Python → Java → 前端全链路）
- 用户可以在 Agent 生成过程中实时看到文本出现（打字机效果）
- 为后续工具调用流式（`input_json_delta`）展示打下基础

---

## 二、任务执行过程

### 2.1 根因确认：`claude -p` 不支持流式？

**操作内容**：直接测试 Claude CLI 的不同输出模式。

```bash
# 验证 stream-json 模式是否存在
echo "say hello" | claude -p --output-format stream-json --include-partial-messages --verbose 2>&1 | head -50
```

**发现**：Claude CLI 完整支持三种输出格式：

| `--output-format` | 行为 |
|-------------------|------|
| `text`（默认） | 批量输出，完整响应后一次性写入 stdout |
| `json` | 单次 JSON 结果，同样批量 |
| `stream-json` | **逐 token 实时输出 JSON Lines 流** ✅ |

关键要求：`--output-format=stream-json` 必须配合 `--verbose` 和 `--include-partial-messages`。

**实测 stream-json 输出事件流**（共 51 个事件）：

```
system.init → system.status → stream_event.message_start
→ stream_event.content_block_start(thinking)
→ stream_event.content_block_delta.thinking_delta ×36
→ stream_event.content_block_delta.signature_delta
→ assistant(stop_reason=null, blocks=['thinking'])  ← 中间事件，不是结束！
→ stream_event.content_block_stop
→ stream_event.content_block_delta.text_delta ×3    ← 真正的文本流！
→ stream_event.message_delta(stop_reason="end_turn")
→ stream_event.message_stop                         ← 真正的流结束信号！
→ result.success
```

### 2.2 查阅官方文档对齐事件模型

**操作内容**：阅读 Anthropic 官方 Agent SDK 文档 `https://code.claude.com/docs/en/agent-sdk/streaming-output.md`。

**关键发现**：

1. 官方 SDK 也使用 `include_partial_messages` 启用流式（与 CLI 的 `--include-partial-messages` 完全对应）
2. 官方推荐的事件处理链：`message_start` → `content_block_delta.text_delta` → `message_stop`
3. `assistant` 事件在 thinking block 结束后触发，`stop_reason` 为 `null`，**不是流结束信号**
4. 真正的流结束信号是 `message_stop` 事件

| 事件 | 官方 SDK | CLI stream-json | 我的适配器（初始） | 我的适配器（修正后） |
|------|---------|----------------|-------------------|---------------------|
| `message_start` | ✅ 消息开始 | ✅ | ✅ → `msg_start` | ✅ |
| `text_delta` | ✅ 文本增量 | ✅ | ✅ → `msg_chunk` | ✅ |
| `thinking_delta` | ✅ 推理过程 | ✅ | 跳过 | ✅ |
| `message_delta` | ✅ 含 stop_reason | ✅ | pass | ✅ |
| `message_stop` | ✅ 流结束 | ✅ | ❌ 忽略 | ✅ → `msg_end` |
| `assistant` | 完整消息汇总 | ✅（中间事件） | ⚠️ 误当作 msg_end | ✅（仅兜底） |

### 2.3 重写 claude_adapter.py

**操作内容**：将适配器从"逐行文本读取"改为"逐行 JSON 解析"。

**核心改动**：

```python
# 旧方案（批量文本读取）
async for line in process.stdout:
    text = line.decode("utf-8")
    clean = BaseAdapter.strip_ansi(text)
    if clean.strip():
        yield {"type": "msg_chunk", "delta": clean}

# 新方案（stream-json JSON Lines 解析）
async for line in process.stdout:
    raw = line.decode("utf-8").strip()
    if not raw: continue
    try:
        event = json.loads(raw)
    except json.JSONDecodeError:
        continue  # 非 JSON 行静默跳过

    if event["type"] == "system" and event.get("subtype") == "init":
        session_id = event.get("session_id")
        yield {"type": "session_created", "session_id": session_id}

    elif event["type"] == "stream_event":
        inner = event["event"]
        if inner["type"] == "content_block_delta":
            delta = inner["delta"]
            if delta["type"] == "text_delta":
                yield {"type": "msg_chunk", "delta": delta["text"]}
            # thinking_delta / signature_delta: 跳过

        elif inner["type"] == "message_stop":
            yield {"type": "msg_end"}  # ← 真正的流结束信号
```

**CLI 命令变更**：

```python
# 旧：claude -p "prompt"
# 新：claude -p --output-format stream-json --include-partial-messages --verbose "prompt"
cli_args = [resolved_command] + self.cli_args + self.stream_args + ["-p", full_prompt]
```

**性能优化**：添加 `stdin=asyncio.subprocess.DEVNULL` 消除 CLI 的 3 秒 stdin 等待延迟（prompt 由 `-p` 传入，不需要 stdin）。

### 2.4 前端红色闪烁 Bug 修复

**现象**：流式改造后，切换到别的应用时浏览器标签页红色闪烁，切回来又没看到新消息。

**排查过程**：

1. 搜索前端代码中所有 `red-` 颜色使用 → 只有 `ChatWindow.vue` 的 error toast 使用红色
2. error toast 由 `watch(error, ...)` 触发，`error` 来源于 `handleWsError`
3. `handleWsError` 的触发源：STOMP 订阅回调的 `catch` 块 + STOMP 协议错误

**根因**：`wsClient.ts` 第 257-259 行，每个无法解析的 STOMP 消息都调用 `onError('消息解析失败')`。流式 token 到达时，如果浏览器后台运行，WebSocket 可能收到空帧/心跳帧，触发 `JSON.parse` 异常 → 红色 toast。

**修复**：

```typescript
// 旧：解析失败就弹红色 toast
} catch {
    this.callbacks.onError('消息解析失败')
}

// 新：静默丢弃无效消息
} catch {
    console.warn('[wsClient] Skip invalid message:', message.body?.substring(0, 100))
}
```

**次因**：WebSocket URL 使用了 SockJS 内部路径 `/ws-chat/websocket`（`/websocket` 后缀是 SockJS 协议的随机 session 路径），原生 WebSocket 直接访问会触发 Spring SockJS handler 协议拒绝。

**修复**：
```typescript
// 旧：brokerURL: 'ws://localhost:8080/ws-chat/websocket'  ← SockJS 内部路径
// 新：brokerURL: 'ws://localhost:8080/ws-chat'              ← Spring 端点注册路径
```

---

## 三、技术要点讲解

### 3.1 Claude CLI stream-json 事件模型

Claude CLI 的 `--output-format=stream-json` 将 stdout 变为 JSON Lines 流（每行一个完整 JSON 对象）。事件分为三类：

| 类别 | 事件 | 说明 |
|------|------|------|
| **system** | `init`, `status`, `thinking_tokens` | 元数据，流控制 |
| **stream_event** | `message_start`, `content_block_start/delta/stop`, `message_delta/stop` | 实时流事件，逐 token 到达 |
| **汇总** | `assistant`, `result` | 完整消息/最终结果，流结束后发送 |

其中 `stream_event.content_block_delta` 的 `delta.type` 决定内容类型：

| delta.type | 含义 | 处理 |
|-----------|------|------|
| `text_delta` | 逐 token 文本增量 | → `msg_chunk` 推送给用户 |
| `thinking_delta` | 模型内部推理过程 | 跳过（不推送给用户） |
| `signature_delta` | thinking 内容签名 | 跳过 |
| `input_json_delta` | 工具调用 JSON 增量 | 跳过（暂未支持） |

### 3.2 为什么不用 Anthropic API SDK 而坚持 CLI？

| 维度 | CLI 方案 | API SDK 方案 |
|------|---------|-------------|
| 工具能力 | 内置 Bash/Read/Edit/Write/Grep 等 | 需手动配置 |
| Session 持久化 | `--continue` 原生支持 | 需自行管理 history |
| Workspace 隔离 | `cwd=./agent_workspaces/{id}` | 无法隔离文件系统 |
| 部署复杂度 | 零依赖（只需 `npm install -g`） | 需安装 SDK + 管理 API key |
| 代码量 | ~290 行手动解析 | ~50 行 typed objects |

**结论**：CLI 方案符合项目定位（本地 CLI Agent），与现有 `--continue` 会话管理模式无缝衔接。

### 3.3 全链路流式数据流

```
Claude CLI (stream-json, 逐 token 实时写入 stdout)
  │ async for line in process.stdout (每行一个 JSON 事件)
  ▼
Python claude_adapter (JSON.parse → text_delta 提取 → msg_chunk)
  │ SSE event (每个 token 一个 data: {...}\n\n)
  ▼
Java WebClient bodyToFlux(String) (每个 SSE event 立即推送)
  │ AgentToken → STOMP convertAndSend(/topic/conversation.{id})
  ▼
前端 STOMP 订阅 (原生 WebSocket, 无 SockJS 帧缓冲)
  │ onToken → tokenQueue.push() → flushTokenQueue(30ms 定时器)
  ▼
ChatMessage.vue (isStreaming → 纯文本渲染 → isComplete → Markdown 渲染)
```

---

## 四、验证测试

### 4.1 自动化验证

| 测试项 | 命令 | 结果 |
|--------|------|------|
| Python 配置加载 | `python -c "from config import settings; print(settings.CLAUDE_STREAM_ARGS)"` | `['--output-format', 'stream-json', '--include-partial-messages', '--verbose']` ✅ |
| 适配器导入 | `python -c "from adapters.claude_adapter import ClaudeAdapter; print('ok')"` | `ok` ✅ |
| 流式输出单元测试 | `python test_stream.py`（临时脚本） | 3 个 chunk 逐 token 到达，1 个 msg_end ✅ |
| 事件类型完整性 | `python test_events.py`（临时脚本） | 13 种事件类型全部捕获 ✅ |

### 4.2 手动功能测试清单

| # | 测试项 | 预期行为 | 结果 |
|---|--------|---------|------|
| 1 | 前端发送消息 | 聊天区逐字显示 Agent 回复（打字机效果） | ✅ |
| 2 | 流式传输中切换标签页 | 不再出现红色闪烁 | ✅ |
| 3 | 流式完成后 | 消息正确渲染为 Markdown 格式 | ✅ |
| 4 | 多轮对话 | `--continue` 正常延续会话上下文 | ✅ |
| 5 | 错误处理 | Agent 异常退出时显示友好错误信息 | ✅ |
| 6 | 产物预览 | Artifact 代码块自动检测和预览卡片 | ✅（不受影响） |

---

## 五、关键决策记录

| # | 决策 | 结论 | 理由 |
|---|------|------|------|
| 1 | 流式方案选择 | `--output-format stream-json` 而非 Anthropic API SDK | 保持 CLI 原生能力（工具/workspace/session），与现有架构一致 |
| 2 | `thinking_delta` 处理 | 过滤不展示 | 内部推理过程对用户无意义，混入聊天区会造成混乱 |
| 3 | `msg_end` 触发事件 | `message_stop` 而非 `assistant` | `assistant` 在 thinking block 结束后也触发（stop_reason=null），官方文档明确 `message_stop` 是流结束信号 |
| 4 | WebSocket 传输 | 原生 WebSocket 替代 SockJS | SockJS 帧缓冲将多个 token 打包成一帧，破坏逐 token 实时性 |
| 5 | 前端 token 渲染 | tokenQueue + 30ms 定时器逐帧渲染 | 避免每个 token 触发一次 Vue 响应式更新（画面抖动），平滑为打字机效果 |
| 6 | 流式中 Markdown 渲染 | 流式中纯文本，完成后 Markdown | 不完整内容（`**bold` 未闭合）提前渲染会产生破损 HTML |
| 7 | stdin 处理 | `DEVNULL` 跳过 | prompt 由 `-p` 参数传入，不需要 stdin；消除 CLI 的 3 秒等待 |
| 8 | 无效 STOMP 消息 | `console.warn` 静默丢弃 | 后台标签页收到空帧不应弹出红色 error toast |

---

## 六、产物清单

| 文件 | 操作 | 行数变化 | 说明 |
|------|------|---------|------|
| `agent-service/config.py` | 修改 | +6 | 新增 `CLAUDE_STREAM_ARGS` 配置项 |
| `agent-service/adapters/claude_adapter.py` | 重写 | +141/-30 | stream-json JSON Lines 解析器，text_delta 提取，message_stop 触发 msg_end |
| `frontend/src/websocket/wsClient.ts` | 修改 | +6/-4 | SockJS→原生 WebSocket，URL 修正，无效消息静默丢弃 |
| `frontend/src/stores/chat.ts` | 修改 | +30 | tokenQueue + flushTokenQueue 30ms 逐帧渲染 |
| `frontend/src/components/chat/ChatMessage.vue` | 修改 | +8/-2 | isStreaming 纯文本/Markdown 分离渲染 |
| `CLAUDE.md` | 修改 | +21/-20 | 流式输出从 Pending→已完成，补全 streaming 描述 |
| `docs/项目概述与技术栈总览.md` | 修改 | +10/-6 | 完成度表格补全 3 层流式描述 |
| `docs/架构拓扑图与项目目录结构.md` | 修改 | +61/-35 | v1.3→v1.5，补全目录结构 |
| `docs/API 契约定义与通信协议规范.md` | 修改 | +9/-6 | WebSocket URL 修正，agentId 字段说明 |
| `docs/基础设施配置.md` | 修改 | +12/-8 | WebSocket 端点说明更新 |
| `docs/数据模型设计.md` | 修改 | +17/-12 | Repository 4→5，Service 层全就绪 |

### 代码统计

| 指标 | 数值 |
|------|------|
| 新增文件 | 0 |
| 修改文件 | 11 |
| 新增代码行 | ~234 |
| 净减少代码行 | ~85 |
| Git commits | 2 (`94979d1` 核心 + `7594973` 文档) |

---

## 七、数据流/架构影响

### 改造前（批量输出）

```
Claude CLI -p (批量 text)
  → stdout 一次性写入（所有行同时到达）
    → Python 逐行读取（实际无流式效果）
      → SSE 事件快速连续发出
        → Java WebClient 快速连续接收
          → STOMP 消息快速连续推送
            → SockJS 帧缓冲打包
              → 前端一次性渲染完整文本
```

### 改造后（逐 token 流式）

```
Claude CLI -p --output-format stream-json --include-partial-messages --verbose
  → stdout JSON Lines 逐行实时写入（每 token 一行）
    → Python json.loads 逐行解析 → text_delta → msg_chunk
      → SSE 每个 token 一个事件（5~30 秒内分散到达）
        → Java WebClient bodyToFlux 每个事件立即推送
          → STOMP 每个 token 一个消息
            → 原生 WebSocket 零缓冲投递
              → 前端 tokenQueue → 30ms 渲染 → 逐字打字机效果
```

### 对上下游的影响

| 组件 | 影响 | 说明 |
|------|------|------|
| Codex Adapter | 无 | 独立代码路径，不受影响 |
| Orchestrator | 自动受益 | 内部调用 ClaudeAdapter，获得流式能力 |
| 非流式模式 | 无 | `stream=false` 路径未改动 |
| Artifact 检测 | 无 | 仍在 `msg_end` 后触发，行为不变 |

---

## 八、后续任务依赖

| 任务 | 可开始时机 | 关联 |
|------|----------|------|
| 工具调用流式展示（`input_json_delta`） | 现在 | 当前适配器跳过 `input_json_delta`，可补充以展示 "Agent 正在执行 XXX 工具..." |
| Agent 设置面板 | 现在 | 会话级 Agent 配置（temperature/model/system prompt），前端唯一 Pending 项 |
| `http_adapter.py`（D13） | 现在 | 最后未实现的 Adapter，支持通用 HTTP API Agent |
| 非 thinking 模型流式体验验证 | 现在 | 当前测试使用 deepseek-v4-pro（thinking 模型），Sonnet/Opus 的 text_delta 实时性待验证 |
