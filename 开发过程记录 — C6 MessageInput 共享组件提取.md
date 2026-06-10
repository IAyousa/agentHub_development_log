# 开发过程记录 — C6 MessageInput 共享组件提取

> **日期**: 2026-05-28\
> **任务编号**: C6\
> **所属模块**: frontend\
> **前置依赖**: 无（纯前端重构，零后端依赖）

---

## 一、背景

成员C 的 [11 项任务](团队任务分配.md) 中，C6 是唯一一个**纯前端重构**任务，不需要等待任何后端接口。

当前 `ChatWindow.vue` 包含 280 行代码，其中输入区域（工具栏 + textarea + 发送按钮）占据约 35 行模板 + 相关的 sendMessage 逻辑。按照设计文档，后续 C10（OfficeView 底部聊天面板）也需要**完全相同的输入区域**。如果直接复制粘贴，后续任何改动（如修改发送按钮样式、增加附件上传）都需要在两个文件中同步修改，极易遗漏。

**设计原则**：ChatView 和 OfficeView 是两套"皮肤"，内核共用同一套消息引擎。消息输入作为核心交互入口，必须作为共享组件供两边组合使用。

---

## 二、任务执行过程

### 2.1 现状分析

重构前的 ChatWindow.vue 结构（280 行）：

```
ChatWindow.vue
├── Header          — 标题栏 + 返回按钮 + 更多操作
├── Messages        — 消息列表（ChatMessage 循环渲染）
├── Loading         — "Agent 正在思考..." 动画
└── Input Area      — 将被提取到 MessageInput.vue
    ├── Toolbar     — Smile/Folder/Scissors/History 图标
    ├── Textarea    — v-model="input" + Enter 发送
    └── Send Button — 灰色/渐变 状态切换
```

**需要抽取的部分**：仅 Input Area（footer），Header 和 Messages 保留在 ChatWindow 中。

**需要移除的依赖**：`input` ref（输入框状态移入子组件）、`sendMessage` 函数（替换为 `handleSend(content)` 接收事件）、4 个图标导入（Smile/Folder/Scissors/History 移入子组件）。

### 2.2 组件接口设计

遵循 **Props Down, Events Up** 单向数据流原则：

```
父组件（ChatWindow）
  │
  │  Props（数据向下流）
  │  :disabled="isLoading"
  │  placeholder="输入消息..."
  ▼
子组件（MessageInput）
  │
  │  Emits（事件向上冒泡）
  │  @send="handleSend"
  ▼
父组件（ChatWindow）
  → handleSend(content) 被调用，执行消息发送逻辑
```

子组件**不直接访问 Pinia store**，只通过 Props/Emits 与父组件通信。这样在 OfficeView 中使用时，父组件可以自由决定发送消息的行为（群聊广播 vs 单聊发送）。

### 2.3 创建 MessageInput.vue

**文件**: `frontend/src/components/chat/MessageInput.vue`（67 行）

```typescript
// Props 定义 — Vue 3.3+ 类型化语法
const props = withDefaults(defineProps<{
  disabled?: boolean
  placeholder?: string
}>(), {
  disabled: false,
  placeholder: '输入消息...',
})

// Emits 定义 — 事件携带 string 参数
const emit = defineEmits<{
  send: [content: string]
}>()

// 输入框状态 — 由子组件独立管理
const input = ref('')

// 计算属性 — 发送按钮是否可用
const canSend = computed(() => input.value.trim().length > 0 && !props.disabled)

// 发送逻辑 — 清空由子组件负责
const handleSend = () => {
  const content = input.value.trim()
  if (!content || props.disabled) return
  emit('send', content)
  input.value = ''
}
```

**接口说明**：

| 接口 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `disabled` | Prop (boolean, 默认 false) | 父→子 | 加载中时禁用输入和发送 |
| `placeholder` | Prop (string, 默认 "输入消息...") | 父→子 | 不同场景可定制提示文字 |
| `@send` | Emit (payload: string) | 子→父 | 用户点击发送或按 Enter 时触发，携带文本内容 |

### 2.4 重构 ChatWindow.vue

**模板层**：35 行 footer 替换为 3 行 `<MessageInput>`：

