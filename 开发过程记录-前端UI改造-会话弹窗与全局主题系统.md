# 开发过程记录 — 前端 UI 改造：会话弹窗 + 全局主题系统

> **日期**: 2026-06-09  
> **关联提交**: `5836aa3`（改动1-3）、`25d0537`（主题系统）、`c724191`（PR/12合入）、`5dd3d8d`（主题重构）  
> **关联 PR**: PR/12（会话 created_by 绑定登录用户）  
> **涉及范围**: Vue 3 前端全面 UI 改造 + 主题色系统 + Redis 降级修复

---

## 一、背景

前端在完成登录/注册功能后，用户提出4项 UI 改造需求。实施过程中因产物预览卡片逻辑反复推倒重来，最终只保留前3项改动。后续扩展为全局主题色切换系统，并在过程中修复了 Redis 不可用导致登录失败的降级问题。

---

## 二、改动1：新建会话弹窗

### 2.1 现状

`ChatList.vue` 点"+"按钮后在搜索栏位置弹出内联输入框，只能输入会话名称，无法选择会话类型和参与 Agent。

### 2.2 实现

**新建 `CreateConversationModal.vue`**（参照 `CreateOfficeModal.vue` 的 Teleport modal 模式）：

- 会话名称输入框（必填，≤50字符）
- 会话类型切换：一对一（direct）/ 群聊（group）
- direct 模式：下拉单选 Agent
- group 模式：checkbox 多选 Agent
- 校验：名称必填 + Agent 必选

**修改 `ChatList.vue`**：删除内联输入逻辑（`showCreateInput`/`createTitle`），"+"按钮打开 modal，`@create` 事件调用 `createConversation(title, type, agentIds)`。

**修改 `stores/chat.ts`**：`createConversation()` 签名扩展为 `(title, type, agentIds)`，API 调用传入正确的 type 和 agentIds。

### 2.3 关键技术点

后端 `POST /conversations` 已支持 `type: 'direct'|'group'` + `agentIds: string[]`，无需改动。创建成功后 `selectedAgentId` 设为第一个 Agent ID。

---

## 三、改动2：移除 Agent 切换下拉框

### 3.1 现状

`ChatWindow.vue` header 右侧有 `<select>` 下拉框和 `MoreHorizontalIcon` 占位图标。

### 3.2 实现

删除模板中的 `<select>` 元素和 `<MoreHorizontalIcon>`，清理对应的 import。Agent 选择改为在创建会话时完成（改动1）。

`selectedAgentId` 保留在 store 中（默认 `agent_claude_001`），`sendMessage()` 继续使用它。

---

## 四、改动3：办公室 mock 数据清理

### 4.1 现状

3处 mock 数据：
1. `offices` 初始值：默认办公室 + mock 成员（张小明/李小红/王大伟）+ mock availableUsers（陈静静/刘先生/产品经理）
2. `createOffice()`：硬编码 availableUsers（陈静静/刘先生/产品经理/数据分析师）
3. `syncOfficesFromConversations()`：同步时添加 mock members + availableUsers

### 4.2 实现

- 新增 `officeAvailableAgents` computed：将 `agents` 列表转为 `OfficeMember[]`
- 默认办公室初始值改为空数组 `[]`，`currentOfficeId` 初始为空
- `inviteMember` 支持从 `officeAvailableAgents` 查找
- `OfficeView.vue` 的 `allGuests` 改为过滤已在办公室的 Agent
- 删除3处 mock 数据

---

## 五、Redis 不可用导致登录失败

### 5.1 问题

用户输入正确账号密码，进入 chat 页面瞬间被踢回登录页。控制台 401 错误。

### 5.2 根因

Redis 未启动 → `JwtAuthenticationFilter` 中 `stringRedisTemplate.hasKey()` 抛出 `RedisConnectionFailureException` → 请求返回 500 → 前端 401 拦截器清 token 跳登录。

### 5.3 修复

