# 开发过程记录 — 前端 UI 增强：Markdown 渲染与交互优化

> **日期**: 2026-06-06\
> **关联分支**: `dev`\
> **关联 commit**: `b55687e`\
> **涉及范围**: ChatMessage Markdown 渲染、代码块复制、消息操作栏、会话删除、预览卡片持久化、系统提示词修正

---

## 一、背景

在使用 Agent 进行日常对话时，用户发现了三个体验问题：

1. **消息无法复制**：聊天中的文本和代码无法选中或复制
2. **会话无法删除**：左侧列表无法删除不需要的会话
3. **代码块未渲染**：Agent 返回的 Markdown 代码块以原始格式显示，无语法高亮

这三个问题虽小，但直接影响日常使用体验。

---

## 二、Markdown 渲染

### 2.1 方案选择

ChatMessage 的消息类型有 `text`、`code`、`artifact_preview` 三种。当前 `text` 类型使用 `{{ content }}` 纯文本插值渲染。Agent 回复中包含 Markdown 格式（标题、列表、代码块、表格等），需要解析后渲染。

项目已有的 Monaco Editor 适合全屏代码查看，但不适合内联渲染。安装 `marked` 库做 Markdown 解析：

```bash
npm install marked
```

### 2.2 实现

`ChatMessage.vue` 将文本内容通过 `marked.parse()` 转为 HTML，使用 `v-html` 渲染。同时编写完整的 Markdown CSS 样式表，覆盖标题、代码块、列表、表格、引用等常用元素。

### 2.3 视觉问题：气泡底部多余空白

用户反馈："消息气泡框下面会多出一行空白出来"。

**根因**：`whitespace-pre-wrap` 保留了 Markdown 原始文本中的换行符，同时 Markdown 渲染后 `<p>` 标签的 `margin-bottom` 在最后一行产生额外间距。

**修复**：
- 移除 `whitespace-pre-wrap`（Markdown 渲染后由 HTML 元素管理格式）
- 添加 `.markdown-body > *:first-child { margin-top: 0 }` 和 `.markdown-body > *:last-child { margin-bottom: 0 }`

---

## 三、代码块复制功能

### 3.1 第一版：简单复制按钮

初始实现在代码块上方添加浅灰顶栏 + 复制按钮。用户查看后认为：**"可以做成像 DeepSeek 网页版一样的形式"**。

### 3.2 对照 DeepSeek 风格重构

用户提供了 DeepSeek 网页版的实际 HTML 源码。分析其结构特点：

- 深色一体设计，顶栏含语言标签 + 胶囊风格按钮
- 按钮 hover 时有微妙高亮
- 无边框的极简风格

参照实现：

```html
<div class="md-code-block md-code-block-dark">
  <div class="md-code-block-banner">
    <span class="md-code-lang">html</span>
    <button class="md-code-btn">复制</button>
  </div>
  <pre><code>...</code></pre>
</div>
```

使用 `marked.Renderer` 的自定义 `code` 方法注入顶栏，全局 `click` 事件代理处理复制逻辑。

### 3.3 配色调整

用户尝试浅灰配色后认为："底色有点太深了，可以设置为浅灰色"。改为浅灰后又说："还是先改回来吧"。最终保持深色主题 `#1e1e1e` 底色 + `#252525` 顶栏，与 DeepSeek/ChatGPT 风格一致的暗色代码区。

---

## 四、消息操作栏

### 4.1 第一版：文字链接式复制按钮

初始方案是在消息下方添加一个文字链接："复制"。

### 4.2 用户反馈：扩展性不足

用户指出："消息下方的复制按钮能够放在一个容器栏里，这个容器栏在鼠标悬浮时浮现，方便后期拓展新的按钮"。

### 4.3 最终设计

改为独立容器栏：

```html
<div class="opacity-0 group-hover:opacity-100 flex items-center gap-1">
  <button>复制</button>
  <!-- 以后可扩展：编辑、重试、点赞等 -->
</div>
```

- `opacity-0 group-hover:opacity-100`：hover 时浮现，150ms 过渡动画
- 白色背景 + 边框的胶囊按钮风格
- 初始仅含复制按钮，容器为后续扩展预留

