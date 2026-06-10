# 开发过程记录 — Orchestrator 多 Agent 协作编排实现

> **日期**: 2026-06-07  
> **关联分支**: `feat/orchestrator`（从 `dev` 分出）  
> **关联提交**: `0b1cacd`（核心实现）、`a71d93d`（降级+优化）  
> **关联文档**: `Orchestrator 调度器设计方案文档.md`（v1.0）  
> **涉及范围**: Python 调度器核心、Java 会话路由、前端会话绑定、System Prompt 优化  

---

## 一、背景

AgentHub 的设计愿景是"一个用户与多个 AI Agent 协作"。MVP 阶段已实现单 Agent 手动选择模式，但缺少自动化的多 Agent 编排调度。用户希望优先实现这一核心功能。

### 1.1 已有基础

- Orchestrator 系统提示词（`system_prompts.py`）
- AGENT_REGISTRY 含 orchestrator 条目
- `agent_switch` SSE 事件的前端支持
- 前端多 Agent 头像/名称渲染
- Orchestrator 设计方案文档

### 1.2 方案选择

设计文档提供了两个方案：

| 方案 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| A: 硬编码编排 | 固定流程依次调用 Agent | 简单 | 不灵活 |
| B: LLM 驱动 | Claude Code 作为编排大脑 | 动态决策 | 多 50 行代码 |

**选择方案 B**。Claude Code 驱动仅多 50 行代码，但灵活性有数量级的提升，一步到位。

---

## 二、核心架构

### 2.1 编排流程

```
用户发送消息
  → Java WebSocketController: conversation.type=group → agentType=null
  → Python messages.py: agentType 为空 → 路由到 Orchestrator
  → Orchestrator.execute():
      1. _plan(): Claude Code + orchestrator prompt → JSON 执行计划
      2. 按计划逐步骤调用子 Agent
      3. 每步发送 agent_switch 通知前端切换头像
      4. 子 Agent 失败 → 自动降级到 Claude Code 重试
      5. msg_end: 流结束
```

### 2.2 数据流

```
Java (agentType=null) → FastAPI /api/agent/chat
  → messages.py: is_orchestrator = True
  → Orchestrator.execute()
    → _plan() → claude -p "{orchestrator_prompt}\n{msg}" → JSON plan
    → for step in plan.steps:
        → AdapterFactory.get_adapter(step.agent)
        → adapter.chat_stream(task=step.task)  (无状态执行)
        → agent_switch 事件 + 流式 chunk
    → msg_end
```

### 2.3 改动文件

| 文件 | 类型 | 说明 |
|------|------|------|
| `orchestrator.py` | 新建 | 核心调度器：plan 生成 + 步骤执行 + 降级策略 |
| `messages.py` | 改造 | 新增 Orchestrator 路由 + `_orchestrator_stream`/`_orchestrator_non_stream` |
| `WebSocketController.java` | 改造 | group 会话传 agentType=null 触发编排 |
| `AgentGatewayService.java` | 修复 | null agentType 透传（之前强行改成 claude_code） |
| `models.py` | 改造 | agentType 改为 Optional[str] |
| `system_prompts.py` | 优化 | 移除"纯文本模式"/"权限"等负面措辞 |
| `chat.ts` | 修复 | 超时延长至 120s + createOffice 后同步 conversationList |
| `ChatList.vue` | 改造 | group 会话路由到 ChatView（支持多 Agent 协作） |

---

## 三、调度器核心设计

### 3.1 Orchestrator 类结构

```python
class Orchestrator:
    def __init__(self):
        self.factory = AdapterFactory

    async def _plan(self, message, working_directory) -> dict
        # 调用 Claude Code → 收集输出 → 解析 JSON 计划

    def _parse_plan(self, text) -> dict
        # 三级策略提取 JSON：```json``` → ```{}``` → 裸正则

    async def execute(self, message, working_directory, conversation_id) -> AsyncGenerator
        # 主调度循环
```

### 3.2 调度策略