- `JwtAuthenticationFilter`：黑名单检查包裹 try-catch，Redis 不可用时降级放过
- `AuthController.logout()`：Redis 写入包裹 try-catch，不可用时跳过黑名单

```java
// JwtAuthenticationFilter
try {
    Boolean isBlacklisted = stringRedisTemplate.hasKey(blacklistKey);
    if (Boolean.TRUE.equals(isBlacklisted)) { ... return; }
} catch (Exception e) {
    logger.warn("Redis 不可用，跳过 Token 黑名单检查: {}", e.getMessage());
}
```

---

## 六、全局主题色切换系统

### 6.1 第一版：组件级 CSS 覆盖（已废弃）

在每个组件 `<style scoped>` 中用 `!important` 覆盖 Tailwind 生成的 indigo/purple 类名。

**问题**：
1. 每个组件都要手动添加覆盖规则，容易遗漏
2. CSS 变量初始化时序不可控
3. Tailwind 类名转义复杂（如 `.after\:border-l-indigo-500::after`）
4. 滚动条 `::-webkit-scrollbar` 伪元素不支持 CSS 变量动态更新

### 6.2 第二版：Tailwind @theme 原生方案（当前方案）

**核心思路**：不在每个组件里覆盖，而是在 `style.css` 用 Tailwind v4 的 `@theme` 指令把 indigo/purple 全色阶映射到 CSS 变量。

```css
@theme {
  --color-indigo-500: var(--accent-start);
  --color-indigo-600: color-mix(in srgb, var(--accent-start) 85%, #1e293b);
  --color-purple-500: var(--accent-end);
  /* ... 50/100/200/300/400/500/600/700/800/900 全色阶 */
}
```

**效果**：所有 `text-indigo-*`、`bg-indigo-*`、`border-indigo-*`、`from-indigo-*` 等 Tailwind 工具类自动使用主题色，无需组件级覆盖。

**删除 6 个组件中 40+ 条 `!important` 规则**，LoginView 任意颜色值（`[#6366f1]`）替换为标准类（`indigo-500`）。

### 6.3 CSS 变量注入时机

`settings.ts` 模块加载时（`import` 阶段）立即调用 `applyCssVars()`，早于 Vue 组件渲染，确保初始 CSS 变量就绪。

### 6.4 滚动条特殊处理

`::-webkit-scrollbar` 伪元素在 WebKit 特殊渲染层，不支持 CSS 变量动态更新。改为 JS 动态注入 `<style>` 标签：

