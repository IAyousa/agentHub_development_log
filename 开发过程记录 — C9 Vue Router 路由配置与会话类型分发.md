# 开发过程记录 — C9 Vue Router 路由配置与会话类型分发

> **日期**: 2026-05-28\
> **任务编号**: C9\
> **所属模块**: frontend\
> **前置依赖**: C8 ChatList 会话列表改造 ✅

---

## 一、背景

C8 完成后，ChatList 已能正确展示用户会话列表，每个会话通过 `ConversationSummary.type` 标记为 `'direct'` 或 `'group'`。但视图切换仍然依赖 `chatStore.currentView`（一个在 store 中手动维护的状态变量），而非标准的路由机制。

**当前状态的问题**：

| 方面 | 当前（C8 完成后） | 目标（C9 完成后） |
|------|------------------|------------------|
| 视图切换方式 | `chatStore.currentView = 'chat'/'office'` + `v-if` | Vue Router 路径匹配 + `<RouterView>` |
| URL 同步 | 无（始终为 `/`） | `/chat/:conversationId` 或 `/office` |
| 会话选中 | `chatStore.selectConversation(id)` 直接调用 | 路由参数变化 → `watch` → `selectConversation` |
| 组件映射 | 硬编码在 App.vue 的 template 中 | 集中在 `router/index.ts` 一个路由文件 |

引入 Vue Router 的核心价值：

- **URL 可分享**：用户可复制 `/chat/conv_backend_001` 发给同事，直接打开该会话
- **浏览器前进/后退**：原生支持，无需手动维护历史栈
- **视图与导航解耦**：新增视图只需加一条路由记录，App.vue 零改动
- **为 C10 铺垫**：`/office/:id` 路由参数可被 OfficeView 读取，实现群聊对话

---

## 二、任务执行过程

### 2.1 现状分析

改造前的架构数据流：

```
SideBar（goChat/goOffice）
  → chatStore.currentView = 'chat' | 'office'
    → App.vue（v-if="currentView === 'chat'"）
      → 渲染 ChatList + ChatWindow + ArtifactWindow
    → App.vue（v-else）
      → 渲染 OfficeView

ChatList（handleSelect）
  → chatStore.selectConversation(id)
  → chatStore.mobileView = 'chat'
```

三个问题：
1. **视图切换逻辑散落**：`currentView` 在 store 中，`v-if` 在 App.vue 模板中，`goChat/goOffice` 在 SideBar 中——修改视图需要改三个地方
2. **URL 不反映状态**：无论用户在哪个会话，浏览器地址栏始终显示 `/`，无法分享、无法收藏
3. **会话切换无路由感知**：`handleSelect` 直接操作 store，未来 `conversation.type === 'group'` 时需要在 App.vue 中再判断跳转到哪个视图

### 2.2 安装 vue-router 依赖

```bash
npm install vue-router@4
```

- 版本选择：`vue-router@4` 对应 Vue 3，与项目现有 `vue@^3.5.34` 兼容
- 新增 2 个包，审计无新增漏洞

### 2.3 路由配置：router/index.ts

新建 `frontend/src/router/index.ts`：

```typescript
import { createRouter, createWebHistory } from 'vue-router'
import ChatView from '../views/ChatView.vue'

const router = createRouter({
  history: createWebHistory(),
  routes: [
    {
      path: '/',
      redirect: '/chat/conv_frontend_001'
    },
    {
      path: '/chat/:conversationId',
      name: 'chat',
      component: ChatView
    },
    {
      path: '/office',
      name: 'office',
      component: () => import('../views/OfficeView.vue')
    }
  ]
})
```

**三条路由的设计意图**：

| 路由 | 类型 | 作用 |
|------|------|------|
| `/` → `/chat/conv_frontend_001` | 重定向 | 用户打开应用默认进入"前端博客开发"会话，避免空白页 |
| `/chat/:conversationId` | 动态路由 | `:conversationId` 为动态参数，匹配任意会话 ID |
| `/office` | 静态路由 + 懒加载 | 办公室视图，P2 阶段使用较少，按需加载减少首屏体积 |

**关键设计决策**：

- **History 模式**（`createWebHistory`）：生成干净 URL（`/chat/conv_001` 而非 `/#/chat/conv_001`），更美观且对 SEO 友好。代价是生产环境需要后端配置 fallback 到 `index.html`（Vite 开发服务器已内置）
- **命名路由**（`name: 'chat'`）：SideBar 通过 `route.name === 'chat'` 判断激活态，比 `route.path.startsWith('/chat')` 更可靠——路径重构时命名不变
- **OfficeView 懒加载**：`() => import(...)` 使 OfficeView 的 SVG 场景代码在访问 `/office` 时才加载，减少首屏 JS 体积

