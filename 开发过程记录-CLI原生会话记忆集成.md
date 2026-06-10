# 开发过程记录 — CLI 原生会话记忆集成（阶段 1）

> **日期**: 2026-06-07  
> **关联文档**: `docs/CLI原生记忆设计方案.md`（v3.0 → v3.1）  
> **关联分支**: `dev`  
> **涉及范围**: Java 后端简化、Python 会话跟踪、Claude Code `--continue` 集成、Windows 兼容修复

---

## 一、背景

### 1.1 问题起源

项目目录下出现了一份新文档 `docs/CLI原生记忆设计方案.md`（v3.0），提出利用 Claude Code CLI 和 Codex CLI 各自的原生会话持久化能力，替代当前 Java 端 `buildContextString()` 手动拼接历史消息的做法。

用户要求评估该方案是否可以融入现有项目。

### 1.2 改造前的架构问题

```
每轮对话:
  Java WebSocketController
    → DB 查询最近 20 条历史消息
    → buildContextString() 拼成纯文本
    → HTTP POST /api/agent/chat (context = 全部历史 + 当前消息)
    → Python build_prompt() 再注入 system_prompt
    → claude -p "{system_prompt}\n\n{全部历史}\n\n{当前消息}"  ← 每次全新调用
```

问题：context 长度 O(n) 增长、system_prompt 被历史稀释、Java 维护冗余拼接代码、无法利用 CLI 原生的 `--continue` 会话恢复能力。

---

## 二、方案评估与验证

### 2.1 文档质量评估

文档在以下方面表现优秀：
- 准确识别了当前架构的 5 个核心痛点
- 对两种 CLI 的会话模型分析透彻
- 边界情况覆盖完整（服务重启、文件删除、跨会话串扰）
- 多 Agent Orchestrator 扩展设计合理

**结论**：建议融入项目，分两阶段实施（先 Claude、后 Codex）。

### 2.2 关键验证：`claude -p` 是否持久化会话

文档假设 `-p` 模式会创建会话文件、`--continue` 可恢复。为确保可靠性，在 `test-session-dir` 中进行了三组验证：

| 测试 | 命令 | 结果 |
|------|------|:--:|
| 首轮 | `claude -p "记住：我叫张三，前端工程师，喜欢蓝色"` | Claude 回复"已记住" |
| 追问 | `claude --continue -p "我叫什么名字？职业？颜色？"` | 正确答出"张三、前端工程师、蓝色" |
| 隔离 | 切换到不同 cwd 后 `--continue` | 不知道"张三"——cwd 隔离生效 |

**结论**：`--continue` 方案完全可行，且按 cwd 自动隔离会话。

### 2.3 发现文档中的事实错误

| 文档假设 | 实际情况（经官方文档 + 实机验证） |
|----------|-------------------------------|
| 会话存储在 `{cwd}/.claude/sessions/xxx.json` | 实际在 `~/.claude/projects/<project>/<session-id>.jsonl` |
| `--continue` 按 cwd 下的 `.claude/` 查找 | 按 cwd 在全局 `~/.claude/projects/` 中查找 |
| 文件系统回退检查可行 | 全局路径无法从 cwd 反向查找，此策略删除 |
| 存在 `--cwd` 参数 | 不存在，工作目录通过子进程的 `cwd` 参数指定 |

参考官方文档：
- `https://code.claude.com/docs/zh-CN/sessions`
- `https://code.claude.com/docs/zh-CN/headless`

---

## 三、实施方案

### 3.1 改动清单

| 文件 | 改动 | 效果 |
|------|------|------|
| `WebSocketController.java` | 删除 `buildContextString()` 方法 + `buildConversationContext()` 调用；context = `content.trim()` | Java 不再组装历史，只传当前消息 |
| `messages.py` | 新增 `_session_tracker: dict`；`is_first_message` 判断；`msg_end` 时标记会话 | Python 内存跟踪每个会话是否首轮 |
| `claude_adapter.py` | 首轮注入 system_prompt + `-p`；后续 `--continue -p` | CLI 按 cwd 自动管理对话记忆 |
| `main.py` | Windows `SelectorEventLoopPolicy` 修复 | 解决 `asyncio.create_subprocess_exec` 的 `NotImplementedError` |

**净增约 30 行代码**（Java 删除多于新增）。

