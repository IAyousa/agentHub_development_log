# 开发过程记录 — MVP 测试与 Bug 修复

> **日期**: 2026-06-07  
> **关联分支**: `dev`  
> **关联提交**: `5f99f60`（6 项 Bug 修复）、`2d2324c`（XSS 防护）  

---

## 一、背景

AgentHub MVP（P0）已完整可演示，涉及前端 Vue 3、Java Spring Boot、Python FastAPI 三层架构。本次测试聚焦**高风险区域**——可能导致数据丢失、服务崩溃、用户体验严重受损的问题。

## 二、测试计划

设计 7 大类 18 项测试用例，按风险等级排序：

| 等级 | 测试项 | 最坏后果 |
|:--:|------|------|
| 🔴 | Agent 不可用降级 | 前端崩溃 |
| 🔴 | 删除当前会话 | 前端僵尸状态 |
| 🔴 | 路径遍历攻击 | 文件系统安全 |
| 🟡 | WebSocket 断连恢复 | 用户无感知 |
| 🟡 | Python 重启后记忆 | 上下文丢失 |
| 🟡 | 会话隔离 | 跨会话串扰 |
| 🟢 | XSS、空消息、特殊字符 | 用户体验 |

## 三、发现的 Bug 与修复

### Bug 1: Agent 不可用时前端无限卡死 🔴

**现象**: 关闭 Python 服务后发送消息，前端永远停留在"Agent 正在思考..."加载状态，无任何错误提示。

**根因**: `AgentGatewayService.doOnError()` 只打印日志，没有通过 `onToken` 回调推送错误消息给前端。

**修复** (`AgentGatewayService.java`):
```java
.doOnError(e -> {
    log.error("Agent SSE streaming error", e);
    String friendlyMsg = e instanceof WebClientRequestException
            ? "Agent 服务暂时不可用，请检查 Python 服务是否已启动（端口 8000）"
            : "Agent 服务异常，请稍后重试";
    onToken.accept(AgentToken.error(friendlyMsg));
})
```

**额外修复** (`ChatWindow.vue`): 添加 `watch(error)` 监听 store 的 error 状态，自动弹出 toast 通知。

---

### Bug 2: 删除最后会话后右侧窗口僵尸状态 🔴

**现象**: 删除最后一个会话后，最右侧聊天窗口停留在已删除会话的界面（标题显示"chat"），无法发送消息。

**根因**: 
1. `currentConversationId` 被设为空字符串但路由未跳转
2. 初始值硬编码 `'conv_frontend_001'`
3. `conversationList` 和 `conversations` 包含 mock 数据

**修复** (4 处):
- `chat.ts`: `currentConversationId` 初始值改为 `''`
- `chat.ts`: 清空 `conversationList` 和 `conversations` 的 mock 数据
- `chat.ts`: `selectConversation` 增加会话存在性校验
- `router/index.ts`: `/chat/:conversationId` 改为可选参数，根路径重定向到 `/chat`
- `ChatView.vue`: 取消 watch `immediate:true`，改为 `onMounted` 先 `await loadConversationList()` 再选择
- `ChatWindow.vue`: `currentConversationId` 为空时显示引导界面
- `SideBar.vue`: 导航改为 `/chat`

---

### Bug 3: 会话在 AI 回复中被删除导致 DB 异常 🟡

**现象**: 用户在 AI 正在回复时删除会话，Java 日志出现 `IllegalArgumentException: 会话不存在`。

**根因**: AI 回复完成后 `messageService.saveMessage()` 尝试保存到已删除的会话，外键约束失败。

**修复** (`WebSocketController.java`): saveMessage 调用加 try-catch，捕获 `IllegalArgumentException` 后静默 info 日志。

---

### Bug 4: WebSocket 断连时前端静默 🔴

**现象**: 关闭 Java 服务后发送消息，用户自己的消息添加到界面上，但没有任何错误反馈——"发出去就消失了"。

**根因**: 
1. SockJS 心跳检测有延迟（5-10 秒），期间 `ws.connected` 仍为 `true`，消息被 STOMP publish 到虚空
2. `sendMessage` 中 `error.value` 在 `ws.connected === true` 路径下不会触发
3. `ChatWindow` 的 `watch(error)` 被错误嵌入 `showToast` 函数体内部，语法错误导致编译失败

