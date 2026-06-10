# AgentHub 技术框架文档 — 会话模式路由与 Agent 调度逻辑
>版本: v1.0\
>创建日期: 2026-06-03\
>用途: 集中说明单 Agent 模式与多 Agent 模式的完整路由逻辑\
>关联文档: API 契约定义与通信协议规范、数据模型设计、Orchestrator 调度器设计方案

## 1. 两种会话模式定义
### 1.1 数据库层面
| 字段 | 值 | 含义 |
| --- | --- | --- |
| conversations.type | "direct" | 单 Agent 模式（用户对一个 Agent） |
| conversations.type | "group" | 多 Agent 模式（用户对多个 Agent，群聊） |

### 1.2 模式对比总览
| 维度 | 单 Agent 模式 (direct) | 多 Agent 模式 (group) |
| --- | --- | --- |
| 谁选择 Agent | 用户在创建会话时指定，或发送消息时临时选择 | Orchestrator 自动决策 |
| Agent 数量 | 1 个（会话默认 Agent） | 多个（会话关联的所有 Agent） |
| agentType 传参 | 明确传入（如 "claude_code"） | 传入 null 或 "orchestrator" |
| agent_switch 事件 | 不推送 | 每次切换 Agent 时推送 |
| 上下文构建 | 历史消息直接作为当前 Agent 的输入 | Orchestrator 将前序步骤结果注入后续步骤 |
| 前端 UI 表现 | ChatView，单头像，无切换 | OfficeView（群聊），多头像，动态切换 |
| 失败处理 | 直接返回错误 | Orchestrator 降级策略（一个 Agent 失败，其他继续） |

---

## 2. 完整路由决策链路
### 2.1 三层决策架构
```text
┌─────────────────────────────────────────────────────────────────┐
│                     决策链路（自上而下）                          │
│                                                                 │
│  ┌─────────────────────┐                                        │
│  │ 数据库层              │  conversations.type                   │
│  │ → direct / group     │  决定会话的基础属性                     │
│  └─────────┬───────────┘                                        │
│            ↓                                                     │
│  ┌─────────────────────┐                                        │
│  │ Java 后端层           │  WebSocketController                 │
│  │ → 查 type，决定传参   │  agentType = 具体值 / null             │
│  └─────────┬───────────┘                                        │
│            ↓                                                     │
│  ┌─────────────────────┐                                        │
│  │ Python Agent 服务层   │  main.py                             │
│  │ → 判断 agentType     │  有值→单Agent / 空→Orchestrator        │
│  └─────────────────────┘                                        │
└─────────────────────────────────────────────────────────────────┘
```
### 2.2 决策流程图
```text
用户在聊天窗口发送消息
        │
        ▼
┌───────────────────┐
│ Java 后端           │
│ WebSocketController│
└────────┬──────────┘
        │
        │ 1. 查询 conversation.type
        ▼
   ┌────────────┐
   │ type 是什么？│
   └──┬──────┬──┘
      │      │
  direct   group
      │      │
      ▼      ▼
┌──────────┐ ┌──────────────┐
│ 单聊模式  │ │ 群聊模式      │
│          │ │              │
│ 获取会话  │ │ agentType    │
│ 默认Agent │ │ = null       │
│ 的类型    │ │              │
│          │ │ （用户可临时   │
│ agentType│ │  指定覆盖）    │
│ ="claude │ │              │
│  _code"  │ │              │
└────┬─────┘ └──────┬───────┘
     │              │
     └──────┬───────┘
            │
            │ HTTP POST /api/agent/chat
            ▼
┌───────────────────┐
│ Python Agent 服务  │
│ main.py           │
└────────┬──────────┘
        │
        │ 2. 判断 request.agentType
        ▼
   ┌──────────────┐
   │ agentType     │
   │ 是否为空？     │
   └──┬────────┬──┘
      │        │
     有值      空
      │        │
      ▼        ▼
┌──────────┐ ┌──────────────┐
│ 单 Agent  │ │ 多 Agent      │
│ 模式      │ │ 模式          │
│          │ │              │
│ 适配器工厂│ │ Orchestrator │
│ .get()   │ │ .execute()   │
│          │ │              │
│ 直接调用  │ │ LLM生成计划   │
│ Agent    │ │ 串行调用Agent │
│          │ │              │
│ 不推送    │ │ 推送          │
│ agent_   │ │ agent_switch  │
│ switch   │ │ + chunk事件   │
└────┬─────┘ └──────┬───────┘
     │              │
     └──────┬───────┘
            │
            │ SSE 流式返回
            ▼
┌───────────────────┐
│ Java 后端          │
│ WebClient 接收流   │
└────────┬──────────┘
        │
        │ STOMP WebSocket 推送
        ▼
┌───────────────────┐
│ Vue 前端           │
│ 根据事件类型渲染   │
└───────────────────┘
```

