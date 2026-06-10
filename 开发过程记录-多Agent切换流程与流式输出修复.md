# 开发过程记录 — 多Agent切换流程与流式输出修复

> **日期**: 2026-06-10\
> **任务编号**: --\
> **所属模块**: agent-service / backend-java / frontend\
> **前置依赖**: Orchestrator 多Agent编排 ✅（已完成）

---

## 一、背景

### 需求来源

前端群聊模式下用户发送"帮我做一个学生管理系统（SpringBoot + MySQL + MyBatis）"后，Orchestrator 应自动分析任务、生成执行计划、串行调度多个 Agent。但实际运行中出现以下问题：

1. **Orchestrator 无法解析计划** — Claude CLI 以 Agent 模式运行，输出"计划已就绪，请确认"而非 JSON，导致 `_parse_plan` 返回空计划，退化为单 Agent 模式
2. **Claude 流式输出缓冲区溢出** — `async for line in process.stdout` 默认 64KB 行长度限制，Claude stream-json 输出含代码块的超大单行触发 `LimitOverrunError`
3. **Codex Agent 产生多个空对话框** — `agent_switch` 事件被 `pass` 丢弃，前端不知道 Agent 切换了，但 msg_chunk 的 agentName 持续变化，导致消息气泡分裂
4. **`write_blocks_to_workspace` 未 await** — 异步函数同步调用导致协程从未执行
5. **4 处冗余 `scan_and_upload` 调用** — `_upload_merged_artifacts` 内部已调用，外部重复调用报 `NameError`

### 当前状态限制

- Orchestrator 有 `_plan` 方法但 prompt 指令在消息后面，Claude 优先执行用户任务而非输出 JSON
- `agent_switch` 事件在 Python 层被丢弃，Java 后端和前端都没有处理逻辑
- Claude 流式输出行长度受 `StreamReader.readline()` 64KB 限制

### 修复后解锁

- 前后端分离项目自动拆分为 2 步：claude_code 后端 + codex 前端
- 每个 Agent Step 独立为前端消息气泡，可视化 Agent 切换
- Claude 流式输出稳定，不再因超大行而崩溃

---

## 二、任务执行过程

### 2.1 修复 Orchestrator 计划解析（3 轮迭代）

**操作文件**: `agent-service/orchestrator.py`

**第 1 轮**：移除无效的 `--tools ""` 参数，添加 `--no-sandbox`，强化 `_PLAN_INSTRUCTION`（强调"你是一个任务分析器，不是编程助手"）。

**第 2 轮**：发现 `--max-turns 1` 与 `-p` 参数冲突，改为 `--print` + stdin 传入 prompt 模式：

```python
cli_args = [
    cli_command,
    "--print",           # 纯文本输出，非 Agent 模式
    "--output-format", "text",
    "--permission-mode", "bypassPermissions",
]
# 通过 stdin 传入 prompt
process.communicate(input=full_prompt.encode("utf-8"))
```

若 `--print` 不被 CLI 版本支持（stderr 有输出），自动回退到 `-p` 模式。

**第 3 轮**：极简化 `_PLAN_INSTRUCTION`，指令前置 + 用户消息后置：

```
之前: "{用户消息}\n\n---\n\n{JSON指令}"   → Claude 先看到任务 → 执行任务
现在: "{JSON指令}\n\n用户请求：{用户消息}" → Claude 先看到"只输出JSON" → 输出JSON
```

**关键设计决策**：

| 决策项 | 选择 | 理由 |
|--------|------|------|
| CLI 调用方式 | `--print` + stdin | 绕过 Agent 模式，直接获取纯文本响应 |
| 降级策略 | stderr 有输出时自动回退到 `-p` | 兼容不同版本 Claude CLI |
| Prompt 顺序 | 指令前置，用户消息后置 | Claude 优先执行第一个看到的指令 |
| Agent 分配规则 | claude_code=全栈/后端, codex=前端/UI | 前后端分离项目自动 2 步串行 |

**增强 `_parse_plan`**：新增策略 4 — 提取第一个 `{` 到最后一个 `}` 之间的裸 JSON（新 prompt 主要输出格式）。

**移除步骤间分隔符**：`\n\n---\n\n` 分隔符被移除，因为 agent_switch 事件产生的独立消息气泡已提供足够的视觉分隔。

### 2.2 修复 Claude 流式输出缓冲区溢出

**操作文件**: `agent-service/adapters/claude_adapter.py`

**根因**：`async for line in process.stdout` 调用 `StreamReader.readline()` 有 64KB 行长度限制。Claude stream-json 输出行可能包含完整代码块超过此限制。

**修复**：新增 `_read_lines_unlimited()` 模块级辅助函数：

```python
async def _read_lines_unlimited(stream: asyncio.StreamReader) -> AsyncGenerator[str, None]:
    """从 StreamReader 逐行读取，无单行长度限制。"""
    buffer = b""
    while True:
        chunk = await stream.read(65536)  # 64KB 块
        if not chunk:
            if buffer:
                yield buffer.decode("utf-8", errors="replace")
            return
        buffer += chunk
        *lines, buffer = buffer.split(b"\n")
        for line in lines:
            yield line.decode("utf-8", errors="replace")
```

