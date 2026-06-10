# 开发过程记录 — PR #6 Agent 注册表与系统提示词审查合并

> **日期**: 2026-06-05\
> **关联分支**: `review/pr-6-zsy125`（审查） → `dev`（直接合并）\
> **PR 作者**: zsy125，fork 仓库 → `IAyousa/Agenthub_demo`\
> **PR 概要**: Python Agent 服务增强，5 个文件变更（+282/-3 行）

---

## 一、背景

### 1.1 PR #6 是什么

成员 zsy125 从 fork 仓库 `zsy125/Agenthub_demo:dev` 向主仓库 `IAyousa/Agenthub_demo:dev` 提交了 PR #6。变更范围集中在 Python Agent 服务：新增 Agent 注册表模块、增强系统提示词、配置扩展。

### 1.2 当前项目状态

在审查 PR #6 之前，`dev` 分支已完成：
- **PR #7 合并**：ConversationController、MessageController、WebSocketSessionManager、WebSocketController handler 全部就绪
- **文档全面审查**：8 份文档已与实际代码对齐
- **端到端消息链路**：前端 STOMP → Java WebSocketController → Python Agent Service → SSE 流式回复完整跑通

**Python Agent 服务的已知缺口**：
- `systemPrompt` 从 Java 传入时始终为空字符串 `""`，Agent 收不到角色提示词
- 系统提示词只有 4 个通用角色（coder/designer/reviewer/architect），与 agent type（claude_code/codex）无对应关系
- Agent 元数据（能力、角色定义）在 Java DB 中存在，但 Python 侧无访问途径

---

## 二、PR #6 代码审查

### 2.1 审查方法

```bash
# zsy125 远程仓库已在先前添加
git fetch zsy125 dev
git checkout -b review/pr-6-zsy125 zsy125/dev

# 对比 origin/dev 和 zsy125/dev 的差异
git diff origin/dev...zsy125/dev --stat   # 5 files, +282/-3
git diff origin/dev...zsy125/dev           # 逐文件审查
```

### 2.2 变更清单

| # | 文件 | 行数 | 内容 |
|---|------|------|------|
| 1 | `agents.py`（新建） | +75 | Agent 注册表工具模块：get_agents() / get_agent() / agent_exists() |
| 2 | `config.py` | +71 | 新增 AGENT_REGISTRY：4 个 Agent 的完整元数据 |
| 3 | `system_prompts.py` | +119 | 4 个新提示词：claude_code / codex / orchestrator / custom |
| 4 | `requirements.txt` | +2/-1 | httpx 注释修正 |
| 5 | `基础设施配置.md` | +1/-1 | 快速启动步骤补充 Swagger UI |

### 2.3 审查发现：技术栈硬编码问题

`system_prompts.py` 的 `codex` 提示词写死了 React 技术栈：

> "你是一个专业的 React 前端开发专家"\
> "React 18+（函数组件 + Hooks）"\
> "useMemo、useCallback、React.memo"

**问题**：AgentHub 是通用平台，用户的项目可能是 Vue/Angular 等任意前端框架。且当前 AgentHub 项目本身也是 Vue 3。Agent 应该根据项目实际代码自适应技术栈。

### 2.4 审查发现：架构决策需要前置讨论

PR #6 的 `AGENT_REGISTRY` 和 `agents.py` 在 Python 端维护了一份 Agent 元数据。但 Java DB 的 `agents` 表已经是 Agent 数据的持久化存储。两份数据如何协调，需要先决定架构方案。

**关键矛盾**：Orchestrator 设计文档将调度逻辑放在 Python 端，这意味着 Python 确实需要 Agent 元数据。但"Java 是唯一数据源"是项目的核心架构原则。

---

## 三、架构设计讨论（关键决策）

### 3.1 问题：Agent 元数据应该放在哪？

用户与 AI 经过多轮讨论，分析了 3 种方案：

