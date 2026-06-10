# AgentHub — Agent 项目生成与交付规划

> **版本**: v1.0  
> **创建日期**: 2026-06-09  
> **状态**: 规划中  

---

## 1. 目标概述

让 Agent（Claude Code / Codex）在独立的会话工作目录下完成项目编写，解决以下三个问题：

| # | 问题 | 目标 |
|---|------|------|
| 1 | Agent 执行时可能弹出交互式确认（如 "Do you want to proceed? [y/N]"），阻塞子进程 | 自动应答 `y`，实现无人值守执行 |
| 2 | Agent 生成的项目文件零散分布在 workspace 中 | 项目完成后自动扫描、打包、推送到前端 |
| 3 | 前端用户需要一个统一入口查看/下载 Agent 生成的项目 | 通过现有 Artifact 体系 + 新增"项目快照"事件交付给前端 |

---

## 2. 当前架构分析

### 2.1 已有基础设施

```
前端 (Vue 3)                 Java (Spring Boot)              Python (FastAPI)
──────────────              ───────────────────              ─────────────────
 ChatWindow                  WebSocketController              messages.py
    │                              │                              │
    │ WebSocket/STOMP               │ HTTP POST /api/agent/chat   │
    │  ←── token 流式推送 ──        │  ←── SSE 流式解析 ←──       │ ClaudeAdapter
    │                              │                              │ CodexAdapter
    │                              │                              │
    │  ←── preview_card ───        │ /internal/artifacts ←──      │ artifact_uploader.py
    │                              │   (保存文件 + WebSocket push) │   (解析 Markdown 代码块)
```

**已有的关键能力**：