```ts
// settings.ts — applyCssVars() 中
let styleEl = document.getElementById('scrollbar-theme')
if (!styleEl) {
  styleEl = document.createElement('style')
  styleEl.id = 'scrollbar-theme'
  document.head.appendChild(styleEl)
}
styleEl.textContent = `
  ::-webkit-scrollbar-thumb { background: color-mix(in srgb, ${accent} 30%, #cbd5e1); }
  ::-webkit-scrollbar-thumb:hover { background: color-mix(in srgb, ${accent} 50%, #94a3b8); }
`
```

每次主题切换时重写 `<style>` 内容，用具体颜色值（如 `#6366f1`）替代 CSS 变量。

### 6.5 主题覆盖范围

| 区域 | 覆盖方式 |
|------|---------|
| SideBar 背景/激活条/图标 | 内联 `:style` + CSS 变量 |
| ChatList 背景/选中项/按钮/标签 | `@theme` + 部分内联 style |
| ChatWindow 背景/header/加载动画 | `@theme` |
| ChatMessage 气泡/预览卡片/markdown | `@theme`（气泡三角除外，保留 CSS 规则） |
| MessageInput 工具栏/发送按钮/边框 | `@theme` |
| 所有弹窗 header/按钮/选中态 | `@theme` |
| 办公室标题/按钮/弹窗 | `@theme` |
| 登录页 Logo/按钮/输入框 | `@theme` + 标准类替换任意值 |
| 滚动条滑块 | JS 动态注入 `<style>` |

### 6.6 4 套预设主题

与 CreateOfficeModal 原本的 4 套办公室主题（后被移除）一致：

| ID | 名称 | SideBar 色 | 强调色 |
|----|------|-----------|--------|
| modern | 现代靛紫 | `#1e1b4b`→`#312e81` | `#6366f1`→`#8b5cf6` |
| ocean | 深海蓝 | `#0c1929`→`#1a365d` | `#3b82f6`→`#06b6d4` |
| forest | 翠竹绿 | `#052e16`→`#14532d` | `#10b981`→`#34d399` |
| sunset | 日落橙 | `#431407`→`#7c2d12` | `#f97316`→`#ef4444` |

---

## 七、PR/12 审查合并

### 7.1 改动内容

| 文件 | 改动 |
|------|------|
| `ConversationController.java` | `"system"` → `getCurrentUserId()` 从 SecurityContext 获取真实用户ID |
| `chat.ts` | 新增 `resetState()` 清空会话状态 |
| `SideBar.vue` | 登出时调用 `resetState()` + `destroyWsClient()` |
| `LoginView.vue` | 重新登录时刷新聊天区 |

### 7.2 合并方式

cherry-pick PR 的唯一切实提交 `359a446`，手动解决 SideBar.vue 的 import 冲突（两边保留）。

---

## 八、调试过程

### 8.1 产物预览卡片反复修改

改动4（非 HTML 产物 Markdown 预览）经历多次尝试：
- 第一轮：修改 ChatMessage.vue 预览卡片 + ArtifactWindow.vue → 用户不满意样式
- 第二轮回退后重新实施改动1-3 → 发现 `/internal/artifacts` 401 问题
- 第三轮尝试让非 web 语言不显示预览卡片 → 逻辑不对
- 最终回退，改动4暂不实施

### 8.2 主题系统第一版的 CSS 编译错误

在 ChatMessage.vue 的 `:style` 中使用模板字面量 `${}` 导致 Vue 模板编译器报错。改为纯 CSS 类 + `<style>` 规则解决。

`color-mix()` 在 `:style` 绑定中的逗号被 Vue 解析器误识别为 JavaScript 表达式分隔符。改为 CSS 类解决。

### 8.3 滚动条不跟随主题

经历3次尝试：
1. `style.css` 中用 `var(--accent-start)` → 不生效（伪元素层不支持动态 CSS 变量）
2. 模块级 `applyCssVars()` 提前注入 → 仍不生效（时序不是根因）
3. 最终方案：JS 动态注入 `<style>` 标签，用具体颜色值 → 成功

---

## 九、经验总结

### 9.1 Tailwind @theme 优于组件级覆盖

在 `style.css` 中一次性将 indigo/purple 色阶映射到 CSS 变量，让所有标准 Tailwind 类自动跟随主题，远比在每个组件里手写 `!important` 覆盖规则优雅且可靠。

### 9.2 WebKit 伪元素的 CSS 变量限制

`::-webkit-scrollbar` 等伪元素在特殊渲染层，CSS 自定义属性变化不会触发重绘。需要 JS 直接操作样式表。

### 9.3 CSS `color-mix()` 在 Vue 模板中的陷阱

`color-mix(in srgb, var(--accent-start) 30%, #cbd5e1)` 中的逗号在 Vue `:style` 绑定中被当作 JavaScript 表达式分隔符，导致编译错误。应放在 `<style>` CSS 规则中。

### 9.4 Redis 降级是 JWT 系统的必要保障

JWT 黑名单（登出功能）依赖 Redis，但 Redis 不可用时不应阻断正常认证流程。try-catch 降级放过是低成本高收益的容错措施。

### 9.5 前后端联调需同步重启

SecurityConfig 改动需要重启 Java 后端；前端改动依赖 Vite HMR 热更新（部分情况下需手动刷新）。调试时应确认两端均已加载最新代码。