**修复** (2 处):
- `chat.ts`: 新增 30 秒响应超时定时器 `sendTimeout`，在 `handleToken` 首 token 到达和 `handleWsError` 时清除
- `ChatWindow.vue`: `watch(error)` 移到函数外部正确位置

---

### Bug 5: `<script>` 标签内容在用户消息中不可见 🟢

**现象**: 用户发送 `<script>alert('xss')</script>`，标签内容在聊天气泡中完全不可见。

**根因**: `marked` 将 HTML 标签作为原生 HTML 透传，`v-html` 渲染为不可见的 DOM script 元素。

**修复** (`ChatMessage.vue`): 在 markdown 解析前用正则 `/<(\/?[a-zA-Z][a-zA-Z0-9]*)/g` 转义 HTML 标签为 `&lt;`，同时保留 markdown 自动链接语法 `<url>`。

---

### Bug 6: 内联代码背景色过亮 🟢

**现象**: 用户消息中的 `` `代码` ``（反引号内联代码）背景色 `#f1f5f9` 接近白色，文字几乎不可见。

**修复** (`ChatMessage.vue`): 背景改为 `#e2e8f0`，文字色明确设为 `#1e293b`。

---

### Bug 7: Swagger UI 缺失

**现象**: Java 后端无 Swagger UI，API 测试不便。

**修复** (`pom.xml`): 添加 `springdoc-openapi-starter-webmvc-ui:2.5.0` 依赖。

---

## 四、验证通过的功能

| 测试 | 功能 | 结果 |
|------|------|:--:|
| 4.3 | 路径遍历攻击防护（`../../../etc/passwd`） | ✅ 400 拒绝 |
| 3.2 | `--continue` cwd 会话隔离 | ✅ 不串扰 |
| 2.3 | 刷新浏览器状态恢复 | ✅ 消息+预览卡片完整 |
| 7.1 | 空消息拦截 | ✅ 前端不发 |
| 5.2 | WebSocket 断连 toast | ✅ 30s 超时提示 |

## 五、已知限制（接受不改）

| 限制 | 影响 | 计划 |
|------|------|------|
| Python 重启后 `_session_tracker` 丢失 | 每个活跃会话浪费 1 次 system_prompt 传输 | P1 持久化 |
| Codex CLI 未配置 API Key | 切换到 Codex 发送消息会超时 | 阶段 2 |
| SockJS 断连检测延迟 | 发送消息后可能等 30s 才有超时提示 | 可接受 |

## 六、代码变更统计

本次测试共改动 8 个文件（不计 CSS 微调），涉及三层架构：

| 层 | 文件 | 修复的 Bug |
|------|------|-----|
| Java | `AgentGatewayService.java` | Bug 1 |
| Java | `WebSocketController.java` | Bug 3 |
| Java | `pom.xml` | Bug 7 |
| Python | `claude_adapter.py` | 调试日志（已移除） |
| 前端 | `chat.ts` | Bug 2, 4 |
| 前端 | `ChatWindow.vue` | Bug 1, 2, 4 |
| 前端 | `ChatView.vue` | Bug 2 |
| 前端 | `ChatMessage.vue` | Bug 5, 6 |
| 前端 | `SideBar.vue` | Bug 2 |
| 前端 | `router/index.ts` | Bug 2 |

## 七、经验总结

### 7.1 断连检测要有多层兜底

SockJS 的断连检测不是即时的（依赖心跳超时），单靠 `ws.connected` 无法覆盖所有场景。本次添加了应用层的 30 秒超时定时器作为兜底，在收到首 token 时清除。这种"乐观发送 + 悲观超时"的组合模式值得在其他实时通信场景复用。

### 7.2 前端状态初始化不能依赖 mock 数据

`conversationList` 和 `currentConversationId` 的硬编码初始值导致了大量边界问题（刷新后选中不存在的会话、删除后僵尸状态）。正确的做法是初始化为空集合/空值，等 API 返回真实数据后再填充。

### 7.3 错误推送链路必须端到端覆盖

`AgentGatewayService.doOnError` → `WebSocketController.onToken` → `ChatWindow.watch(error)` → toast。这个链路中任何一个环节断裂（本次断在第一步），用户就完全不知道出错了。测试时必须覆盖全链路。