| 方案 | 描述 | 优劣 |
|------|------|------|
| **A** — Python 自维护（PR #6 方向） | Python config.py 定义 Agent 角色元数据 | 简单但两份数据需手动同步 |
| **B** — Java 每次请求携带 | Java 从 DB 查出 Agent 列表放入请求体 | 单一数据源，但每次多传 ~1KB |
| **C** — Hybrid（最终选择） | Python config fallback + Java 请求携带 + P1 迁 Redis | 兼顾 MVP 简单性和长期架构 |

### 3.2 最终方案：三层演进

```
MVP（当前）          →  P1                →  P1 末期

config.py            Java 请求携带          Redis 共享缓存
AGENT_REGISTRY       availableAgents[]      Java 写入 → Python 读取
(fallback缓存)       (从DB实时查询)        (复用已有Redis实例)
```

关键决策点：
- **Java DB 是唯一权威数据源** — 这条原则不动摇
- **Python AGENT_REGISTRY 是 fallback** — MVP 阶段的降级兜底，不是替代数据源
- **方案 B 的"1KB 额外网络开销"不是问题** — 远小于 LLM 流式回复的量级
- **Redis 演进是自然路径** — 项目已有 "WebSocket 会话管理 ConcurrentHashMap → Redis" 的迁移计划

### 3.3 重新评估 PR #6 的接受决策

基于上述架构决策，重新评估结果：

| 文件 | 初始判断 | 最终决策 | 原因 |
|------|:--:|:--:|------|
| agents.py | ❌ 拒绝 | ✅ 接受 | 作为 AGENT_REGISTRY 的工具模块，有明确的消费者 |
| config.py AGENT_REGISTRY | ❌ 拒绝 | ✅ 接受（修改） | 定位从"独立数据源"改为"fallback 缓存" |
| system_prompts.py | ✅ 接受（修改） | ✅ 接受（修改） | 去除技术栈硬编码 + 保留旧版 fallback |

---

## 四、代码改造实施

### 4.1 PR #6 代码改造

**system_prompts.py** — 技术栈通用化：

| 角色 | PR #6 原版 | 修改后 |
|------|-----------|--------|
| codex | "React 18+（函数组件 + Hooks）" | "根据项目现有技术栈自动适配（Vue/React/Angular 等）" |
| codex | "useMemo、useCallback、React.memo" | "合理使用缓存、懒加载、代码分割等优化手段" |
| claude_code | "Python/Java/Go" | 改为通用的代码规范要求 |

**旧版 4 个提示词（coder/designer/reviewer/architect）保留**，作为无 promptKey 匹配时的兜底。

### 4.2 MVP 接线（PR #6 没包含的部分）

**【Python 侧】messages.py** — systemPrompt fallback 逻辑（+6 行）：

```python
if not system_prompt and agent_type:
    from prompts.system_prompts import SYSTEM_PROMPTS
    system_prompt = SYSTEM_PROMPTS.get(agent_type, SYSTEM_PROMPTS.get("coder", ""))
```

**【Java 侧】WebSocketController.java** — 从 DB 读取 systemPrompt（~15 行改动）：

```java
// 修改前（硬编码空字符串）
agentGatewayService.sendToAgent(context, "claude_code", "", callback);

// 修改后（从 DB 读取 Agent 实体）
Agent agent = agentRepository.findById("agent_claude_001").orElse(null);
String systemPrompt = (agent != null && agent.getSystemPrompt() != null)
        ? agent.getSystemPrompt() : "";
agentGatewayService.sendToAgent(context, agentType, systemPrompt, callback);
```

**【P2 预留】models.py** — availableAgents 字段（+6 行）：

```python
availableAgents: list = Field(
    default_factory=list,
    description="P2预留：Java传入的可用Agent列表，用于Orchestrator调度决策",
)
```

### 4.3 文档同步

| 文档 | 更新内容 |
|------|----------|
| CLAUDE.md | agent-service 模块清单更新、完成度 ~75%→~80%、架构说明补充 |
| API 契约 §4 | 请求字段表新增 `availableAgents` + systemPrompt fallback 说明 |
| Orchestrator 设计文档 | 新增 §6 "Agent 元数据存储与传递"三层演进设计 |
| 架构拓扑图 | 目录树新增 `agents.py`、描述表同步 |