```
用户消息 → Orchestrator._plan() → Claude Code 分析 → JSON 计划
                                            ↓
                              { "analysis": "...", "steps": [...] }
                                            ↓
                              0 步骤 → 退化为单 Agent 模式
                              1 步骤 → 单 Agent 执行
                              2 步骤 → 串行执行（步骤间有分隔线）
```

**Agent 选择策略**（编码在 `_PLAN_INSTRUCTION` 中）：
- 绝大多数任务用 1 个 Agent
- 优先选 `claude_code`（全栈）
- `codex` 仅用于纯前端任务（HTML/CSS/JS 组件）
- 对话/问候/问答 → `claude_code`，1 个步骤

### 3.3 降级策略（三层）

```
第 1 层 — Plan 生成失败
  _plan() 异常 / JSON 解析失败
    → steps = []
    → 退化为单 Agent 模式（Claude Code 直接回复）

第 2 层 — 子 Agent CLI 执行失败
  子 Agent 返回 error chunk && agent_type != "claude_code"
    → 自动切 Claude Code 重试
    → 前端显示 "Claude Code (降级)"
    例: Codex 未配置 → 降级 → Claude Code 完成

第 3 层 — Claude Code 自身失败
  Claude Code 返回 error chunk
    → 前端显示 "⚠️ Claude Code 执行失败，请稍后重试"
```

### 3.4 日志设计

```
[Orchestrator] 开始分析任务: 用 Python 写冒泡排序...
[Orchestrator] 执行计划: 2 个步骤, analysis=用户请求前后端分离...
[Orchestrator] 步骤 1/2: 调度 Claude Code → 用纯 Python 实现...
[Orchestrator] Codex 失败，降级到 Claude Code        ← 降级触发
[Orchestrator] 步骤 2/2: 调度 Codex → 编写单元测试...
[Orchestrator] 全部 2 个步骤执行完成
```

---

## 四、Prompt 工程

### 4.1 Orchestrator Plan 提示词

```python
_PLAN_INSTRUCTION = """
## 你的工作方式

当收到用户请求后，你必须按以下格式输出一个 JSON 执行计划（仅 JSON，不要其他文字）：

```json
{
  "analysis": "一句话分析用户需求",
  "steps": [
    {"agent": "claude_code", "task": "具体的任务描述"},
    {"agent": "codex", "task": "具体的任务描述"}
  ]
}
```

规则：
- `agent`: "claude_code"（全栈，首选）或 "codex"（仅纯前端）
- **优先选 claude_code**：除非纯前端任务，否则都用 claude_code
- 对话/聊天/问候/简单问答：用 claude_code，1 个步骤
- 每个 `task` 必须自包含且**只描述要做什么，绝对不要提及 Agent 名称**
- 输出必须是纯 JSON，不要任何解释文字
```

### 4.2 System Prompt 优化

**问题**: 旧提示词强调"你运行在纯文本模式下，无法写入文件、无法执行命令"，导致 Agent 回复中频繁提到"需要批准权限"，用户体验差。

**修复**: 去掉所有负面描述，改为正面引导。

```diff
- "## 重要：你运行在纯文本模式下\n"
- "- 你无法写入文件、执行命令或修改系统。你的所有输出都是纯文本。\n"
- "- 不要要求用户给予文件写入权限——你只能通过聊天输出代码。\n"
+ "## 重要：工作模式\n"
+ "- 你通过聊天输出所有内容，代码用 Markdown 代码块包裹并标注语言和文件路径。\n"
+ "- 直接输出完整代码，不要询问权限或要求用户批准。\n"
```

### 4.3 子 Agent 任务 Prompt 设计

子 Agent 不依赖 CLI 会话记忆，任务 prompt 必须**自包含**。Orchestrator 的 `_PLAN_INSTRUCTION` 强制要求：

- ✅ 正确: "用纯 Python 写一个冒泡排序函数，含单元测试"
- ❌ 错误: "Claude Code 写一个冒泡排序"（不应提及 Agent 名称）

