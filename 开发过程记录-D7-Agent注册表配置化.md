# 开发过程记录 — Agent 注册表配置化

> **日期**: 2026-06-03\
> **任务编号**: C8\
> **所属模块**: agent-service\
> **前置依赖**: B1 Agent 服务本地 CLI 适配器重构 ✅（已完成）

---

## 一、背景

Agent 服务的适配器工厂 `adapter_factory.py` 中硬编码了 Agent 类型映射（`claude_code` → `ClaudeAdapter`、`codex` → `CodexAdapter`），但缺少一份统一的 Agent 元数据注册表。后续其他模块（如 Orchestrator 任务调度、前端 Agent 选择器）需要查询"有哪些 Agent 可用、各自擅长什么"，如果各模块自行维护一份列表，会导致数据不一致。

本任务创建一份统一的 Agent 注册表，作为单一数据源（Single Source of Truth），供 Python 端所有模块通过 `import` 直接使用。

经过三轮方案迭代：

| 轮次 | 方案 | 问题 |
|------|------|------|
| 第一轮 | 新建 `agents.yaml` + PyYAML 解析 → REST API 端点 | 引入新依赖（pyyaml），且 Agent 元数据不需要对外暴露为 HTTP 接口 |
| 第二轮 | 改为 `.env` 中 `AGENT_REGISTRY` JSON 字符串 | `.env` 中一长串 JSON 不便于阅读和编辑；env 应只放密钥类环境变量 |
| 第三轮（最终） | `config.py` 中定义 Python list，`agents.py` 提供纯函数 | 无新依赖，代码即配置，Python 端直接 import |

---

## 二、任务执行过程

### 2.1 第一轮：agents.yaml + REST API

**操作内容**：新建 `agents.yaml`（4 个 Agent 定义），新建 `app/api/endpoints/agents.py`（YAML 加载 + `GET /api/agents` 和 `GET /api/agents/{id}` 两个端点），在 `main.py` 注册路由，在 `requirements.txt` 添加 `pyyaml>=6.0`。

**用户反馈**：agents.py 只需要提供获取 agent 列表的函数，没必要包装成 REST 接口。

### 2.2 第二轮：去除 REST 接口 + 改从 .env 读取

**操作内容**：

1. 将 `agents.py` 中所有 router、端点函数、`_error_json` 删除，只保留 `get_agents`/`get_agent`/`agent_exists`/`reload_agents` 四个纯函数
2. 在 `config.py` 新增 `AGENT_REGISTRY: str = json.dumps([...])` 字段，提供 `get_agent_registry()` 方法做 JSON 解析
3. 在 `.env` 末尾追加 `AGENT_REGISTRY=[{...}]` 一长行 JSON
4. 从 `main.py` 移除 agents 路由注册和导入

**用户反馈**：把 config.py 作为配置文件，env 中 Agent 注册表可以去掉。

### 2.3 第三轮（最终）：config.py 纯 Python list

**操作内容**：

1. 将 `config.py` 中 `AGENT_REGISTRY` 字段类型从 `str`（JSON 字符串）改为 `list`（Python 列表），直接在代码中定义 4 个 Agent 的元数据字典：

```python
# config.py — 最终版本
from pydantic_settings import BaseSettings
from typing import List

class Settings(BaseSettings):
    # ... 其他配置 ...
    
    AGENT_REGISTRY: list = [
        {
            "id": "claude_code",
            "name": "Claude Code",
            "type": "claude_code",
            "description": "全栈工程师，擅长后端逻辑、代码审查、架构设计与重构优化",
            "model": "anthropic/claude-sonnet-4-20250514",
            "promptKey": "claude_code",
            "capabilities": [...],
            "tags": [...],
            "status": "active",
        },
        # ... codex, orchestrator, custom
    ]
```

2. 从 `config.py` 移除 `import json` 和 `get_agent_registry()` 方法
3. 从 `.env` 移除 `AGENT_REGISTRY` 行（env 恢复为仅含密钥）
4. 更新 `agents.py`：`_AGENTS = settings.AGENT_REGISTRY`（直接从 `settings` 取 list，无需 JSON 解析）

**设计决策**：

| 决策项 | 选择 | 理由 |
|--------|------|------|
| Agent 注册表存放位置 | `config.py` Python list | 代码即配置，IDE 有语法高亮和自动补全；pydantic-settings 的 `extra="ignore"` 已确保新增字段不会冲突 |
| 是否对外暴露 REST API | 不暴露 | Agent 注册表是 Python 内部数据，前端/Java 通过现有接口即可获取 Agent 信息，无需新增端点 |
| `agents.py` 的 `_AGENTS` | 缓存引用 `settings.AGENT_REGISTRY` | 不拷贝数据，所有模块共享同一份列表；`reload_agents()` 重新从 settings 取值即可 |
| 注册表字段设计 | id/name/type/description/model/promptKey/capabilities/tags/status | 覆盖适配器映射、UI 展示、Orchestrator 调度三类场景所需信息 |

---

## 三、技术要点讲解

### 3.1 pydantic-settings 的 list 类型字段

`pydantic-settings` 支持 `list` 类型字段，配合类中定义的默认值即可使用，无需 `.env` 覆盖：

```python
class Settings(BaseSettings):
    AGENT_REGISTRY: list = [{"id": "claude_code", ...}]

    class Config:
        env_file = ".env"
        extra = "ignore"  # 关键：忽略 .env 中的旧字段，不报错

settings = Settings()
print(type(settings.AGENT_REGISTRY))  # <class 'list'>
print(len(settings.AGENT_REGISTRY))  # 4
```

如果 `.env` 中需要覆盖，pydantic-settings 会将逗号分隔的值解析为 list 元素，但对于复杂嵌套结构（dict 列表），不建议在 `.env` 中维护，直接在代码默认值中定义更清晰。