```html
<!-- 改前：35 行内联 footer -->
<footer class="h-[180px] bg-white/90 ...">
  <div class="h-10 px-4 flex items-center gap-4 text-indigo-400">
    <SmileIcon ... /><FolderIcon ... /><ScissorsIcon ... /><HistoryIcon ... />
  </div>
  <div class="flex-1 px-4">
    <textarea v-model="input" ... @keydown.enter.exact.prevent="sendMessage" />
  </div>
  <div class="h-12 px-4 flex justify-end items-center">
    <button @click="sendMessage" :class="[...]">发送</button>
  </div>
</footer>

<!-- 改后：3 行组件引用 -->
<MessageInput
  :disabled="isLoading"
  placeholder="输入消息..."
  @send="handleSend"
/>
```

**脚本层**：三处改动：

| 改动 | 改前 | 改后 |
|------|------|------|
| 导入 | 6 个图标导入 | 2 个图标 + `MessageInput` 组件导入 |
| 状态 | `const input = ref('')` 由 ChatWindow 管理 | 移除，由 MessageInput 管理 |
| 发送函数 | `sendMessage()` 自己取 `input.value` 并清空 | `handleSend(content)` 接收参数 |

**对比**：

```typescript
// ── 改前 ──
const sendMessage = () => {
  if (!input.value.trim() || isLoading.value) return
  chatStore.addMessage({ ... content: input.value ... })
  const userQuery = input.value
  input.value = ''   // ChatWindow 负责清空
  simulateResponse(userQuery)
}

// ── 改后 ──
const handleSend = (content: string) => {
  chatStore.addMessage({ ... content: content ... })
  simulateResponse(content)   // 不再管理 input
}
```

---

## 三、技术要点讲解

### 3.1 为什么不用 v-model 双向绑定？

如果 MessageInput 支持 `v-model`，父组件需要维护一个 `input` ref 并传给子组件。但这里的设计理念是：**父组件只需要知道"用户发送了什么"，不需要知道"用户正在输入什么"**。

清空输入框是输入组件的自身职责，父组件收到 `@send` 事件后只管发消息。这让 MessageInput 成为一个**高内聚的自治组件**。

### 3.2 `canSend` 计算属性

```typescript
const canSend = computed(() => input.value.trim().length > 0 && !props.disabled)
```

`computed` 是 Vue 的响应式计算属性。当 `input.value`（用户打字）或 `props.disabled`（加载状态变化）任一变化时，`canSend` 自动重新计算。模板中的按钮样式和 `:disabled` 属性也随之实时更新，无需手动调用更新函数。

这个表达式替代了原 ChatWindow 模板中分散在两处的内联判断：
- 第 78 行：`input.trim() ? 'bg-gradient...' : 'bg-gray-100...'`
- 第 80 行：`:disabled="!input.trim() || isLoading"`

### 3.3 `@keydown.enter.exact.prevent` 修饰符链

```html
<textarea @keydown.enter.exact.prevent="handleSend"></textarea>
```

| 修饰符 | 作用 |
|--------|------|
| `.enter` | 只在按下 Enter 键时触发 |
| `.exact` | 只在单独按 Enter（不含 Shift/Ctrl/Alt 组合）时触发，允许 Shift+Enter 换行 |
| `.prevent` | 阻止浏览器默认行为（Enter 换行），避免 textarea 中出现多余空行 |

### 3.4 组合（Composition）优于继承

Vue 生态的核心理念之一是组合优于继承。我们不让 ChatView 和 OfficeView 继承同一个"带输入框的父类"，而是让它们各自组合共享组件：

```
ChatView                     OfficeView（C10 未来）
  ├── ChatMessage (共享)        ├── ChatMessage (共享)
  ├── MessageInput (共享)  ←──  ├── MessageInput (共享)
  └── 自己的 Header              └── 自己的 SVG 场景
```

如果未来要改发送按钮样式或增加附件上传功能，只需改 `MessageInput.vue` 一个文件。

---

## 四、验证测试

### 4.1 自动化验证

| 测试项 | 命令 | 结果 |
|--------|------|------|
| TypeScript 类型检查 | `npx vue-tsc -b --noEmit` | ✅ 通过（仅存预存的 OfficeView `onOwnerChairClick` 未使用警告，与本次无关） |
| 开发服务器 | `npm run dev` | ✅ 运行在 `http://localhost:5173/` |

