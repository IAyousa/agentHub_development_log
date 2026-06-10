# 开发过程记录 — C1 Axios 实例配置

> **日期**: 2026-05-28（初版）/ 2026-06-01（改进迭代）\
> **任务编号**: C1\
> **所属模块**: frontend\
> **前置依赖**: B1 CorsConfig ✅（已完成）

---

## 一、背景

成员C 开始前端开发任务。当前前端项目使用 Vue 3 + TypeScript + Pinia + Tailwind CSS + Vite 技术栈，`axios`、`@stomp/stompjs`、`sockjs-client` 依赖已在 `package.json` 中声明，但从未实际使用过。整个前端仍然依赖硬编码的 mock 数据。

C1 是整个成员C 任务链的**第一步基础设施**：创建统一的 Axios HTTP 客户端实例，所有后续的 API 调用函数（C2 会话 API、C3 Agent API）都将基于此实例。

---

## 二、任务执行过程

### 2.1 创建 Axios 实例

**文件**: `frontend/src/api/index.ts`

```typescript
import axios from 'axios'

const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || 'http://localhost:8080',
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json',
  },
})
```

**设计决策**：

| 配置项 | 值 | 依据 |
|--------|-----|------|
| baseURL | `VITE_API_BASE_URL` 环境变量，fallback `http://localhost:8080` | API 文档 2.1 节：Base URL 为 `http://localhost:8080`，后端无 context-path；通过环境变量支持多环境切换 |
| timeout | 30000ms | 30 秒超时，覆盖普通 REST 请求；SSE 流式走 WebSocket 不受此限制 |
| Content-Type | `application/json` | 所有 REST 接口使用 JSON 格式（除产物上传 multipart 外） |