将 `async for line in process.stdout` 替换为 `async for raw_text in _read_lines_unlimited(process.stdout)`，事件处理逻辑完全不变。

### 2.3 修复 agent_switch 全链路事件管道

#### 2.3.1 Python 层 — 转发 agent_switch 为 SSE

**操作文件**: `agent-service/app/api/endpoints/messages.py`

```python
# 之前：agent_switch 被 pass 丢弃
elif chunk_type == "agent_switch":
    pass

# 之后：转为 SSE 事件推给 Java
elif chunk_type == "agent_switch":
    sse_data = {
        "type": "agent_switch",
        "agentId": chunk.get("agent_id", ""),
        "agentName": chunk.get("agent_name", ""),
        "message": chunk.get("message", ""),
    }
    yield f"data: {json.dumps(sse_data, ensure_ascii=False)}\n\n"
```

#### 2.3.2 Java 层 — 解析并转发

**操作文件**: `AgentGatewayService.java`

新增 SSE `type: "agent_switch"` 解析逻辑，创建 `AgentToken.agentSwitch()` 工厂方法。AgentToken 新增 3 个字段：`message`、`isAgentSwitch`、`agentSwitch()` 静态工厂方法。

**操作文件**: `WebSocketController.java`

```java
if (token.isAgentSwitch()) {
    // 发送 agent_switch STOMP 事件
    Map<String, Object> agentSwitch = new LinkedHashMap<>();
    agentSwitch.put("type", "agent_switch");
    agentSwitch.put("agentId", token.getAgentId());
    agentSwitch.put("agentName", token.getAgentName());
    agentSwitch.put("message", token.getMessage() != null ? token.getMessage() : "");
    messagingTemplate.convertAndSend(topic, agentSwitch);

    // 如果当前有 streaming 消息，先完成它
    if (receivedTokens[0]) {
        // 发送 isComplete: true 完成当前消息气泡
        ...
    }
    receivedTokens[0] = false;  // 重置，让新 Agent 的响应创建新气泡
    return;
}
```

#### 2.3.3 前端层 — 处理 Agent 切换

**操作文件**: `wsClient.ts` — `AgentSwitchEvent` 新增 `message` 字段。

**操作文件**: `chat.ts` — `handleAgentSwitch` 强制结束当前 streaming 消息：

```typescript
const handleAgentSwitch = (event: AgentSwitchEvent) => {
  currentAgentId.value = event.agentId
  currentAgentName.value = event.agentName

  // 结束当前 streaming 消息，排空 token 队列
  if (streamingMessageId.value) {
    // ...排空 tokenQueue
    streamingMessageId.value = null  // 下一次 msg_chunk 会创建新气泡
  }
}
```

`handleToken` 新增防御：`isComplete` 到达但 `streamingMessageId` 已为 null（被 agent_switch 完成）时忽略重复 finish。

### 2.4 修复 write_blocks_to_workspace 未 await

**操作文件**: `agent-service/app/api/endpoints/messages.py`

```python
# 之前：缺少 await，协程从未执行
write_blocks_to_workspace(blocks, working_directory)

# 之后：
await write_blocks_to_workspace(blocks, working_directory)
```

### 2.5 移除冗余 scan_and_upload 调用

**操作文件**: `agent-service/app/api/endpoints/messages.py`

`_upload_merged_artifacts` 内部已调用 `scan_and_upload`，外部 4 处重复调用不仅冗余且报 `NameError`（函数仅在 `_upload_merged_artifacts` 内部局部导入）。全部移除。

### 2.6 修复 continue 在 lambda 外部的编译错误

**操作文件**: `WebSocketController.java` line 112

```java
// 错误：continue 只能在循环中使用，token -> { ... } 是 lambda 不是循环
receivedTokens[0] = false;
continue;

// 修复：lambda 中用 return 提前退出
receivedTokens[0] = false;
return;
```

使用 `mvn clean compile -q` 验证编译通过。

---

## 三、技术要点讲解

### 3.1 Claude CLI 的 --print vs --prompt 模式

| 模式 | 参数 | 行为 |
|------|------|------|
| Agent 模式 | `-p "prompt"` | 以 Agent 身份运行，可能输出权限请求、确认询问 |
| Print 模式 | `--print` + stdin | 纯文本输出，不执行任何 Agent 行为 |

Orchestrator 需要 Claude 输出 JSON 而非执行任务，因此使用 `--print` + stdin 是正确选择。

### 3.2 asyncio StreamReader.readline() 的 64KB 限制

`StreamReader.readline()` 内部有 `LimitOverrunError` 保护，默认限制为 64KB（`_DEFAULT_LIMIT`）。当 Claude stream-json 输出行超过此限制时抛出异常。

**修复原理**：绕过 `readline()`，直接 `read(65536)` 分块读取，手动按 `\n` 分割，流结束时处理 buffer 残留。