### 2.4 视图容器：views/ChatView.vue

新建 `frontend/src/views/ChatView.vue`（53 行），将原来 App.vue 中的三栏布局（ChatList + ChatWindow + ArtifactWindow）抽离为独立视图组件：

```vue
<script setup lang="ts">
import { watch, onMounted } from 'vue'
import { useRoute } from 'vue-router'
import ChatList from '../components/chat/ChatList.vue'
import ChatWindow from '../components/chat/ChatWindow.vue'
import ArtifactWindow from '../components/chat/ArtifactWindow.vue'
import { useChatStore } from '../stores/chat'

const route = useRoute()
const chatStore = useChatStore()

watch(
  () => route.params.conversationId,
  (id) => {
    if (id && typeof id === 'string') {
      chatStore.selectConversation(id)
    }
  },
  { immediate: true }
)

onMounted(() => {
  chatStore.mobileView = 'list'
})
</script>
```

**核心机制：`watch` + `{ immediate: true }`**：

```
URL /chat/conv_backend_001
  → route.params.conversationId = 'conv_backend_001'
    → watch 触发（immediate: true 保证挂载时立即执行）
      → chatStore.selectConversation('conv_backend_001')
        → currentConversationId 变更
          → ChatWindow 通过 currentMessages computed 获取对应消息
```

这个 `watch` 使得以下所有场景都能正确工作：
- 用户在 ChatList 中点击会话（router.push 触发 URL 变化）
- 用户点击浏览器前进/后退按钮（URL 变化但组件未重新挂载）
- 用户在地址栏直接输入 `/chat/conv_review_001`
- 外部链接直接打开会话

**为什么选择 `watch` 而非在 `onMounted` 中读取一次？**

`onMounted` 只在组件首次挂载时执行一次。如果用户在 ChatView 内部通过 ChatList 切换会话，URL 变了（`/chat/conv_frontend_001` → `/chat/conv_backend_001`），但 ChatView 组件没有重新挂载（Vue Router 复用了同一个组件实例），`onMounted` 不会再次触发——只有 `watch` 能感知到参数变化。

### 2.5 视图容器：views/OfficeView.vue

新建 `frontend/src/views/OfficeView.vue`（11 行），作为办公室视图的路由级入口：

```vue
<script setup lang="ts">
import OfficeViewComponent from '../components/office/OfficeView.vue'
</script>

<template>
  <div class="flex-1 flex overflow-hidden">
    <OfficeViewComponent />
  </div>
</template>
```

**命名说明**：由于 `views/` 目录下的 `OfficeView.vue` 和 `components/office/OfficeView.vue` 同名，使用别名 `OfficeViewComponent` 导入以避免命名冲突。当前阶段仅为路由入口壳，C10 任务将在此添加 ChatList 侧栏和底部聊天面板。

### 2.6 App.vue 精简

改造前 App.vue 共 118 行，直接 import 了 ChatList、ChatWindow、ArtifactWindow、OfficeView 四个组件，并在模板中用 `v-if/v-else` 做视图切换。

改造后精简为 68 行：

```vue
<script setup lang="ts">
import SideBar from './components/layout/SideBar.vue'
</script>

<template>
  <div class="h-screen w-screen flex overflow-hidden font-sans select-none">
    <SideBar class="flex-shrink-0" />
    <Transition name="main-view" mode="out-in">
      <RouterView />
    </Transition>
  </div>
</template>
```

App.vue 现在只做两件事：
1. 渲染固定的 SideBar（所有视图共享的导航栏）
2. 用 `<Transition>` + `<RouterView>` 渲染当前路由对应的视图组件

**Transition 保留**：原来 Chat ↔ Office 切换时的滑入动画（`main-view`）得以保留，用户体验不打折。

### 2.7 SideBar.vue 路由化改造

改造前 SideBar 通过 `chatStore.currentView` 判断激活态，通过 `chatStore.currentView = 'chat'/'office'` 切换视图。

改造后使用 Vue Router 的组合式 API：

```typescript
import { computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'

const route = useRoute()
const router = useRouter()

const isChatActive = computed(() => route.name === 'chat')
const isOfficeActive = computed(() => route.name === 'office')

const goChat = () => {
  router.push('/chat/conv_frontend_001')
}

const goOffice = () => {
  router.push('/office')
}
```