---

## 3. 各层实现代码
### 3.1 Java 后端：WebSocketController
```java
// backend-java/src/main/java/com/agenthub/controller/WebSocketController.java

@Controller
public class WebSocketController {

    private final SimpMessagingTemplate messagingTemplate;
    private final MessageService messageService;
    private final AgentGatewayService agentGatewayService;
    private final ConversationRepository conversationRepository;

    // 构造函数注入省略...

    /**
     * 接收用户消息，根据会话类型路由到不同的 Agent 调用模式
     */
    @MessageMapping("/chat.send")
    public void handleUserMessage(@Payload SendMessageRequest request) {

        // ========================================
        // 第 1 步：保存用户消息
        // ========================================
        messageService.saveUserMessage(
            request.getConversationId(),
            request.getContent()
        );

        // ========================================
        // 第 2 步：判断会话类型，决定 agentType
        // ========================================
        Conversation conversation = conversationRepository
            .findById(request.getConversationId())
            .orElseThrow(() -> new NotFoundException("会话不存在"));

        String agentType = resolveAgentType(request, conversation);

        // ========================================
        // 第 3 步：构建上下文
        // ========================================
        List<Map<String, Object>> context = messageService
            .buildContext(request.getConversationId());

        // ========================================
        // 第 4 步：调用 Python Agent 服务
        // ========================================
        Flux<String> agentStream = agentGatewayService.sendToAgent(
            agentType,      // 单聊：具体类型；群聊：null
            null,            // systemPrompt 使用 Agent 默认值
            context
        );

        // ========================================
        // 第 5 步：流式转发给前端
        // ========================================
        String conversationId = request.getConversationId();

        agentStream.subscribe(
            chunk -> messagingTemplate.convertAndSend(
                "/topic/conversation." + conversationId,
                parseChunk(chunk)
            ),
            error -> messagingTemplate.convertAndSend(
                "/topic/conversation." + conversationId,
                buildErrorChunk(error)
            ),
            () -> {
                // 流结束，保存完整的 Agent 回复
                // messageService.saveAgentMessage(conversationId, ...)
            }
        );
    }

    /**
     * 根据会话类型和用户请求，决定传给 Python 的 agentType
     *
     * @param request      前端发来的消息请求
     * @param conversation 数据库中的会话记录
     * @return 具体的 Agent 类型字符串，或 null（表示由 Orchestrator 自动决策）
     */
    private String resolveAgentType(SendMessageRequest request, Conversation conversation) {

        // 优先级 1：用户在输入框中临时选择了 Agent（覆盖会话默认）
        if (request.getAgentType() != null && !request.getAgentType().isEmpty()) {
            return request.getAgentType();
        }

        // 优先级 2：单聊模式 → 使用会话关联的第一个 Agent
        if ("direct".equals(conversation.getType())) {
            Set<Agent> agents = conversation.getAgents();
            if (agents != null && !agents.isEmpty()) {
                return agents.iterator().next().getType();
            }
            // 单聊但没有关联 Agent → 降级为 Orchestrator 决策
            return null;
        }

        // 优先级 3：群聊模式 → Orchestrator 自动决策
        // conversation.type == "group"
        return null;
    }
}
```
### 3.2 Python Agent 服务：main.py
```python
# agent-service/main.py

import json
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from models import ChatRequest
from adapters.adapter_factory import AdapterFactory
from orchestrator import LLMOrchestrator

app = FastAPI(title="AgentHub Agent Service")

adapter_factory = AdapterFactory()
orchestrator = LLMOrchestrator(adapter_factory, planner_llm)


@app.post("/api/agent/chat")
async def agent_chat(request: ChatRequest):
    """
    统一的 Agent 对话接口
    根据 agentType 参数自动路由到不同的处理模式

    路由规则：
    - agentType 有值 → 单 Agent 模式：直接调用对应适配器
    - agentType 为空或 "orchestrator" → 多 Agent 模式：启动 Orchestrator
    """
    # ========================================
    # 路由判断
    # ========================================
    is_single_agent = request.agentType and request.agentType != "orchestrator"

    if is_single_agent:
        return await handle_single_agent(request)
    else:
        return await handle_multi_agent(request)


async def handle_single_agent(request: ChatRequest):
    """
    单 Agent 模式：
    - 直接调用对应适配器
    - 不推送 agent_switch 事件（因为只有一个 Agent 在说话）
    - 每个 chunk 仍然携带 agentId 和 agentName（保持格式一致）
    """
    adapter = adapter_factory.get_adapter(request.agentType)
    agent_id = f"agent_{request.agentType}_001"
    agent_name = get_agent_display_name(request.agentType)

    if request.stream:
        return StreamingResponse(
            single_agent_stream(adapter, request, agent_id, agent_name),
            media_type="text/event-stream"
        )
    else:
        result = await adapter.chat(
            request.systemPrompt,
            format_context(request.messages)
        )
        return {"content": result, "messageId": generate_id()}


async def single_agent_stream(adapter, request, agent_id: str, agent_name: str):
    """
    单 Agent 流式响应：
    - 只推送 chunk 事件，不推送 agent_switch
    - 每个 chunk 中固定携带同一个 agentId 和 agentName
    """
    context = format_context(request.messages)

    async for token in adapter.chat_stream(request.systemPrompt, context):
        chunk = json.dumps({
            "type": "chunk",
            "content": token,
            "finish": False,
            "agentId": agent_id,
            "agentName": agent_name
        }, ensure_ascii=False)
        yield f"data: {chunk}\n\n"

    # 流结束
    final = json.dumps({
        "type": "finish",
        "finish": True,
        "messageId": generate_id()
    }, ensure_ascii=False)
    yield f"data: {final}\n\n"


async def handle_multi_agent(request: ChatRequest):
    """
    多 Agent 模式（群聊）：
    - 启动 Orchestrator 自动拆解任务
    - 推送 agent_switch 事件 + chunk 事件
    """
    if request.stream:
        return StreamingResponse(
            multi_agent_stream(request),
            media_type="text/event-stream"
        )
    else:
        result = await orchestrator.execute_sync(
            request.messages[-1].content
        )
        return {"content": result, "messageId": generate_id()}


async def multi_agent_stream(request):
    """
    多 Agent 流式响应：
    - Orchestrator 按顺序推送 agent_switch 和 chunk
    - 前端根据事件类型动态切换头像和名称
    """
    user_message = request.messages[-1].content if request.messages else ""

    async for event in orchestrator.execute(user_message):
        if event["type"] == "agent_switch":
            # Agent 切换事件
            chunk = json.dumps({
                "type": "agent_switch",
                "agentId": event["agentId"],
                "agentName": event["agentName"]
            }, ensure_ascii=False)
        elif event["type"] == "chunk":
            # 内容片段
            chunk = json.dumps({
                "type": "chunk",
                "content": event["content"],
                "finish": False,
                "agentId": event["agentId"],
                "agentName": event["agentName"]
            }, ensure_ascii=False)
        elif event["type"] == "finish":
            # 消息结束
            chunk = json.dumps({
                "type": "finish",
                "finish": True,
                "messageId": event["messageId"]
            }, ensure_ascii=False)
        else:
            continue

        yield f"data: {chunk}\n\n"


def get_agent_display_name(agent_type: str) -> str:
    """根据 Agent 类型返回显示名称"""
    names = {
        "claude_code": "Claude Code",
        "codex": "Codex",
        "custom": "Custom Agent"
    }
    return names.get(agent_type, agent_type)


def generate_id() -> str:
    """生成 UUID"""
    import uuid
    return str(uuid.uuid4())
```
### 3.3 前端：chatStore.js
```javascript
// frontend/src/stores/chatStore.js

import { defineStore } from 'pinia';
import { ref } from 'vue';
import wsClient from '../websocket/wsClient';

export const useChatStore = defineStore('chat', () => {

  // 当前活跃的会话
  const activeConversationId = ref(null);

  // 消息列表（已完成的消息）
  const messages = ref([]);

  // 当前正在流式传输的消息
  const streamingMessage = ref({
    agentId: '',
    agentName: '',
    content: '',
    isStreaming: false,
    isMultiAgent: false  // 是否处于多 Agent 模式
  });

  /**
   * 发送消息
   * @param {string} content - 消息内容
   * @param {string|null} agentType - 临时选择的 Agent（可选）
   */
  function sendMessage(content, agentType = null) {
    if (!activeConversationId.value) return;

    // 1. 本地添加用户消息气泡
    messages.value.push({
      senderType: 'user',
      content: content,
      messageType: 'text',
      createdAt: new Date().toISOString()
    });

    // 2. 通过 WebSocket 发送
    wsClient.sendMessage(
      activeConversationId.value,
      content,
      agentType  // 单聊可传具体值，群聊不传
    );
  }

  /**
   * 处理收到的 WebSocket 消息
   */
  function handleIncomingMessage(data) {
    switch (data.type) {

      case 'agent_switch':
        // ========================================
        // 多 Agent 模式：切换当前发言人
        // ========================================
        streamingMessage.value.agentId = data.agentId;
        streamingMessage.value.agentName = data.agentName;
        streamingMessage.value.isMultiAgent = true;

        // 如果之前已有流式内容（前一个 Agent 说完了），先固化
        if (streamingMessage.value.content.length > 0) {
          messages.value.push({
            senderType: 'agent',
            agentId: streamingMessage.value.agentId,
            agentName: streamingMessage.value.agentName,
            content: streamingMessage.value.content,
            messageType: 'text',
            createdAt: new Date().toISOString()
          });
          streamingMessage.value.content = '';
        }
        break;

      case 'chunk':
        // ========================================
        // 内容片段：追加到流式消息
        // ========================================
        if (!streamingMessage.value.isStreaming) {
          streamingMessage.value.isStreaming = true;
        }
        // 始终以 chunk 中的 agentId 为准（冗余保障）
        streamingMessage.value.agentId = data.agentId;
        streamingMessage.value.agentName = data.agentName;
        streamingMessage.value.content += data.content;
        break;

      case 'finish':
        // ========================================
        // 消息结束：固化到消息列表
        // ========================================
        if (streamingMessage.value.content.length > 0) {
          messages.value.push({
            senderType: 'agent',
            agentId: streamingMessage.value.agentId,
            agentName: streamingMessage.value.agentName,
            content: streamingMessage.value.content,
            messageType: streamingMessage.value.isMultiAgent ? 'text' : 'text',
            createdAt: new Date().toISOString()
          });
        }
        // 重置流式状态
        streamingMessage.value = {
          agentId: '',
          agentName: '',
          content: '',
          isStreaming: false,
          isMultiAgent: false
        };
        break;
    }
  }

  return {
    activeConversationId,
    messages,
    streamingMessage,
    sendMessage,
    handleIncomingMessage
  };
});
```