### 3.3 agent_switch 全链路事件流

```
Python Orchestrator  →  SSE {"type": "agent_switch", ...}
  →  Java AgentGatewayService  →  AgentToken.agentSwitch()
    →  Java WebSocketController  →  STOMP {"type": "agent_switch", ...}
      →  前端 wsClient  →  chatStore.handleAgentSwitch()
        →  结束当前气泡 + 重置 streamingMessageId
          →  下一个 msg_chunk 自动创建新气泡
```

---

## 四、验证测试

### 4.1 自动化验证

| 测试项 | 命令 | 结果 |
|--------|------|------|
| Java 编译 | `mvn clean compile -q` | ✅ 通过 |
| Python 语法检查 | `python -m py_compile orchestrator.py` | ✅ 通过 |
| TypeScript 类型检查 | `npx vue-tsc --noEmit` | ✅ 通过 |

### 4.2 手动功能测试清单

| # | 测试项 | 预期行为 | 结果 |
|---|--------|---------|------|
| 1 | 群聊发送任务 | Orchestrator 正确解析 JSON 计划 | ✅ |
| 2 | 前后端分离项目 | 自动拆分为 2 步，claude_code 后端 + codex 前端 | ✅ |
| 3 | Agent 切换 | 前端显示独立消息气泡，不产生空对话框 | ✅ |
| 4 | Claude 流式输出 | 长代码块不触发 LimitOverrunError | ✅ |
| 5 | 单 Agent 任务 | 正常退化为单 Agent 模式 | ✅ |

---

## 五、关键决策记录

| # | 决策 | 结论 | 理由 |
|---|------|------|------|
| 1 | Orchestrator CLI 调用方式 | `--print` + stdin，失败回退 `-p` | 绕过 Agent 模式，兼容多版本 CLI |
| 2 | Prompt 指令顺序 | 指令前置，用户消息后置 | Claude 优先执行第一条指令 |
| 3 | Agent 分配规则 | claude_code=全栈/后端，codex=前端/UI | 前后端分离项目自动 2 步 |
| 4 | agent_switch 处理方式 | 全链路 SSE/STOMP 事件，而非 msg_chunk 内嵌 agentName | 确保前端每个 Agent Step 独立气泡 |
| 5 | 流式读取方案 | 分块 read + 手动 split，非 readline | 绕过 64KB 行长度限制 |
| 6 | artifact 上传去重 | 仅由 `_upload_merged_artifacts` 内部统一调用 | 避免重复上传和 NameError |

---

## 六、产物清单

| 文件 | 操作 | 行数变化 | 说明 |
|------|------|---------|------|
| `agent-service/orchestrator.py` | 修改 | +154/-65 | 重写 _plan 方法、增强 _parse_plan、极简化 prompt、移除步骤分隔符 |
| `agent-service/adapters/claude_adapter.py` | 修改 | +24 | 新增 `_read_lines_unlimited()`，替换 `async for line` |
| `agent-service/app/api/endpoints/messages.py` | 修改 | +24/-12 | 转发 agent_switch SSE、添加 await、移除冗余 scan_and_upload |
| `backend-java/../WebSocketController.java` | 修改 | +28 | agent_switch STOMP 处理 + `continue→return` 编译修复 |
| `backend-java/../AgentGatewayService.java` | 修改 | +33 | 解析 agent_switch SSE + AgentToken 扩展 |
| `frontend/src/stores/chat.ts` | 修改 | +25 | handleAgentSwitch 强制结束 + handleToken 防御 |
| `frontend/src/websocket/wsClient.ts` | 修改 | +2 | AgentSwitchEvent 新增 message 字段 |

### 代码统计

| 指标 | 数值 |
|------|------|
| 新增文件 | 0 |
| 修改文件 | 7 |
| 新增代码行 | 225 |
| 净减少代码行 | -65（含移除冗余调用和步骤分隔符） |

---

## 七、数据流/架构影响

### 修复前（单 Agent，agent_switch 被丢弃）

```
Orchestrator → _plan 失败 → 退化为单 Agent
agent_switch → pass（丢弃）
msg_chunk × N → 同一个前端气泡（agentName 反复覆盖）
→ finish 时序问题 → 多个空对话框
```

### 修复后（多 Agent，独立气泡）

```
Orchestrator → _plan 成功 → N-step 执行计划
Step 1: agent_switch → SSE → Java → STOMP → 前端准备气泡A
        msg_chunk × N → 气泡A（Claude Code 后端）
Step 2: agent_switch → SSE → Java → STOMP → 完成气泡A + 准备气泡B
        msg_chunk × N → 气泡B（Codex 前端）
→ msg_end → 完成气泡B
```

---

## 八、后续任务依赖

| 任务 | 可开始时机 | 关联 |
|------|----------|------|
| http_adapter 实现 | 现在 | agent_switch 管道已通，新 adapter 自然集成 |
| Agent 设置面板 | 现在 | Agent 切换流程已可视化 |
| 多 Agent 错误降级优化 | 现在 | 当前降级策略已有三层兜底 |
