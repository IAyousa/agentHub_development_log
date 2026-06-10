# 开发过程记录 — PR/9 PostgreSQL Profile 化 & PR/10 JWT 认证审查合并

> **日期**: 2026-06-09  
> **关联 PR**: PR/9 (kyo19c)、PR/10 (kyo19c)  
> **关联提交**: `2f3fd80`（PG Profile）、`6d96c0e`（JWT 认证）  
> **涉及范围**: Spring Boot 数据源配置、Spring Security JWT 认证、配置文件分离、环境变量占位符模式

---

## 一、背景

kyo19c 提交了两个 P1 阶段后端任务 PR：
- **PR/9**: B2 — H2→PostgreSQL 数据库迁移
- **PR/10**: B1 — Spring Security + JWT 用户认证

当前项目使用 H2 文件数据库、无认证机制。这两个 PR 是 P1 阶段（生产就绪）的核心基础设施。

---

## 二、PR/9: PostgreSQL 迁移 → Profile 化改造

### 2.1 原始内容

| 文件 | 改动 |
|------|------|
| `docker-compose.yml` | **新建** — PostgreSQL 15-alpine 容器，端口 5432 |
| `application.yml` | H2 配置**直接替换**为 PostgreSQL |
| `data.sql` | H2 `MERGE INTO` 语法 → PostgreSQL `INSERT ON CONFLICT` |

### 2.2 影响分析

**直接合并会导致本地开发环境炸裂**：当前机器无 Docker、无 PostgreSQL，Spring Boot 启动即报 `Connection refused`。且没有 H2 回退机制。

### 2.3 改造方案：Spring Profile 分离

保持 H2 为默认（日常开发零依赖），PostgreSQL 作为可选 Profile：

| 操作 | 文件 | 说明 |
|------|------|------|
| 不动 | `application.yml` | 保持 H2 文件数据库 |
| 不动 | `data.sql` | 保持 H2 语法种子数据 |
| 新建 | `application-pg.yml` | PG Profile 配置，密码用 `${PG_PASSWORD:default}` |
| 新建 | `data-pg.sql` | PG 幂等种子数据 |
| 保留 | `docker-compose.yml` | PG 本地开发容器 |

使用方式：
```bash
# 日常开发 — H2 零依赖
mvn spring-boot:run

# PostgreSQL 模式
docker-compose up -d
mvn spring-boot:run -Dspring-boot.run.profiles=pg
```

### 2.4 配置文件 example 模板

同步为团队创建了标准化配置文件模板：

| 文件 | 提交？ | 说明 |
|------|:--:|------|
| `agent-service/.env.example` | ✅ | API Key 模板 |
| `agent-service/.gitignore` | ✅ | 保护 `.env` |
| `application-secret.yml.example` | ✅ | 敏感配置模板（JWT + PG） |
| `.gitignore` → `application-secret.yml` | ✅ | 保护本地真实 secret |

---

## 三、PR/10: JWT 认证审查

### 3.1 代码审查

| 文件 | 改动 | 审查结论 |
|------|------|:--:|
| `pom.xml` | +4 依赖（spring-security + jjwt 0.12.5） | ✅ 版本正确 |
| `SecurityConfig.java` | 注释占位 → 实际安全配置 | ⚠️ 需增强 |
| `AuthController.java` | **新建** — `/auth/register` + `/auth/login` | ✅ 设计规范 |
| `JwtAuthenticationFilter.java` | **新建** — Bearer Token 提取验证 | ✅ 标准实现 |
| `JwtTokenProvider.java` | **新建** — HS256 JWT 生成/验签 | ⚠️ secret 硬编码 |
| `application.yml` | +3 行 JWT 配置 | 🔴 secret 明文 |

### 3.2 发现的问题与修复

| 问题 | 原因 | 修复 |
|------|------|------|
| 🔴 **JWT secret 明文硬编码** | `jwt.secret: AgenthubJwtSecretKey2026P1B1!Secure` | 改为 `${JWT_SECRET:agent-hub-dev-jwt-secret-key-for-jjwt-256bit}` |
| 🔴 **默认 secret 长度不足** | 原值 31 字符 = 248 bits，jjwt 0.12.x 要求 ≥256 bits | 调整为 44 字符（352 bits） |
| 🟡 **SecurityConfig 无 EntryPoint** | 未认证请求默认返回 403，无 JSON 错误信息 | 新增 `AuthenticationEntryPoint`（401 JSON） + `AccessDeniedHandler`（403 JSON） |
| 🟢 **敏感配置分离** | 与 PG 配置保持一致模式 | 新增 `application-secret.yml.example` 模板 |

### 3.3 jjwt 256 bits 密钥限制