---

## 4. 两种模式的 SSE 事件流对比
### 4.1 单 Agent 模式（用户选择 Claude Code）
```text
SSE 事件流（简化表示）：

data: {"type":"chunk","content":"好的","finish":false,"agentId":"agent_claude_code_001","agentName":"Claude Code"}

data: {"type":"chunk","content":"，这是","finish":false,"agentId":"agent_claude_code_001","agentName":"Claude Code"}

data: {"type":"chunk","content":"生成的代码","finish":false,"agentId":"agent_claude_code_001","agentName":"Claude Code"}

data: {"type":"finish","finish":true,"messageId":"msg_456"}
```
### 4.2 多 Agent 模式（群聊，Orchestrator 调度）
```text
SSE 事件流（简化表示）：

data: {"type":"agent_switch","agentId":"agent_claude_code_001","agentName":"Claude Code"}

data: {"type":"chunk","content":"我来设计前端页面...","finish":false,"agentId":"agent_claude_code_001","agentName":"Claude Code"}

data: {"type":"chunk","content":"代码完成","finish":false,"agentId":"agent_claude_code_001","agentName":"Claude Code"}

data: {"type":"agent_switch","agentId":"agent_codex_001","agentName":"Codex"}

data: {"type":"chunk","content":"我来审查前端代码...","finish":false,"agentId":"agent_codex_001","agentName":"Codex"}

data: {"type":"chunk","content":"发现3个问题","finish":false,"agentId":"agent_codex_001","agentName":"Codex"}

data: {"type":"agent_switch","agentId":"agent_claude_code_001","agentName":"Claude Code"}

data: {"type":"chunk","content":"根据审查意见修改...","finish":false,"agentId":"agent_claude_code_001","agentName":"Claude Code"}

data: {"type":"finish","finish":true,"messageId":"msg_789"}
```
### 特点：
每个 Agent 开始发言前推送 agent_switch