### 3.2 核心逻辑

```
第1轮: Java context = "帮我写按钮" → Python is_first=True
       → claude -p "{system_prompt}\n\n---\n\n{msg}"
       → 会话持久化到 ~/.claude/projects/{project}/{id}.jsonl

第2轮: Java context = "改红色" → Python is_first=False
       → claude --continue -p "{msg}"
       → CLI 从 JSONL 恢复完整上下文
       → context 长度 O(1)，不再增长
```

### 3.3 边界情况处理

| 场景 | 行为 |
|------|------|
| Python 重启 | `_session_tracker` 清空 → 下次误判为首轮 → 重新注入 system_prompt（多消耗 ~1000 字），之后恢复正常 |
| 切换会话 | 不同 cwd → `--continue` 各自查找各自的会话，天然隔离 |
| 新会话 | `track_key` 不在 tracker → 正常首轮注入 system_prompt |

---

## 四、问题排查

### 4.1 Windows `NotImplementedError`

**现象**：uvicorn reload 后 `create_subprocess_exec` 抛出 `NotImplementedError`。

**原因**：Windows 默认的 `ProactorEventLoop` 不支持 subprocess；uvicorn reload 机制改变了事件循环上下文。

**修复**：在 `main.py` 顶部添加：
```python
import asyncio, sys
if sys.platform == "win32":
    asyncio.set_event_loop_policy(asyncio.WindowsSelectorEventLoopPolicy())
```

**注意**：此修改要求重启 Python 服务（`--reload` 不触发 main.py 顶层改动），且不能使用 `--reload`（reload 会再次改变事件循环策略）。

### 4.2 文件系统回退检查误判

**现象**：原实现在 cwd 下查找 `.claude/sessions/` 目录来修正 `is_first_message`，但该路径不存在。

**原因**：会话实际存储在 `~/.claude/projects/` 全局目录。

**修复**：删除文件系统回退逻辑，`_session_tracker` 作为 `is_first_message` 的唯一判断依据。

---

## 五、测试验证

三组端到端测试全部通过：

| 测试 | 操作 | 结果 |
|------|------|:--:|
| 首轮消息 | 新建会话，发送"HTML 倒计时按钮" | AI 正常生成代码 |
| 多轮追问 | 同会话，发送"改成 5 秒 + 红色" | AI 理解按钮上下文并正确修改 |
| 会话隔离 | 新建会话，发送"Python 斐波那契" | 返回纯 Python，无 HTML 混入 |

---

## 六、文档同步

`docs/CLI原生记忆设计方案.md` 从 v3.0 更新至 v3.1：
- 修正了所有存储路径引用（`{cwd}/.claude/sessions/` → `~/.claude/projects/`）
- §3.5 Claude 适配器代码对齐实际实现
- §3.6 Codex 标注为阶段 2
- §4.1 重启策略修正
- §5.1 变更清单对齐实际实施
- §6 实施步骤标注完成状态
- §7 风险表更新
- §9 删除不存在的 `--cwd` 参数

---

## 七、剩余工作

| 模块 | 状态 | 说明 |
|------|:--:|------|
| Claude Code `--continue` | ✅ 完成 | 阶段 1 已实施并验证 |
| Codex `exec resume` | ⏳ 阶段 2 | `is_first_message` 参数已预留 |
| 多 Agent Orchestrator | ⏳ P2 | §8 设计保留 |
| `_session_tracker` 持久化 | ⏳ P1 | 解决 Python 重启后误判首轮的问题 |

---

## 八、经验总结

### 8.1 设计文档 ≠ 实施手册

设计文档提供了正确的方向和架构思路，但存储路径、CLI 参数等细节需要在实施前验证。官方文档是最终权威来源。

### 8.2 从小实验开始

用两条 `claude -p` / `claude --continue -p` 命令在测试目录下 5 分钟验证了核心假设，避免了基于错误假设的大量编码。

### 8.3 Windows 特殊性

`asyncio.create_subprocess_exec` 在 Windows 上的兼容性问题需要显式处理事件循环策略。此类平台差异在跨平台项目（特别是用 Python subprocess 的场景）中容易被忽视。

### 8.4 渐进式交付

阶段 1 只做 Claude Code（项目主 Agent），Codex 留到阶段 2。这种划分让每个阶段的改动量可控，测试范围明确。