---

## 五、问题排查与修复

### 5.1 agentType null 被 Java 强制改写

**现象**: Orchestrator 从未被触发，Python 未收到 null agentType。

**根因**: `AgentGatewayService.java` 中 `body.put("agentType", agentType != null ? agentType : "claude_code")` 将所有 null 改为 `claude_code`。

**修复**: 改为 `body.put("agentType", agentType)`，允许 null 透传。

### 5.2 Pydantic 拒绝 null agentType

**现象**: Python 返回 400 Bad Request。

**根因**: `models.py` 中 `agentType: str` 不接受 null 值。

**修复**: 改为 `agentType: Optional[str] = Field(default=None)`。

### 5.3 Lambda 变量非 effectively final

**现象**: Java 编译失败，lambda 中的 `agent` 变量不是 effectively final。

**根因**: `agent` 在 if/else 分支中被赋值，Java 编译器不允许 lambda 捕获非 final 变量。

**修复**: 声明 `final Agent agent` 并在两个分支中各自赋值一次。

### 5.4 Python 语法错误

**现象**: `SyntaxError: invalid character '。'`，但字符在 `"""..."""` 字符串内。

**根因**: 前一行存在一个空的 `"""` 表达式，Python 解析器将其视为新字符串的开始，导致后续代码解析错乱。

**修复**: 删除空 `"""` 行。

### 5.5 办公室创建后聊天抽屉未绑定

**现象**: 创建办公室后打开聊天抽屉，发送消息无回应。

**根因**: `createOffice` 创建的 conversation 不在 `conversationList` 中，`selectConversation` 的校验逻辑直接拒绝了该 ID。

**修复**: `createOffice` 成功后立即将新会话 `unshift` 到 `conversationList`。

---

## 六、测试验证

| 测试场景 | 预期 | 结果 |
|---------|------|:--:|
| 简单问候 "你好" | plan 0 步骤 → 退化为单 Agent | ✅ |
| Python 冒泡排序 | 1 步骤 → Claude Code | ✅ |
| HTML 页面 + 指定 Codex | Codex 失败 → 降级到 Claude | ✅ |
| 前端+后端跨领域任务 | 2 步骤 → 串行执行 | ✅ |
| 办公室创建后聊天 | conversationList 同步 → 可发消息 | ✅ |
| 回复中无虚假 Agent 名称 | task 描述净化 → 不提及 Agent | ✅ |
| 调度日志完整性 | 全流程日志可追溯 | ✅ |

---

## 七、当前局限与后续方向

| 局限 | 说明 | 计划 |
|------|------|------|
| Codex 未配置 | 真正多 Agent 并行无法测试 | 阶段 2 |
| 子 Agent 串行执行 | 当前按顺序执行，不支持并行 | P2 优化 |
| Plan 解析依赖 JSON | Claude 偶尔不按格式输出 | 可改进 prompt 或加重试 |
| 无 Plan 缓存 | 相同任务重复分析 | P2 可加缓存 |

---

## 八、经验总结

### 8.1 LLM 驱动的编排比硬编码更灵活

方案 B 仅多 50 行代码，但可以处理任意类型的任务组合。Claude Code 作为调度大脑，动态决定 Agent 分配，而不需要为每种任务类型写死流程。

### 8.2 降级策略要多层兜底

三层降级（plan 失败 → 子 Agent 失败 → Claude 失败）确保在任何异常情况下用户都能得到回复。每一层都是独立的 safety net。

### 8.3 Prompt 措辞影响用户体验

"纯文本模式"、"无法写入"、"权限批准"等负面措辞会让 Agent 在回复中反复提及权限问题，降低回复质量。改为正面引导"直接输出代码"后问题消失。

### 8.4 null 值的跨语言传递容易被截断

Java → JSON → Python 的 null 传递链中，任何一层的"防御性 fallback"都可能破坏语义。本次修复了三处：Java HashMap put、Spring WebClient body、Pydantic 类型校验。