| 能力 | 现状 | 所在位置 |
|------|------|----------|
| 会话独立工作目录 | Java 传递 `./agent_workspaces/{conversationId}` → Python | [WebSocketController.java#L82](file:///e:/AgentHub/Agenthub_demo-main/Agenthub_demo-main/backend-java/src/main/java/com/agenthub/controller/WebSocketController.java#L82) |
| 工作目录创建 | Python 端 `os.makedirs(wd, exist_ok=True)` | [messages.py#L175-L176](file:///e:/AgentHub/Agenthub_demo-main/Agenthub_demo-main/agent-service/app/api/endpoints/messages.py#L175-L176) |
| CLI 子进程 cwd 指定 | `asyncio.create_subprocess_exec(..., cwd=working_directory)` | [claude_adapter.py#L127](file:///e:/AgentHub/Agenthub_demo-main/Agenthub_demo-main/agent-service/adapters/claude_adapter.py#L127), [codex_adapter.py#L194](file:///e:/AgentHub/Agenthub_demo-main/Agenthub_demo-main/agent-service/adapters/codex_adapter.py#L194) |
| 产物上传 | Python → Java `/internal/artifacts`，含文件存储 + WebSocket 推送 | [artifact_uploader.py](file:///e:/AgentHub/Agenthub_demo-main/Agenthub_demo-main/agent-service/app/utils/artifact_uploader.py), [ArtifactController.java#L103](file:///e:/AgentHub/Agenthub_demo-main/Agenthub_demo-main/backend-java/src/main/java/com/agenthub/controller/ArtifactController.java#L103) |
| 产物预览 | Java 静态资源映射 `/artifacts/**` + 前端 iframe 预览 | [ArtifactConfig.java](file:///e:/AgentHub/Agenthub_demo-main/Agenthub_demo-main/backend-java/src/main/java/com/agenthub/config/ArtifactConfig.java) |
| 会话上下文延续 | Claude `--continue`, Codex `resume --last` | [claude_adapter.py#L108](file:///e:/AgentHub/Agenthub_demo-main/Agenthub_demo-main/agent-service/adapters/claude_adapter.py#L108), [codex_adapter.py#L180](file:///e:/AgentHub/Agenthub_demo-main/Agenthub_demo-main/agent-service/adapters/codex_adapter.py#L180) |

### 2.2 缺失的能力

| 缺失项 | 影响 |
|--------|------|
| Agent CLI 交互式确认自动应答 | 用户说"帮我写个 React 项目"后，Agent 可能在 `npm init` 处卡住等待 `y` |
| 工作目录文件变更追踪 | 不知道 Agent 生成了哪些文件（只能靠 Markdown 代码块猜测） |
| 多文件项目打包交付 | 当前仅上传 Markdown 中 ` ``` ` 代码块，无法交付完整项目（如 10+ 文件） |
| 前端"项目包"展示 | 前端只有单文件 preview_card 卡片，没有多文件工程视图 |

---

## 3. 方案设计

### 3.1 总体数据流

```
用户: "帮我用 React+TS 写一个待办事项应用"
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. Java WebSocketController                                 │
│    workspacePath = "./agent_workspaces/{conversationId}"    │
│    → POST /api/agent/chat  (含 workingDirectory)            │
└──────────────────────────┬──────────────────────────────────┘
                           │
    ┌──────────────────────▼──────────────────────────────────┐
    │ 2. Python messages.py                                    │
    │    - 确保工作目录存在                                     │
    │    - 注入 workspace snapshot（告知 Agent 当前文件结构）    │
    │    - 调用 Adapter.chat_stream()                          │
    └──────────────────────┬──────────────────────────────────┘
                           │
    ┌──────────────────────▼──────────────────────────────────┐
    │ 3. Adapter (Claude/Codex)                                │
    │    - 新增 stdin 管道: 检测提示自动输入 "y\n"             │
    │    - 新增 CLI 参数: --dangerously-skip-permissions / -y │
    │    - 子进程 cwd = workspacePath  ✅ 已有                  │
    │    - 流式输出 SSE  ✅ 已有                                │
    └──────────────────────┬──────────────────────────────────┘
                           │
    ┌──────────────────────▼──────────────────────────────────┐
    │ 4. 消息完成后 (msg_end)                                   │
    │    - workspace_scanner.py 扫描工作目录文件列表            │
    │    - 对比前一轮快照，识别增量文件                          │
    │    - 对每个文件调用 /internal/artifacts 上传              │
    │    - 推送 project_bundle 事件到 WebSocket                 │
    └──────────────────────┬──────────────────────────────────┘
                           │
    ┌──────────────────────▼──────────────────────────────────┐
    │ 5. 前端 ChatWindow                                        │
    │    - 收到 project_bundle 事件                             │
    │    - 渲染"项目成果"卡片（文件树 + 预览/下载链接）          │
    │    - 点击文件 → iframe/代码块预览                          │
    └─────────────────────────────────────────────────────────┘
```

### 3.2 问题 1：自动应答 Agent 交互式提示

#### 3.2.1 Claude Code

Claude Code CLI 支持两种无交互模式：

```bash
# 方式 1: 自动批准所有权限请求（推荐）
claude -p "prompt" --dangerously-skip-permissions

# 方式 2: 管道输入 "y"
echo "y" | claude -p "prompt"
```

**实现**: 在 `config.py` 的 `CLAUDE_CLI_ARGS` 默认值中添加该参数：

```python
# config.py
CLAUDE_CLI_ARGS: List[str] = ["--dangerously-skip-permissions"]
```

或通过 `.env` 配置：
```env
CLAUDE_CLI_ARGS=["--dangerously-skip-permissions"]
```

**风险**: `--dangerously-skip-permissions` 会跳过 Claude Code 的所有安全确认，包括文件写入、shell 命令执行。但 AgentHub 使用场景本身就是"AI 替你写代码"，在隔离的 workspace 目录下运行，风险可控。

#### 3.2.2 Codex

OpenAI Codex CLI 的交互控制：

```bash
# 方式 1: 自动批准
codex exec --approve "prompt"

# 方式 2: 环境变量跳过权限检查
export CODEX_SKIP_PERMISSIONS=true
codex exec "prompt"
```

**实现**: 在 `config.py` 中添加配置项：

```python
# config.py
CODEX_CLI_ARGS: List[str] = ["--approve"]
```

或在 subprocess 中设置环境变量：
```python
env = os.environ.copy()
env["CODEX_SKIP_PERMISSIONS"] = "true"
process = await asyncio.create_subprocess_exec(
    *cli_args,
    stdout=asyncio.subprocess.PIPE,
    stderr=asyncio.subprocess.PIPE,
    stdin=asyncio.subprocess.PIPE,  # ← 新增 stdin 管道
    cwd=working_directory,
    env=env,
)
```

#### 3.2.3 通用兜底：stdin 自动应答

对于无法通过参数控制的 CLI，采用 stdin 管道持续输入 `y\n`：

```python
# adapters/base_adapter.py 或各 adapter 中

async def _auto_approve_stdin(process):
    """持续向 subprocess stdin 写入 'y\n' 保持 Agent 不阻塞。"""
    if process.stdin:
        try:
            while process.returncode is None:
                process.stdin.write(b"y\n")
                await process.stdin.drain()
                await asyncio.sleep(2)  # 每2秒检查一次是否需要应答
        except (BrokenPipeError, ConnectionResetError):
            pass  # 进程已结束
```

**推荐策略**: 优先使用 CLI 参数（`--dangerously-skip-permissions` / `--approve`），stdin 应答作为兜底。

---

### 3.3 问题 2：工作目录项目扫描与上传

#### 3.3.1 新增模块：`workspace_scanner.py`

```python
# agent-service/app/utils/workspace_scanner.py

"""
工作区文件扫描器 — Agent 任务完成后扫描工作目录变更文件，
将所有生成的文件上传到 Java 后端的 /internal/artifacts 端点。

流程：
    messages.py (msg_end 后)
        → scan_workspace(workspace_path, ignore_patterns)
        → 对比上一轮快照（增量检测）
        → upload_artifact(每个文件) × N
        → 保存本轮快照
        → 推送 project_bundle 汇总事件
"""

import os
import hashlib
import re
from pathlib import Path
from typing import Dict, List, Optional

# CLI 工具自身的元数据目录，扫描时排除
_DEFAULT_IGNORE = [
    ".claude",          # Claude Code 会话数据
    ".codex",           # Codex 会话数据
    ".git",             # Git 仓库
    "node_modules",     # npm 依赖
    "__pycache__",      # Python 缓存
    ".venv", "venv",    # Python 虚拟环境
    "dist", "build",    # 构建产物
    ".DS_Store",        # macOS
]

# 二进制文件扩展名（不读取内容，仅记录文件名）
_BINARY_EXTENSIONS = {
    ".png", ".jpg", ".jpeg", ".gif", ".svg", ".ico",
    ".woff", ".woff2", ".ttf", ".eot",
    ".pdf", ".zip", ".tar", ".gz",
}


def snapshot_workspace(root: str, ignore_patterns: List[str] = None) -> Dict[str, str]:
    """扫描工作目录，返回文件路径 → SHA256 哈希的映射。

    Args:
        root: 工作目录绝对路径
        ignore_patterns: 额外忽略的 glob 模式

    Returns:
        {"index.html": "a1b2c3...", "src/App.tsx": "d4e5f6..."}
    """
    ignore = set(_DEFAULT_IGNORE) | set(ignore_patterns or [])
    result = {}

    if not os.path.isdir(root):
        return result

    for dirpath, dirnames, filenames in os.walk(root):
        # 过滤忽略目录
        dirnames[:] = [d for d in dirnames if d not in ignore]

        for filename in filenames:
            filepath = os.path.join(dirpath, filename)
            relpath = os.path.relpath(filepath, root).replace("\\", "/")

            try:
                with open(filepath, "rb") as f:
                    file_hash = hashlib.sha256(f.read()).hexdigest()
            except (IOError, PermissionError):
                continue

            result[relpath] = file_hash

    return result


def diff_snapshots(
    prev: Dict[str, str], current: Dict[str, str]
) -> List[str]:
    """对比两轮快照，返回本轮新增/修改的文件列表。

    Args:
        prev: 上一轮快照（首次为空字典）
        current: 本轮快照

    Returns:
        ["index.html", "src/App.tsx", ...]
    """
    changed = []
    for path, current_hash in current.items():
        if path not in prev or prev[path] != current_hash:
            changed.append(path)
    return changed
```

#### 3.3.2 快照存储

```python
# 内存中按 conversation_id 存储（与 _session_tracker 同理）
# 服务重启后丢失，首轮会全量上传（无害）
_workspace_snapshots: Dict[str, Dict[str, str]] = {}
```

#### 3.3.3 集成到 messages.py 的 msg_end 事件

```python
# messages.py 中 msg_end 处理逻辑扩展

elif chunk_type == "msg_end":
    # ... 现有逻辑 ...

    # 新增：扫描工作目录并上传文件
    if conversation_id and message_id and working_directory:
        import asyncio
        from app.utils.workspace_scanner import (
            snapshot_workspace, diff_snapshots, upload_workspace_files
        )
        # 异步上传，不阻塞 SSE 流结束
        asyncio.ensure_future(
            upload_workspace_files(
                conversation_id, message_id, working_directory,
                _workspace_snapshots
            )
        )
```

---

### 3.4 问题 3：前端项目包展示

#### 3.4.1 新增 WebSocket 事件类型：`project_bundle`

Java 后端在接收完一批文件后，通过 WebSocket 推送汇总事件：

```json
{
    "type": "project_bundle",
    "messageId": "msg_abc123",
    "conversationId": "conv_xyz",
    "files": [
        {
            "artifactId": "art_001",
            "filename": "index.html",
            "path": "index.html",
            "language": "html",
            "previewUrl": "/artifacts/art_001",
            "size": 2048
        },
        {
            "artifactId": "art_002",
            "filename": "App.tsx",
            "path": "src/App.tsx",
            "language": "typescript",
            "previewUrl": "/artifacts/art_002",
            "size": 1024
        }
    ]
}
```

#### 3.4.2 Java 端新增 `/internal/artifacts/batch` 端点

```java
// ArtifactController.java 新增

@PostMapping("/internal/artifacts/batch")
public ResponseEntity<?> internalBatchUpload(@RequestBody List<InternalArtifactRequest> reqs) {
    List<ArtifactDTO> results = new ArrayList<>();
    for (InternalArtifactRequest req : reqs) {
        ArtifactDTO dto = artifactService.saveFromContent(...);
        results.add(dto);
    }

    // 推送 project_bundle 汇总
    String topic = "/topic/conversation." + conversationId;
    Map<String, Object> bundle = new LinkedHashMap<>();
    bundle.put("type", "project_bundle");
    bundle.put("messageId", messageId);
    bundle.put("files", results.stream().map(dto -> {
        Map<String, Object> f = new LinkedHashMap<>();
        f.put("artifactId", dto.getId());
        f.put("filename", dto.getFilename());
        f.put("previewUrl", "/artifacts/" + dto.getId());
        // ...
        return f;
    }).toList());
    messagingTemplate.convertAndSend(topic, bundle);

    return ResponseEntity.status(HttpStatus.CREATED).body(results);
}
```

#### 3.4.3 前端 ChatWindow 新增项目包卡片组件

- **位置**: `frontend/src/components/chat/ProjectBundleCard.vue`
- **功能**:
  - 展示文件树（按目录分组）
  - 点击 HTML 文件 → iframe 内嵌预览
  - 点击代码文件 → 代码高亮展示
  - 提供"下载全部"链接（调用后端打包下载 API）

---

## 4. 实施步骤

### Phase 1: 自动应答（最小改动，高收益）

| 步骤 | 文件 | 改动 |
|------|------|------|
| 1.1 | `agent-service/config.py` | CLAUDE_CLI_ARGS 默认加入 `--dangerously-skip-permissions` |
| 1.2 | `agent-service/config.py` | CODEX_CLI_ARGS 默认加入 `--approve` |
| 1.3 | `agent-service/.env.example` | 添加配置说明 |
| 1.4 | 测试 | 验证 Agent 不再被交互式确认阻塞 |

### Phase 2: 工作目录扫描与上传

| 步骤 | 文件 | 改动 |
|------|------|------|
| 2.1 | **新建** `agent-service/app/utils/workspace_scanner.py` | 文件扫描、快照对比、批量上传 |
| 2.2 | `agent-service/app/api/endpoints/messages.py` | msg_end 后调用扫描上传 |
| 2.3 | `agent-service/config.py` | 添加 `AGENT_WORKSPACE_IGNORE` 配置 |
| 2.4 | 测试 | 验证 Agent 生成文件后自动上传 |

### Phase 3: 批量上传端点

| 步骤 | 文件 | 改动 |
|------|------|------|
| 3.1 | `backend-java/.../controller/ArtifactController.java` | 新增 `POST /internal/artifacts/batch` |
| 3.2 | `backend-java/.../controller/ArtifactController.java` | 批量上传后推送 `project_bundle` WebSocket 事件 |
| 3.3 | `agent-service/app/utils/workspace_scanner.py` | 调用批量端点替代逐个上传 |

### Phase 4: 前端展示

| 步骤 | 文件 | 改动 |
|------|------|------|
| 4.1 | **新建** `frontend/src/components/chat/ProjectBundleCard.vue` | 项目包卡片组件 |
| 4.2 | `frontend/src/stores/chat.ts` | 处理 `project_bundle` WebSocket 事件 |
| 4.3 | `frontend/src/components/chat/ChatWindow.vue` | 渲染 ProjectBundleCard |
| 4.4 | `frontend/src/api/conversation.ts` | 添加批量下载 API |

---

## 5. 关键技术决策

### 5.1 为什么不在 Agent 执行后直接打包 zip？

- Agent 可能在多轮对话中持续修改文件，逐轮增量更合理
- 前端 iframe 预览 HTML 需要静态资源 URL，打包 zip 后无法直接预览
- 现有 Artifact 体系已支持单文件访问，扩展为多文件即可

### 5.2 为什么用 SHA256 哈希而非 mtime？

- Agent 可能在子进程中修改文件但保留相同时间戳
- 哈希对比确保精确检测内容变更
- 大文件（>1MB）跳过哈希，直接按 mtime 判断

### 5.3 为什么 workspace 快照存内存而非磁盘？

- 对齐 `_session_tracker` 的设计（快照丢失最多导致首轮全量上传，不影响正确性）
- 避免引入文件系统状态管理复杂度
- 后续可迁至 Redis（与 Agent 缓存共址）

### 5.4 为什么 `--dangerously-skip-permissions` 是安全的？

- Agent 运行在隔离的 `./agent_workspaces/{conversationId}` 目录下
- 该目录仅包含该会话的项目文件，不影响系统
- 用户主动发起的"写项目"请求本身就是信任行为
- 可通过 `.env` 配置选择是否启用

---

## 6. 配置示例

### 6.1 `.env` 文件

```env
# Agent CLI 参数 — 自动批准模式
CLAUDE_CLI_ARGS=["--dangerously-skip-permissions"]
CODEX_CLI_ARGS=["--approve"]

# Agent 工作目录
AGENT_WORKSPACE_ROOT=./agent_workspaces

# 工作空间扫描忽略目录（追加）
AGENT_WORKSPACE_IGNORE=["dist", "build", ".turbo"]
```

### 6.2 `application.yml`（Java 端，如需要）

```yaml
# 产物存储路径（已有）
artifact:
  storage:
    path: ./artifacts

# 前端访问的基础 URL（已有）
app:
  url: http://localhost:8080
```

---

## 7. 风险与注意事项

| 风险 | 缓解措施 |
|------|----------|
| `--dangerously-skip-permissions` 允许 Agent 执行任意 shell 命令 | workspace 目录隔离 + Docker 部署时限制文件系统访问 |
| 大项目（100+ 文件）上传耗时过长 | 限制单次上传文件数上限（如 50），分轮上传；仅上传代码/文本文件，排除 binary 和 node_modules |
| Codex `--approve` 在不同版本间行为不一致 | 同时保留 stdin 管道应答兜底 |
| 多轮对话中用户删除文件后快照不更新 | 快照对比同时检测删除（`prev` 有但 `current` 无），推送 `files_removed` 事件 |
| 不同操作系统文件路径分隔符差异 | 统一使用 `/` 作为路径分隔符 |