### 4.4 扩展到用户消息

用户提出："现在我想让用户发送的信息也可以被复制"。只需移除外层 `v-if="role !== 'user'"` 条件即可。

---

## 五、会话删除

### 5.1 分析现有能力

- API 层：`deleteConversation(id)` 已存在
- Store 层：缺少 `deleteConversation` 方法
- UI 层：ChatList 无删除按钮

### 5.2 实现

**Store**：新增 `deleteConversation(id)`：
- 调 `DELETE /conversations/{id}`
- 从 `conversationList` 过滤移除
- 从 `conversations` map 删除
- 如果删除的是当前会话，自动切换到剩余第一个

**ChatList**：每个会话项添加 hover 浮现的红色 X 按钮：
- `@click.stop` 阻止冒泡（不触发 `handleSelect` 导航）
- `group relative` + `opacity-0 group-hover:opacity-100`

---

## 六、预览卡片持久化

### 6.1 问题发现

用户测试时刷新页面后反馈："生成的预览卡片都消失了"。

**根因**：预览卡片通过 WebSocket 实时推送，存在 Pinia store 内存中，但从未持久化到数据库。刷新后 `loadMessages` 只能查到 `text` 类型的消息。

### 6.2 修复

`loadMessages` 中同步加载产物列表：

```ts
const [msgRes, artRes] = await Promise.all([
  getConversationMessages(conversationId, 0, 50),
  getConversationArtifacts(conversationId).catch(() => ({ data: { artifacts: [] } })),
])
```

将 `Artifact` 实体转换为 `artifact_preview` 类型消息，与普通消息合并后按 `created_at` 排序。同时 `handleArtifactClick` 增加 fetch 逻辑：从 API 加载的产物内容只是文件名，点击时需 `fetch(/artifacts/{id})` 获取实际代码。

---

## 七、系统提示词修正：纯文本模式

### 7.1 问题

用户与 Agent 交互时发现 Agent 反复要求"写入权限"，但实际运行在 `claude -p` 只读模式下，无法写文件。

### 7.2 修复

更新三个 Agent 的系统提示词（`claude_code`、`codex`、`custom`）：

- 明确声明"运行在纯文本模式下"
- "你无法写入文件、执行命令或修改系统"
- "不要要求用户给予文件写入权限"
- 代码必须用 Markdown 代码块输出，文件名写在注释中

---

## 八、最终产物

### 8.1 代码变更

| 文件 | 改动 |
|------|------|
| `ChatMessage.vue` | Markdown 渲染 + 代码块复制栏 + 消息操作栏 + 产物获取 |
| `ChatList.vue` | 会话删除按钮（hover 浮现 X） |
| `chat.ts` | `deleteConversation` 方法 + `loadMessages` 合并产物 + `selectedAgentId` |
| `conversation.ts` | 新增 `getConversationArtifacts` API |
| `package.json` | 新增 `marked` 依赖 |
| `system_prompts.py` | 3 个 Agent 角色明确纯文本模式 |

### 8.2 提交记录

```
b55687e feat(frontend): 前端UI增强—Markdown渲染+代码块复制+会话删除+消息操作栏
```

---

## 九、经验总结

### 9.1 视觉迭代需要快速反馈循环

代码块配色经历了"深色 → 浅灰 → 深色"的往返。用户通过实际浏览效果判断，而非预先设计。前端 UI 开发中，快速看到效果比事先完美设计更重要。

### 9.2 扩展性应该预先考虑

消息复制按钮最初是一个独立的文字链接，用户指出应该放入容器栏方便扩展。改造成本很小（包裹一层 div），但如果在第一版就考虑容器化，就可以避免换行调整。

### 9.3 WebSocket 消息≠持久化数据

预览卡片通过 STOMP 实时推送，看似正常工作。但刷新后消失暴露了数据未持久化的问题。实时推送的数据也应有对应的持久化存储路径，否则用户体验在刷新/重连后会倒退。

### 9.4 系统提示词要匹配运行模式

Agent 要求写入权限是因为提示词说"可以直接读写文件"。但实际上 `claude -p` 模式无法写文件。系统提示词必须准确描述 Agent 的实际能力边界，否则会产生用户困惑。