### 4.2 手动功能测试清单

| # | 测试项 | 预期行为 | 结果 |
|---|--------|---------|------|
| 1 | 页面渲染 | 三栏布局 + 工具栏 + 输入框 + 发送按钮正常显示 | ✅ |
| 2 | 空输入状态 | 发送按钮灰色不可点击 | ✅ |
| 3 | 纯空格输入 | 发送按钮仍然灰色（trim 处理正确） | ✅ |
| 4 | 有内容输入 | 发送按钮变为紫蓝渐变可点击 | ✅ |
| 5 | Enter 键发送 | 消息出现 + 输入框清空 + 按钮恢复灰色 | ✅ |
| 6 | 按钮点击发送 | 同上流程 | ✅ |
| 7 | 加载中禁用 | Agent 思考时按钮灰色，按 Enter 不发送 | ✅ |
| 8 | 模拟回复 | Counter/Animation/Dashboard 关键词返回正确产物 | ✅ |
| 9 | 控制台报错 | 无红色错误，无 Vue 警告 | ✅ |

---

## 五、关键决策记录

| # | 决策 | 理由 |
|---|------|------|
| 1 | 子组件不直接访问 Pinia store | 保持组件自治，使其可在 ChatView 和 OfficeView 中复用，父组件自由决定发送行为 |
| 2 | 用 `@send` 事件而非 `v-model` | 父组件不需要知道"正在输入什么"，只需知道"发送了什么" |
| 3 | 清空输入框由子组件负责 | 输入框状态是 UI 层的职责，不应泄露到父组件 |
| 4 | `canSend` 用 computed 替代模板内联判断 | 逻辑集中在一处，避免模板中多处重复相同的判断表达式 |
| 5 | 保留 `handleSend` 中的二次校验 | 防御性编程：即使按钮 disabled，仍检查 content 和 props.disabled |
| 6 | 导入的图标跟随组件走 | Smile/Folder/Scissors/History 是输入区域独占的，不应留在 ChatWindow |

---

## 六、产物清单

| 文件 | 操作 | 行数变化 | 说明 |
|------|------|---------|------|
| `frontend/src/components/chat/MessageInput.vue` | **新建** | +67 行 | 共享输入组件，含工具栏 + textarea + 发送按钮 |
| `frontend/src/components/chat/ChatWindow.vue` | **修改** | -38/+10 行 | 用 `<MessageInput>` 替代内联 footer，用 `handleSend(content)` 替代 `sendMessage()` |

### 代码统计

| 指标 | 数值 |
|------|------|
| 新增文件 | 1 个 |
| 修改文件 | 1 个 |
| 新增代码行 | 67 行（MessageInput.vue） |
| 净减少代码行 | 28 行（ChatWindow.vue 压缩） |
| 消除的重复代码 | 0（预防性：避免未来 OfficeView 中复制 35 行） |

---

## 七、数据流全景（C6 完成后的状态）

```
用户输入 "帮我写代码"
        │
        ▼
MessageInput.vue
  ├── input.value = "帮我写代码"
  ├── 用户按 Enter 或点击发送
  ├── handleSend() → emit('send', "帮我写代码")
  │
        ▼ 事件冒泡到父组件
ChatWindow.vue
  ├── handleSend(content) 被调用
  ├── chatStore.addMessage({ ... user message ... })
  ├── simulateResponse(content)
  │     └── setTimeout → chatStore.addMessage({ ... AI reply ... })
  │
        ▼ Pinia 响应式
ChatMessage 组件自动渲染新消息
        │
        ▼
用户看到回复气泡
```

---

## 八、后续任务依赖

C6 完成后解锁以下任务：

| 任务 | 可开始时机 | 关联 |
|------|----------|------|
| C7 ChatWindow 重构 | 等 C4（WebSocket）完成后 | MessageInput 已就绪，只需替换 `handleSend` 中的 `simulateResponse` → `wsClient.send()` |
| C10 OfficeView 底部聊天面板 | **现在即可开始** | 直接引入 `<MessageInput>` 组件，零额外开发 |