**改造对比**：

| 方面 | 改造前 | 改造后 |
|------|--------|--------|
| 激活态判断 | `chatStore.currentView === 'chat'` | `route.name === 'chat'` |
| 导航触发 | `chatStore.currentView = 'chat'` | `router.push('/chat/conv_frontend_001')` |
| 依赖 | `useChatStore`（store） | `useRoute` + `useRouter`（router） |

`goChat` 始终跳转到默认会话 `conv_frontend_001`，确保从办公室返回聊天时用户看到的是有内容的界面，而非空路由。

### 2.8 ChatList.vue 会话类型路由分发

这是 C9 最核心的改造——点击会话时根据 `ConversationSummary.type` 分发到不同路由：

```typescript
const handleSelect = (id: string) => {
  const conv = conversationList.value.find(c => c.id === id)
  if (conv && conv.type === 'group') {
    router.push('/office/' + id)
  } else {
    router.push('/chat/' + id)
  }
}
```

**设计决策**：

| 方面 | 选择 | 理由 |
|------|------|------|
| 路由判断位置 | ChatList 组件（UI 层） | store 保持纯数据，不引入 router 依赖 |
| `type === 'group'` 路由 | `/office/:id` | 为 C10 预留，OfficeView 可通过 `route.params.id` 获取群聊 ID |
| `type === 'direct'` 路由 | `/chat/:id` | ChatView 通过 `watch(route.params.conversationId)` 同步 store |
| 移除旧逻辑 | `chatStore.selectConversation(id)` + `chatStore.mobileView = 'chat'` | 这些操作由 ChatView 的 `watch` 完成，ChatList 不再直接操作 store |

### 2.9 main.ts 注册 router

```typescript
import router from './router'

const app = createApp(App)
app.use(createPinia())
app.use(router)    // ← 新增
app.mount('#app')
```

插件注册顺序：`Pinia` → `Router`。Vue 插件按注册顺序依次执行 `install()`，此顺序保证路由守卫中可访问 Pinia store。

---

## 三、技术要点详解

### 3.1 路由驱动 vs 状态驱动

这是 C9 最核心的架构范式转换。

**改造前（状态驱动）**：

```
User Action → 修改 Store State → App.vue v-if/v-else → 切换组件
                ↑
              手动维护状态
```

**改造后（路由驱动）**：

```
User Action → router.push(url) → URL 变化 → Router 匹配 → RouterView 渲染
                                        → watch 监听 → 同步 Store
```

关键区别：**路由是单一真相来源（Single Source of Truth）**。URL 决定了页面显示什么，而不是 store 中的一个可变状态。这带来了三个好处：

1. **状态可恢复**：刷新页面后 URL 不变，Router 自动还原视图
2. **导航可追溯**：浏览器前进/后退自动生效，无需手动管理导航栈
3. **状态可分享**：URL 可复制、可收藏、可通过外部链接直达

### 3.2 `{ immediate: true }` 的必要性

```typescript
watch(
  () => route.params.conversationId,
  (id) => { chatStore.selectConversation(id) },
  { immediate: true }  // ← 关键
)
```

| 场景 | 无 immediate | 有 immediate |
|------|-------------|-------------|
| 首次访问 `/chat/conv_backend_001` | watch 不触发 → ChatWindow 白屏 | watch 立即触发 → 正确显示 |
| 从 `/chat/conv_A` 切到 `/chat/conv_B` | watch 触发 ✅ | watch 触发 ✅ |
| 刷新页面 | watch 不触发 → 白屏 | watch 立即触发 → 正确显示 |

不加 `immediate`，首次渲染时 watch 回调不会执行，ChatWindow 通过 `currentMessages` 取值时 `currentConversationId` 可能仍是默认值或空值，导致显示错误内容或白屏。

### 3.3 懒加载与代码分割

```typescript
component: () => import('../views/OfficeView.vue')
```

Vite 会将动态 import 的模块拆分为独立的 chunk 文件。当用户访问 `/office` 时，浏览器才发起网络请求加载该 chunk。OfficeView 包含完整的 SVG 场景、动画逻辑（GSAP）、椅子组件和人物组件，体积较大，懒加载可显著减少首屏加载时间。

对比：
- ChatView 使用静态 import（`import ChatView from '../views/ChatView.vue'`），因为它是最频繁访问的视图，应包含在首屏 bundle 中
- OfficeView 是 P2 功能，用户在 90% 的时间不会访问

### 3.4 会话类型路由分发的安全性