这是一个在测试中才暴露的隐蔽问题。jjwt 0.12.x 的 `Keys.hmacShaKeyFor()` 会检查密钥字节数是否 ≥ 算法输出位数。HS256 的输出是 256 bits，所以需要 ≥ 32 字节（256 bits）。原默认值 `AgenthubJwtSecretKey2026P1B1!Secure` 仅 31 字符 = 248 bits，导致启动时抛出 `WeakKeyException`。

**教训**：加密库的版本升级可能引入新的最低安全要求，默认值必须实测验证。

### 3.4 SecurityConfig 最终架构

```
HTTP Request
  → CorsFilter (Servlet Filter)
  → SecurityFilterChain
      → JwtAuthenticationFilter: 提取 Authorization: Bearer <token>
          ├─ Token 有效 → SecurityContext.setAuthentication()
          │   → .anyRequest().authenticated() → 通过 → Controller
          └─ Token 无效/缺失
              → AuthenticationEntryPoint → 401 {"error":"UNAUTHORIZED","message":"请先登录"}
```

放行路径：
```
/auth/**      ← 注册、登录
/ws-chat/**   ← WebSocket STOMP 握手
/h2-console/** ← 开发数据库工具
```

### 3.5 测试验证

| 测试 | 请求 | 预期 | 结果 |
|------|------|------|:--:|
| 注册 | `POST /auth/register` `{"username":"testuser","password":"test123456"}` | 201 + token | ✅ |
| 登录 | `POST /auth/login` `{"username":"testuser","password":"test123456"}` | 200 + token | ✅ |
| 无 Token 访问 | `GET /agents` | 401 | ✅ |
| 有效 Token 访问 | `GET /agents` + `Authorization: Bearer <token>` | 200 | ✅ |
| 伪造 Token | `GET /agents` + `Authorization: Bearer invalid` | 401 | ✅ |
| H2 放行 | `GET /h2-console` | 302（正常） | ✅ |

---

## 四、配置文件标准化模式

通过 PR/9 和 PR/10 的改造，确立了项目统一的配置管理规范：

### 4.1 `${VAR:default}` 占位符模式

```
应用启动
  ├─ 环境变量 JWT_SECRET=xxx  → 优先使用（生产）
  ├─ application.yml 默认值    → 无环境变量时使用（开发）
  └─ 都没有                    → 启动报错
```

### 4.2 文件结构

```
提交到仓库                        保留在本地（gitignored）
───────────                      ──────────
.env.example         ─复制→      .env
application-secret.yml.example ─复制→ application-secret.yml
application.yml (含 ${VAR:default})
application-pg.yml
```

### 4.3 开发者首次使用

```bash
# 1. 复制配置模板
cd backend-java/src/main/resources
cp application-secret.yml.example application-secret.yml
cd agent-service
cp .env.example .env

# 2. 填入真实 API Key（或留空使用默认值）
# 3. 启动 — 零额外配置
mvn spring-boot:run
```

---

## 五、合并后的 dev 分支

```
6d96c0e feat(backend): P1-B1 Spring Security + JWT 用户认证   ← PR/10
2f3fd80 feat(backend,agent): PostgreSQL Profile + example 模板 ← PR/9 改造
da19215 fix(agent,backend,frontend): 降级策略+编排修复
6596a12 feat(agent,backend,frontend): 多Agent协作编排
3deba58 fix(agent): Codex会话持久化优化 + Windows修复
e91769f feat(agent): Codex会话持久化
```

---

## 六、经验总结

### 6.1 数据库迁移不应是"替换"

PR/9 直接把 H2 配置替换为 PG，这种"硬切换"在生产部署前会阻塞所有本地开发。Spring Profile 是成熟方案——默认 H2 零依赖开发，需要 PG 时激活 Profile 即可。

### 6.2 加密库的隐性要求

jjwt 在 `hmacShaKeyFor()` 中静默检查密钥长度，不足时直接抛异常。这类要求不在编译时可见，只能在启动时暴露。对于密钥生成逻辑，应该在单元测试中验证 `new JwtTokenProvider(secret).generateToken("test")` 能正常执行。

### 6.3 Security 默认行为的坑

Spring Security 默认 `AuthenticationEntryPoint` 在 REST API 场景下返回 403（而非 401），且响应体为 HTML。不加 `exceptionHandling()` 配置会导致前端收到无意义的 403 状态码，难以区分"未登录"和"无权限"。

### 6.4 配置文件的规范化价值

统一 `${VAR:default}` 模式后，团队新成员 clone 仓库即可启动——不需要研究"该设哪些环境变量"。`example` 模板文件既是文档也是基准配置。

### 6.5 Commit Message 规范

`type(module): description` 格式（module = `frontend`/`backend`/`agent`）让 `git log --oneline` 直接可读。PR/10 合并前通过 soft reset 将两个 commit 压缩为一条干净记录，避免了"修 bug 的 commit 跟在功能 commit 之后"的脏历史。