前端的头像和名称随 agent_switch 动态切换

即使 agent_switch 丢失，chunk 中也携带 agentId（冗余保障）

---

## 5. 边界情况处理
### 5.1 单聊但会话没有关联 Agent
```java
// Java 后端处理
if ("direct".equals(conversation.getType())) {
    Set<Agent> agents = conversation.getAgents();
    if (agents == null || agents.isEmpty()) {
        // 降级：让 Orchestrator 自动决策
        agentType = null;
    }
}
```
### 5.2 群聊但用户临时指定了 Agent
```javascript
// 前端：群聊中输入框支持 @Agent 或下拉选择
wsClient.sendMessage({
  conversationId: 'conv_group_001',
  content: '帮我写前端代码',
  agentType: 'claude_code'  // 用户临时指定
});
```
```java
// Java 后端：优先使用前端传来的 agentType
private String resolveAgentType(SendMessageRequest request, Conversation conversation) {
    // 优先级 1：用户临时选择（即使是群聊也生效）
    if (request.getAgentType() != null && !request.getAgentType().isEmpty()) {
        return request.getAgentType();
    }
    // ...后续逻辑
}
```
此时 Python Agent 服务收到的是有值的 `agentType`，走单 Agent 模式（不经过 Orchestrator）。

### 5.3 Orchestrator 中某个 Agent 调用失败
```python
# orchestrator.py
async def execute(self, user_message: str) -> AsyncGenerator[dict, None]:
    plan = await self._generate_plan(user_message)
    results = {}

    for i, step in enumerate(plan.steps):
        try:
            # ... 调用 Agent
            results[i] = full_response
        except Exception as e:
            # 降级：记录错误，继续执行后续步骤
            results[i] = f"[错误] Agent {step.agent} 执行失败: {str(e)}"
            yield {
                "type": "chunk",
                "content": f"\n⚠️ {step.agent} 执行失败，跳过此步骤。\n",
                "agentId": "system",
                "agentName": "System"
            }

    yield {"type": "finish", "messageId": generate_id()}
```

---

## 关联文档索引：
>数据模型设计 → conversations 表 type 字段

>API 契约定义与通信协议规范 → 第 3.3.2 节 agent_switch 事件

>API 契约定义与通信协议规范 → 第 4.2 节 内部 HTTP 接口 agentType 参数

>Orchestrator 调度器设计方案 → 完整执行流程

>架构评审问题清单 → P3 agent_switch 事件契约