```typescript
if (conv && conv.type === 'group') {
  router.push('/office/' + id)
} else {
  router.push('/chat/' + id)
}
```

由于 `ConversationSummary.type` 是 TypeScript 联合类型 `'direct' | 'group'`，编译期即可保证：
- 不会出现拼写错误（`'gruop'`、`'DIRECT'` 等都会被 tsc 捕获）
- 新增会话类型时（如未来增加 `'channel'`），tsc 会在所有 `type` 判断处报错，强制开发者处理新分支

`else` 分支作为 fallback 而非显式 `type === 'direct'` 检查，也是一种防御性设计：未来如果有非预期值传入，至少会跳转到 ChatView 而非报错。

### 3.5 组件命名空间冲突处理

```
views/OfficeView.vue          ← 新建的路由视图入口
components/office/OfficeView.vue  ← 已有的办公室组件
```

两个文件同名但职责不同。在 `views/OfficeView.vue` 中使用别名导入：

```typescript
import OfficeViewComponent from '../components/office/OfficeView.vue'
```

这样在 `<template>` 中使用 `<OfficeViewComponent />`，语义更清晰：它表明这是一个被包装的组件，而非原始 OfficeView。

---

## 四、验证测试

共设计 9 个测试用例，全部通过：

| # | 测试项 | 操作 | 预期结果 | 结果 |
|---|--------|------|----------|------|
| 1 | 默认重定向 | 访问 `http://localhost:5175/` | URL 自动变为 `/chat/conv_frontend_001`，显示"前端博客开发" | ✅ |
| 2 | 会话列表渲染 | 观察左侧 ChatList | 3 个会话，当前选中"前端博客开发"高亮 | ✅ |
| 3 | 点击切换会话 | 点击"后端接口重构" | URL 变为 `/chat/conv_backend_001`，ChatWindow 切换内容 | ✅ |
| 4 | SideBar 激活态 | 观察左侧导航欄 | 聊天图标白色高亮 + 左侧蓝色指示条 | ✅ |
| 5 | 导航到办公室 | 点击 SideBar 建筑图标 | URL 变为 `/office`，显示办公室 SVG 场景，建筑图标高亮 | ✅ |
| 6 | 从办公室返回 | 点击 SideBar 聊天图标 | URL 变为 `/chat/conv_frontend_001`，回到聊天界面 | ✅ |
| 7 | URL 直接访问 | 地址栏输入 `/chat/conv_review_001` | ChatWindow 显示"代码审查"会话消息 | ✅ |
| 8 | 新建会话 | 点击 + → 输入名称 → 回车 | URL 变为 `/chat/conv_XXXXX`，新会话在列表顶部 | ✅ |
| 9 | 浏览器前进/后退 | 点击浏览器后退按钮 | 回到上一个会话，URL 和内容正确同步 | ✅ |

**类型检查**：`vue-tsc -b --noEmit` 通过，仅 OfficeView 预存的 `onOwnerChairClick` 未使用报错（与 C9 无关）。

---

## 五、关键决策与取舍

| 决策 | 选择 | 理由 |
|------|------|------|
| 路由模式 | History（`createWebHistory`） | 干净 URL，无 `#` 符号；生产环境需 nginx fallback 配置 |
| 默认重定向 | `/` → `/chat/conv_frontend_001` | 用户打开应用直接看到内容，不出现空白页 |
| ChatView import 方式 | 静态 import | 最频繁访问的视图，应包含在首屏 bundle |
| OfficeView import 方式 | 动态 import（懒加载） | P2 功能使用频率低，减少首屏体积 |
| 视图切换位置 | `<RouterView>` 在 App.vue | 全局一致：SideBar 固定左侧，RouterView 占据剩余空间 |
| `handleSelect` 路由逻辑 | 写在 ChatList 组件中 | Store 保持纯数据层，不引入 router 依赖 |
| `watch` vs `onMounted` | `watch(route.params, ..., { immediate: true })` | 覆盖首次渲染 + 参数变化 + 浏览器前进后退所有场景 |
| `chatStore.currentView` | 保留在 store 中不删除 | 部分旧代码可能引用；后续 C5 统一清理 |
| `/office` 路由参数 | 当前为静态路由，C10 时改为 `/office/:id` | 当前 P0 无群聊数据，预留扩展空间 |
| 激活态判断方式 | `route.name === 'chat'` | 命名路由匹配，路径格式变更不影响 |

---

## 六、产物清单

