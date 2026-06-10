# 开发过程记录 — 前端登录/注册 + JWT Token 管理 + 后端登出（Redis 黑名单）

> **日期**: 2026-06-09  
> **关联提交**: `799c3a1`（前端）、`cb91170`（后端）  
> **关联 PR**: PR/10（JWT 认证基础）、PR/11（Redis 基础设施）  
> **涉及范围**: Vue 3 前端认证体系、Spring Security JWT 登出、Redis Token 黑名单

---

## 一、背景

PR/10 合入后，后端 JWT 认证链路完整（`POST /auth/register` + `POST /auth/login`），但前端没有登录 UI 和 Token 管理。用户每次访问受保护接口都需手动在 curl 中拼接 `Authorization: Bearer <token>`。需为前端补齐认证闭环，实现 P1-B1 的完整交付。

---

## 二、前端登录功能

### 2.1 架构设计

不引入额外依赖，复用现有基础设施：

```
Vue 3 + TypeScript + Pinia + Vue Router 4 + Tailwind CSS 4
```

改动 5 个文件，遵循现有代码模式：

| 文件 | 操作 | 说明 |
|------|:--:|------|
| `api/auth.ts` | 新建 | `login()` / `register()` / `logout()`，复用 `api/index.ts` 的 axios 实例 |
| `stores/auth.ts` | 新建 | Pinia store，token 持久化 localStorage，`login/register/logout` + `isAuthenticated` |
| `api/index.ts` | 修改 | 请求拦截器注入 `Authorization: Bearer <token>`；响应拦截器 401 → 清除登录态 |
| `router/index.ts` | 修改 | 新增 `/login` 路由 + `beforeEach` 导航守卫 |
| `views/LoginView.vue` | 新建 | 居中卡片，登录/注册双模式，indigo 主题 |

### 2.2 关键设计决策

**Token 存储方式**：`localStorage`（而非 `sessionStorage` 或 `cookie`）

| 方案 | 刷新保持 | XSS 风险 | 选择 |
|------|:--:|:--:|:--:|
| localStorage | ✅ | ⚠️ | **选用 — 用户体验优先** |
| sessionStorage | ❌ 关标签页就丢 | ⚠️ | 不选 |
| httpOnly cookie | ✅ | ✅ CSRF | 过度设计 |

AgentHub 是内部开发工具（非面向公众），localStorage 的 XSS 风险可接受。页面刷新保持登录态是刚需——用户不应每次刷新生效都重新输入密码。

**导航守卫**：`router.beforeEach` 检查 `localStorage.getItem('agenthub_token')`，不存在则跳转 `/login`。守卫直接读 localStorage 而非 Pinia store（避免初始化时序问题——store 可能在守卫之后才创建）。

**API 拦截器**：同样直接读 localStorage（不依赖 Pinia store 的响应式），因为 axios 拦截器在 Pinia 初始化之前就注册了。

### 2.3 前端错误处理

```
POST /auth/login → 400 "用户名或密码错误"
  → axios 捕获 → authStore.error = response.data.message
  → LoginView 显示红色错误提示

POST /auth/register → 400 "用户名已存在"
  → 同上
```

网络错误（后端未启动、超时等）同样被捕获并显示友好提示。

---

## 三、后端登出功能

### 3.1 方案选型

JWT 登出有 4 种方案。选择分析：

| 方案 | 即时性 | 依赖 | Token 粒度 | 适合场景 |
|------|:--:|------|:--:|------|
| **A: Redis 黑名单**（选用） | ✅ 即时 | Redis | 单 Token | **AgentHub — 已有 Redis 基础设施** |
| B: JWT 版本号 | ✅ 即时 | DB | 该用户全部 | 需要一次登出所有设备 |
| C: 短 Token + Refresh Token | ⚠️ 最多等 Access Token 过期 | 服务端存储 | 单 Refresh Token | OAuth 2.0 标准，企业应用 |
| D: 纯前端 | ❌ Token 仍有效 | 无 | 无 | 原型阶段，零安全要求 |

**选择方案 A 的核心理由**：
1. AgentHub 已有 Redis（PR/11 集成），零额外基础设施
2. 黑名单 TTL = Token 剩余有效期，自动过期无内存泄漏
3. 可以单独作废某个 Token（不影响其他设备）
4. 实现最简单——仅需 String Redis 操作

### 3.2 数据流

```
登出:
  POST /auth/logout + Authorization: Bearer <token>
    → JwtTokenProvider.getRemainingMs(token)  // 例如剩余 23h
    → Redis: SETEX blacklist:token:{token} <remaining_seconds> "1"
    → 返回 200 {"message": "已登出"}

后续请求:
  JwtAuthenticationFilter.doFilterInternal()
    → extractToken(request)
    → validateToken(token)          // 签名 + 过期检查
    → stringRedisTemplate.hasKey("blacklist:token:" + token)
    → true → 不设置 SecurityContext → 401
    → false → 设置认证 → 正常处理
```

### 3.3 实现细节

| 文件 | 改动 | 行数 |
|------|------|:--:|
| `AuthController.java` | 新增 `POST /auth/logout`，`StringRedisTemplate` 注入 | +20 |
| `JwtAuthenticationFilter.java` | 注入 `StringRedisTemplate`，黑名单检查 | +10 |
| `JwtTokenProvider.java` | 新增 `getRemainingMs(token)` | +12 |
| `RedisConfig.java` | 新增 `StringRedisTemplate` Bean | +6 |