### 3.2 纯工具模块 vs FastAPI 端点模块

`agents.py` 不注册任何 FastAPI router，是一个纯 Python 工具模块：

```python
# agents.py — 供其他模块 import 使用
from app.api.endpoints.agents import get_agents, get_agent

claude = get_agent("claude_code")  # 返回 dict 或 None
```

这种设计让模块职责单一：它只负责数据读取，不关心如何对外暴露。如果未来确实需要 HTTP 端点，在调用方（如 `main.py` 或新的 endpoint 文件）封装一层即可。

---

## 四、验证测试

### 4.1 自动化验证

| 测试项 | 命令 | 结果 |
|--------|------|------|
| config.py 中 AGENT_REGISTRY 类型 | `python -c "from config import settings; print(type(settings.AGENT_REGISTRY))"` | `<class 'list'>` ✅ |
| agents.py 工具函数工作正常 | `python -c "from app.api.endpoints.agents import get_agents, get_agent, agent_exists; ..."` | 4 个 Agent 全部可查 ✅ |
| FastAPI 启动无报错 | `python -c "from main import app; print(app.routes)"` | 路由不含 /api/agents ✅ |
| .env 无 AGENT_REGISTRY 残留 | 查看 .env 文件 | 仅含 DEEPSEEK_API_KEY 和注释 ✅ |
| requirements.txt 无 pyyaml | 查看 requirements.txt | 已移除 ✅ |

### 4.2 手动功能测试清单

| # | 测试项 | 预期行为 | 结果 |
|---|--------|---------|------|
| 1 | `get_agents()` 返回全部 Agent | 返回 4 个 dict，id 分别为 claude_code/codex/orchestrator/custom | ✅ |
| 2 | `get_agent("claude_code")` | 返回 Claude Code 完整元数据，含 capabilities/tags | ✅ |
| 3 | `get_agent("unknown")` | 返回 None | ✅ |
| 4 | `agent_exists("codex")` | 返回 True | ✅ |
| 5 | `agent_exists("nonexistent")` | 返回 False | ✅ |
| 6 | `get_agents(status="active")` | 返回 4 个活跃 Agent | ✅ |
| 7 | `reload_agents()` | 返回最新列表（当前无变化） | ✅ |

---

## 五、关键决策记录

| # | 决策 | 结论 | 理由 |
|---|------|------|------|
| 1 | 配置文件格式 | `config.py` Python list，不用 YAML/JSON | 零依赖，IDE 友好，pydantic-settings 原生支持 |
| 2 | Agent 元数据是否对外暴露 HTTP | 不暴露，仅内部 import | 当前无外部消费方需要此端点，避免 API 膨胀 |
| 3 | 注册表定义位置 | `config.py` 而非 `.env` | env 适合密钥/环境差异变量；Agent 定义是业务配置，应在代码中维护 |
| 4 | pyyaml 依赖 | 移除 | 目前无其他模块依赖 YAML，保留会误导后续维护者 |
| 5 | `_AGENTS` 缓存策略 | 直接引用 `settings.AGENT_REGISTRY` | list 是可变对象，引用即可，无需深拷贝 |

---

## 六、产物清单

| 文件 | 操作 | 行数变化 | 说明 |
|------|------|---------|------|
| `agent-service/app/api/endpoints/agents.py` | 新增 | +76 | Agent 注册表工具模块（4 个纯函数） |
| `agent-service/config.py` | 修改 | +75 | 新增 `AGENT_REGISTRY: list` 字段（含 4 个 Agent 的完整元数据） |
| `agent-service/main.py` | 恢复 | -4 | 移除 agents 导入和路由注册（回归原有结构） |
| `agent-service/.env` | 恢复 | -10 | 移除 AGENT_REGISTRY 行（回归密钥配置） |
| `agent-service/agents.yaml` | 已删除 | — | 第一轮产物，最终被 config.py 替代 |
| `agent-service/requirements.txt` | 修改 | -3 | 移除 pyyaml>=6.0（第一轮残留） |

### 代码统计

| 指标 | 数值 |
|------|------|
| 新增文件 | 1（agents.py） |
| 修改文件 | 4（config.py、main.py、.env、requirements.txt） |
| 删除文件 | 1（agents.yaml） |
| 新增代码行 | +151 |
| 净减少代码行 | -17（移除了 REST 端点样板代码和 YAML 依赖） |

---

## 七、数据流/架构影响

**Agent 注册表的数据流向**：

```
config.py (Settings.AGENT_REGISTRY: list)
    │
    ├── agents.py (get_agents / get_agent / agent_exists)
    │       │
    │       ├── 供 messages.py 校验 agentType 时使用（未来）
    │       ├── 供 Orchestrator 获取可用 Agent 列表（未来）
    │       └── 供其他模块查询 Agent 能力元数据（未来）
    │
    └── 直接 import settings.AGENT_REGISTRY（任何模块均可）
```

相比之前各模块自建 dict（如 `messages.py` 中的 `AGENT_DISPLAY_NAMES`），现在所有 Agent 元数据集中维护在 `config.py`，后续可逐步将分散在各处的硬编码映射统一到注册表。

---

## 八、后续任务依赖

| 任务 | 可开始时机 | 关联 |
|------|----------|------|
| Orchestrator 调度逻辑集成 | 立即 | 需要 `get_agents()` 获取可用 Agent 列表来决定任务分派 |
| messages.py 请求校验统一 | 立即 | 可用 `agent_exists()` 替代当前的 try/except ValueError |
| 前端 Agent 选择器动态渲染 | agent 列表暴露后 | 需 Java 后端提供接口转发 `GET /api/agents`（如需要） |