| 文件 | 变更类型 | 行数 | 说明 |
|------|----------|------|------|
| `frontend/src/router/index.ts` | **新建** | 22 行 | Vue Router 配置：3 条路由（重定向、chat、office） |
| `frontend/src/views/ChatView.vue` | **新建** | 53 行 | 聊天视图容器：ChatList + ChatWindow + ArtifactWindow 三栏布局 |
| `frontend/src/views/OfficeView.vue` | **新建** | 11 行 | 办公室视图容器：包装 OfficeView 组件 |
| `frontend/src/App.vue` | 修改 | -50 行 | 移除硬编码的组件 import 和 v-if/v-else，改用 RouterView |
| `frontend/src/components/layout/SideBar.vue` | 修改 | 替换 store 依赖为 router API | 用 `useRoute()`/`useRouter()` 替代 `chatStore.currentView` |
| `frontend/src/components/chat/ChatList.vue` | 修改 | +5 行 | `handleSelect` 按 `conversation.type` 分发路由 |
| `frontend/src/main.ts` | 修改 | +2 行 | 注册 router 插件 |
| `frontend/package.json` | 修改 | +1 行 | 新增 `vue-router@4` 依赖 |

---

## 七、架构影响

### 7.1 改造前后数据流对比

```
改造前（状态驱动）：
  ┌──────────┐    set currentView    ┌───────────────┐    v-if/v-else    ┌──────────────┐
  │ SideBar  │ ────────────────────→ │ chatStore     │ ───────────────→ │  App.vue     │
  │ ChatList │    selectConversation │  currentView  │                  │  template    │
  └──────────┘                       │  currentConvId│                  └──────────────┘

改造后（路由驱动）：
  ┌──────────┐   router.push()   ┌───────────────┐   匹配路由    ┌──────────────┐
  │ SideBar  │ ────────────────→ │ Vue Router    │ ───────────→ │  RouterView  │
  │ ChatList │                   │ /chat/:id     │              │  ChatView    │
  └──────────┘                   │ /office       │              │  OfficeView  │
                                 └───────┬───────┘              └──────┬───────┘
                                         │ route.params               │ watch
                                         └────────────────────────────┘
                                                             selectConversation
                                                                      ↓
                                                              ┌──────────────┐
                                                              │  chatStore   │
                                                              │  Pinia       │
                                                              └──────────────┘
```

### 7.2 对代码维护性的提升

| 维护场景 | 改造前 | 改造后 |
|----------|--------|--------|
| 新增视图 | 改 App.vue template + script + store 类型 | 只需在 router/index.ts 加一条路由记录 |
| 修改路由参数 | 无路由参数概念 | 改 path 中的动态参数名即可 |
| 调试会话问题 | 无法从 URL 判断当前状态 | URL 直接显示 `/chat/conv_xxx` |
| 添加路由守卫 | 无法实现 | `router.beforeEach` 标准 API |

### 7.3 ChatList 组件的职责收敛

C9 改造后，ChatList 的 `handleSelect` 从"直接操作 store + 路由分发"简化为"仅路由分发"：

```
改造前 handleSelect:
  chatStore.selectConversation(id)   ← 同步 store
  chatStore.mobileView = 'chat'      ← 控制移动端布局

改造后 handleSelect:
  router.push('/chat/' + id)         ← 仅触发路由
  // store 同步由 ChatView 的 watch 完成
  // mobileView 由 ChatView 的 onMounted 完成
```

这是一种**关注点分离**：ChatList（列表组件）不应知道自己被嵌入在什么样的布局中，也不应负责 View 级别的状态管理。它只需说"用户想打开这个会话"，剩余的交给 Router 和 ChatView 处理。

---

## 八、后续依赖

```
C9（Vue Router 配置）✅
  │
  ├──→ C5（chatStore 改造为真实 API）
  │     conversationList 和 conversations 的 mock 数据替换为 API 调用
  │     路由层零改动（URL 格式不变）
  │
  ├──→ C10（OfficeView 底部聊天面板）
  │     在 views/OfficeView.vue 中添加路由参数 /office/:id
  │     通过 route.params.id 获取群聊 ID，加载对应消息
  │
  ├──→ C11（登录/注册页面 UI）
  │     新增路由 /login 和 /register
  │     在 router/index.ts 中添加两条路由记录即可
  │
  └──→ C2/C3/C4（API 封装 + WebSocket）
        路由层无需任何改动，URL 设计已与 API 契约对齐
```

**C9 的完成意味着前端有了完整的路由基础设施。后续所有新页面只需在 `router/index.ts` 中添加路由记录，App.vue 和 SideBar 不再需要修改。**