---

## 五、手动测试验证

采用分阶段测试策略：先 Python 侧隔离验证，再端到端。

### 5.1 Python 侧测试（Swagger UI）

| # | 测试输入 | 预期 | 结果 |
|---|---------|------|:--:|
| 2-a | `systemPrompt=""` agentType=`claude_code` | 自动 fallback 到 `SYSTEM_PROMPTS["claude_code"]` | ✅ Agent 回复体现全栈工程师角色 |
| 2-b | `systemPrompt="你是诗人"` | 手动 prompt 覆盖 fallback | ✅ Agent 回复五言绝句 |
| 2-c | agentType=`unknown_type` | 返回 400 错误 | ✅ `"不支持的 Agent 类型: unknown_type"` |

### 5.2 端到端测试

用户发送："请告诉我你是谁，能干什么，主要负责什么工作"

Agent 回复中提到"AgentHub 多智能体协作平台"和三层架构细节，证明：
1. Java 从 DB 成功读取了 `agent.systemPrompt`
2. Python 成功接收并使用了 role-specific 提示词
3. STOMP → HTTP+SSE → STOMP 完整链路正常

### 5.3 Java 日志验证

```
INFO: select a1_0...system_prompt...from agents a1_0 where a1_0.id=?
```

此行日志证实新增了 agent 表查询（之前不会查 agents 表）。

---

## 六、最终产物

### 6.1 代码变更

| 操作 | 文件 | 说明 |
|:--:|------|------|
| 新建 | `agents.py` | Agent 注册表工具模块 |
| 修改 | `config.py` | 新增 AGENT_REGISTRY（+70 行） |
| 修改 | `system_prompts.py` | 增强为 8 角色提示词（+138 行） |
| 修改 | `messages.py` | systemPrompt 自动 fallback（+6 行） |
| 修改 | `WebSocketController.java` | 从 DB 读取 systemPrompt（~15 行改动） |
| 修改 | `models.py` | availableAgents P2 预留字段（+6 行） |
| 修改 | `requirements.txt` | 注释修正 |

### 6.2 提交记录

```
e454954 docs: 同步4份项目文档对齐Agent元数据架构设计+PR #6合并
3098d6e feat(agent-service): 合并PR #6增强系统提示词+Agent注册表，接线systemPrompt端到端链路
```

### 6.3 代码统计

| 指标 | 数值 |
|------|------|
| 新建文件 | 1 |
| 修改文件 | 6 |
| 文档更新 | 4 |
| 新增代码 | ~320 行 |
| 修改代码 | ~30 行 |
| 测试次数 | 5（3 Python + 1 端到端 + 1 日志验证） |

---

## 七、经验总结

### 7.1 架构讨论应该前置

PR #6 最初被判断为"AGENT_REGISTRY 不该在 Python 端"，但在与用户讨论 Orchestrator 定位后才纠正。如果在审查一开始就读 Orchestrator 设计文档，可以更早得出正确的判断。

**教训**：审查代码前先审查关联的设计文档，理解"为什么这样设计"后再判断"代码写得对不对"。

### 7.2 "小 PR"的审查成本更低

PR #6 只有 5 个文件 +282 行，远少于 PR #7 的 50 个文件 +2465 行。没有架构冲突（如 fork 落后导致的误删除），审查周期短、决策清晰。这验证了小粒度 PR 的团队协作价值。

### 7.3 审查中发现的技术债应立即修复

Java 硬编码 `systemPrompt=""` 是在审查 PR #6 时暴露的已有问题。在合并 PR #6 代码的同时修复它（从 DB 读取），比"创建一个新 issue 以后修"更高效。审查窗口是发现和修复相邻问题的好时机。

### 7.4 通用化 vs 具体化的平衡

PR #6 的 `claude_code` 提示词写得很好（具体的代码规范、质量要求），但 `codex` 提示词过度绑定了 React。通用化时保持了"角色能力边界"的具体性（组件开发、样式实现、状态管理），去掉了"特定框架 API"的绑定（useMemo、React.memo）。原则是：**角色定义要具体，技术栈选择要灵活**。