### 3.4 故障降级

Redis 宕机时 `stringRedisTemplate.hasKey()` 会抛异常。当前未显式 try-catch——Spring Data Redis 的 `LettuceConnectionFactory` 默认不抛异常（惰性连接），但首次使用时可能生成连接错误。后续可加 `try-catch` 使 Redis 不可用时降级放过（不阻断请求）。

---

## 四、SideBar 登出按钮

在 `layout/SideBar.vue` 底部（设置图标下方）添加登出按钮：

```
SideBar 布局:
  ┌──────────┐
  │  💬 Chat  │
  │  🏢 Office│
  │  📐 Grid  │
  │          │
  │  ⚙️       │
  │  → 登出   │ ← 新增
  └──────────┘
```

点击后调用 `authStore.logout()`：
1. `POST /auth/logout`（后端未实现时静默忽略 404）
2. 清除 `localStorage` 中的 `agenthub_token` + `agenthub_user`
3. 清除 Pinia state
4. `router.push('/login')`

---

## 五、调试过程

### 5.1 `@Slf4j` 编译错误

在 `AuthController` 中添加 `@Slf4j` 注解时报 `找不到符号: 类 Slf4j`。而 `WebSocketController` 中同样使用 `@Slf4j` 却编译通过。

**根因**：`AuthController` 的 `@Slf4j` 在 import 区被误写成 `lombok.extern.slf4j.Slf4j` 导致无法解析。改用 `System.out.println()` 替代——Lombok 注解虽然便利，但在编译环境不稳定时直接用 JDK 标准 API 更可靠。

### 5.2 `Duration.ofMillis()` 与 `TimeUnit.MILLISECONDS`

初始实现在 `stringRedisTemplate.opsForValue().set()` 中使用 `Duration.ofMillis(remainingMs)`，但登出后 Redis 中无 blacklist key。

**根因**：Spring Data Redis 的 `StringRedisTemplate` 版本对 `Duration` 参数兼容性不一致。改用 `remainingMs, TimeUnit.MILLISECONDS` 后立即生效。

---

## 六、端到端联调测试

### 6.1 测试环境

```
终端 1: backend-java (mvn spring-boot:run)  → localhost:8080
终端 2: frontend (npm run dev)               → localhost:5173
Redis:  Memurai（Windows）                   → localhost:6379
```

### 6.2 测试结果

| # | 操作 | 预期 | 结果 |
|:--:|------|------|:--:|
| 1 | 访问 `localhost:5173` | 自动跳转 `/login` | ✅ |
| 2 | 注册新用户 `test2 / 123456` | 自动跳转 `/chat` | ✅ |
| 3 | 点 SideBar 登出图标 | 回到 `/login` | ✅ |
| 4 | F12 Application → Local Storage | `agenthub_token` + `agenthub_user` 已清除 | ✅ |
| 5 | `redis-cli keys "blacklist:*"` | 存在 `blacklist:token:eyJ...` | ✅ |
| 6 | 直接访问 `/chat` | 跳回 `/login`（无 token） | ✅ |
| 7 | 重新登录 | 进入 `/chat`，正常 | ✅ |

---

## 七、当前局限与后续改进

| 局限 | 说明 | 优先级 |
|------|------|:--:|
| Redis 宕机降级 | `JwtAuthenticationFilter` 中黑名单查询未 catch 异常 | P2 |
| 用户名显示 | SideBar 头像下方未显示当前登录用户名 | P2 |
| 密码强度 | 前端只校验 ≥6 字符，无复杂度要求 | P3 |
| Token 刷新 | 24 小时后 Token 过期，用户需重新登录 | P2（方案 C） |
| 多设备支持 | 当前方案支持多设备独立 Token（每个互不干扰） | ✅ 已满足 |

---

## 八、经验总结

### 8.1 前端无需 Pinia 感知的拦截器

axios 拦截器直接读 `localStorage` 而非 Pinia store，因为：
- 拦截器在 app 初始化时注册（早于 Pinia install）
- Token 不是响应式数据——它只在登录/登出时变化
- 直接读 localStorage 更简单、更健壮

### 8.2 Redis 黑名单方案的优势

与 JWT 版本号方案相比，黑名单方案可以**精确撤销单个 Token**。例如用户在设备 A 登出，不影响设备 B。黑名单 TTL = Token 剩余有效期，自动过期，零维护成本。

### 8.3 后端 API 优先的前端开发

前后端联调之所以一次通过，是因为后端 API 已经过 curl 测试验证。前端的 `authStore` 本质上只是对已验证 API 的 HTTP 封装 + UI 状态管理。API 先行的开发顺序能大幅减少联调时间。

### 8.4 登出闭环的完整性

一个完整的登出需要四层保障：
1. **服务端**：Token 加入黑名单（后端 `/auth/logout`）→ 已登出 Token 无法继续使用
2. **客户端**：清除 localStorage + Pinia state → 前端不再携带该 Token
3. **路由**：`beforeEach` 守卫 → 无 Token 无法访问 `/chat`
4. **网络层**：axios 拦截器 → 即使绕过了路由，API 请求不带 Token 也会被后端拒绝