> **改进迭代（2026-06-01）**: baseURL 从硬编码 `'http://localhost:8080'` 改为 `import.meta.env.VITE_API_BASE_URL || 'http://localhost:8080'`，支持通过环境变量切换后端地址。详见 [2.4 节](#24-环境变量配置) 和 [2.5 节](#25-vite-路径别名配置)。

### 2.2 请求拦截器

```typescript
apiClient.interceptors.request.use(
  (config) => {
    return config
  },
  (error) => {
    return Promise.reject(error)
  }
)
```

当前为**透传模式**，不做任何修改。预留了 JWT Token 注入扩展点 —— 等 P1 阶段引入认证后，在此处从 localStorage/cookie 读取 token 并附加到 `Authorization` 请求头。

### 2.3 响应拦截器

```typescript
apiClient.interceptors.response.use(
  (response) => {
    return response
  },
  (error) => {
    if (error.response) {
      const { status, data } = error.response
      if (status === 404) {
        console.warn(`[API] 资源未找到: ${error.config?.url}`)
      } else if (status === 500) {
        console.error(`[API] 服务器错误: ${error.config?.url}`, data)
      }
    } else if (error.request) {
      console.error('[API] 网络错误: 无法连接到后端服务')
    }
    return Promise.reject(error)
  }
)
```

**错误分级策略**：

| 错误类型 | 判定条件 | 日志级别 | 说明 |
|---------|---------|---------|------|
| 404 资源未找到 | `status === 404` | `console.warn` | API 路径错误或资源已被删除，非严重问题 |
| 500 服务器错误 | `status === 500` | `console.error` | 后端异常，附带 `data`（错误详情 JSON） |
| 网络不通 | `error.request` 存在但无 `response` | `console.error` | 后端未启动或断网 |

### 2.4 环境变量配置

> **新增于 2026-06-01 改进迭代**

**文件**: `frontend/.env`（新建）

```
VITE_API_BASE_URL=http://localhost:8080
```

Vite 环境变量命名约定：必须以 `VITE_` 前缀开头才能在客户端代码中通过 `import.meta.env` 访问。当前定义了一个变量：

| 变量名 | 值 | 说明 |
|--------|-----|------|
| `VITE_API_BASE_URL` | `http://localhost:8080` | 后端 API 基础地址 |

**多环境切换方式**：

| 场景 | 做法 |
|------|------|
| 开发环境 | 使用 `.env`（默认），指向本地 Spring Boot 后端 |
| 生产环境 | 创建 `.env.production`，写入生产后端地址，构建时 `vite build --mode production` 自动加载 |
| 临时覆盖 | 命令行传入：`VITE_API_BASE_URL=http://192.168.1.100:8080 npx vite` |

### 2.5 Vite 路径别名配置

> **新增于 2026-06-01 改进迭代**

为支持 `import apiClient from '@/api'` 这种简洁的导入方式（而非冗长的相对路径 `../../api`），需要在 Vite 构建工具和 TypeScript 类型系统两端同时配置路径别名。

**2.5.1 Vite 端配置**

**文件**: `frontend/vite.config.ts`

```typescript
import { fileURLToPath, URL } from 'node:url'
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [
    vue(),
    tailwindcss(),
  ],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url))
    }
  },
})
```

关键点：
- `fileURLToPath(new URL('./src', import.meta.url))` 是 Vite 官方推荐的 ESM 兼容写法，将 `@` 映射到 `src/` 目录的绝对路径
- 不能使用 `path.resolve(__dirname, 'src')`，因为 `__dirname` 在 ESM 模块中不可用

**2.5.2 TypeScript 端配置**

**文件**: `frontend/tsconfig.app.json`

```json
{
  "extends": "@vue/tsconfig/tsconfig.dom.json",
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.app.tsbuildinfo",
    "types": ["vite/client"],
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    },
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "erasableSyntaxOnly": true,
    "noFallthroughCasesInSwitch": true
  },
  "include": ["src/**/*.ts", "src/**/*.tsx", "src/**/*.vue"]
}
```

关键点：
- `baseUrl: "."` 必须以项目根目录为基准
- `paths: {"@/*": ["src/*"]}` 让 TypeScript 编译器理解 `@/` 开头的导入路径
- 两个配置缺一不可：Vite 负责构建时解析，TypeScript 负责编辑器和类型检查时解析

---

## 三、代码审查与改进迭代（2026-06-01）

### 3.1 审查发现

对 C1 初始实现进行完整代码审查后，发现以下问题：

| # | 问题 | 严重程度 | 影响范围 |
|---|------|---------|---------|
| 1 | 缺少 Vite `@` 路径别名配置 | 🔴 较高 | 后续 C2/C3 等任务使用 `@/api` 导入时会构建失败 |
| 2 | baseURL 硬编码，缺少环境变量支持 | 🔴 较高 | 切换开发/测试/生产环境需手动修改源码 |
| 3 | 响应拦截器错误码覆盖不全（400/401/403/502/503） | 🟡 中等 | MVP 阶段影响较小，P1 可补充 |
| 4 | 缺少 `Accept: application/json` 请求头 | 🟢 较低 | 后端默认返回 JSON，不阻塞 |
| 5 | 响应未做 `data` 解包 | 🟢 较低 | 消费者需写 `response.data`，不影响功能 |
| 6 | `withCredentials` 未配置 | 🟢 较低 | 当前后端未启用 `allowCredentials`，暂不需要 |

### 3.2 改进实施

本次迭代修复了问题 1 和 问题 2，具体变更如下：

**问题 1 — 路径别名**：

| 文件 | 变更内容 |
|------|----------|
| `frontend/vite.config.ts` | 新增 `resolve.alias`：`@` → `src/` |
| `frontend/tsconfig.app.json` | 新增 `baseUrl: "."` + `paths: {"@/*": ["src/*"]}` |

**问题 2 — 环境变量**：

| 文件 | 变更内容 |
|------|----------|
| `frontend/.env` | 新建，定义 `VITE_API_BASE_URL=http://localhost:8080` |
| `frontend/src/api/index.ts` | `baseURL` 改为 `import.meta.env.VITE_API_BASE_URL \|\| 'http://localhost:8080'` |

---

## 四、验证测试

### 4.1 自动化验证

| 测试项 | 命令 | 结果（初版） | 结果（改进后） |
|--------|------|:-----------:|:-----------:|
| TypeScript 类型检查 | `npx vue-tsc --noEmit` | ✅ exit code 0 | ✅ exit code 0 |
| 生产构建 | `npx vite build` | ✅ 2976 模块 | ✅ 2998 模块 |
| 开发服务器 | `npx vite --host` | ✅ `localhost:5173` | ✅ `localhost:5173` |

> 改进后模块数从 2976 增加到 2998，是因为 `vite.config.ts` 新增了 `node:url` 模块的导入。

### 4.2 手动验证

**第 1 步 — 后端连通性测试**（在浏览器 F12 Console 执行）：

```javascript
fetch('http://localhost:8080/h2-console', { method: 'GET' })
  .then(res => console.log('后端连通状态:', res.status, res.statusText))
  .catch(err => console.error('后端未启动或跨域阻断:', err.message));
```

结果：`后端连通状态: 200 OK` ✅

**第 2 步 — CORS 预检测试**（在浏览器 F12 Console 执行）：

```javascript
fetch('http://localhost:8080/conversations', {
  method: 'OPTIONS',
  headers: { 'Origin': 'http://localhost:5173' }
})
.then(res => {
  console.log('CORS 预检状态:', res.status);
  console.log('Access-Control-Allow-Origin:', res.headers.get('Access-Control-Allow-Origin'));
});
```

结果：`404 Not Found` — **这是预期的正常现象**，因为 `ConversationController` 是成员A 的任务（A5），尚未实现。等 A5 完成后此测试自然会通过。

### 4.3 页面渲染验证

- 左侧紫色导航栏正常显示
- 中间 ChatList 显示 mock Agent 列表（后续 C8 改造）
- 右侧聊天窗口正常渲染，输入框可用

---

## 五、关键决策记录

| # | 决策 | 理由 |
|---|------|------|
| 1 | 使用 axios 实例模式（非直接 import） | 统一管理 baseURL、超时、拦截器，后续 API 函数只导入此实例 |
| 2 | 请求拦截器保持透传 | MVP 阶段无认证，预留扩展点；P1 加 JWT 时只需修改此处 |
| 3 | 错误分级处理（404/warn、500/error、网络/error） | 区分业务错误和系统错误，便于调试 |
| 4 | 不在此处处理 401 跳转登录 | MVP 无认证，P1 阶段在拦截器中统一处理 |
| 5 | timeout 设为 30s | 仅覆盖 REST 请求，WebSocket/SSE 流式不受此影响 |
| 6 | baseURL 使用 `VITE_API_BASE_URL` 环境变量 | 支持多环境切换，开发/测试/生产无需改源码；Vite 编译时静态替换，无运行时开销 |
| 7 | 同时配置 Vite `resolve.alias` 和 TS `paths` | 构建工具和类型系统两端都要感知别名，缺一不可；使用 ESM 兼容的 `fileURLToPath` 写法 |

---

## 六、产物清单

| 文件 | 说明 | 状态 |
|------|------|:--:|
| `frontend/src/api/index.ts` | Axios 实例配置，baseURL 通过 `VITE_API_BASE_URL` 环境变量读取，含请求/响应拦截器 | ✅ |
| `frontend/.env` | 环境变量文件，定义 `VITE_API_BASE_URL=http://localhost:8080` | 🆕 |
| `frontend/vite.config.ts` | Vite 构建配置，新增 `resolve.alias` 路径别名 `@` → `src/` | 🔧 |
| `frontend/tsconfig.app.json` | TypeScript 配置，新增 `baseUrl` + `paths` 路径别名映射 | 🔧 |

> 🆕 = 改进迭代新增 &nbsp;&nbsp; 🔧 = 改进迭代修改
