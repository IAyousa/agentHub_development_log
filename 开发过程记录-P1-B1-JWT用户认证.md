# 开发过程记录 — P1-B1 Spring Security + JWT 用户认证

> **日期**: 2026-06-08
> **任务编号**: P1-B1
> **所属模块**: backend-java / 安全认证
> **前置依赖**: User 实体+Repository 已就绪 ✅、SecurityConfig 占位已存在 ✅、P0 全部完成 ✅

---

## 一、背景

### 1.1 为什么要做

MVP 阶段所有请求以 `"system"` 硬编码作为操作者，没有用户身份概念——任何人知道端口就能调接口、发消息、删会话。P1 阶段需要引入用户认证，实现：

1. 用户注册/登录
2. 后续请求携带 Token 证明身份
3. 未认证请求被拦截（返回 403）

### 1.2 当前起点的优势

- `users` 表已有完整的 JPA 实体 (`User.java`)，包含 `id/username/password/createdAt/updatedAt`
- `UserRepository.java` 已有 `findByUsername()` 和 `existsByUsername()` 方法
- `SecurityConfig.java` 已有 P1 激活蓝图（注释形式的 SecurityFilterChain + JwtAuthenticationFilter）
- Spring Boot 3.2.5 + Java 17 生态完全支持

---

## 二、任务执行过程

### 2.1 添加依赖

**操作内容**: `pom.xml` 添加 4 个依赖

```xml
<!-- Spring Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- JWT (io.jsonwebtoken 0.12.5) -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
```

| 决策项 | 选择 | 理由 |
|--------|------|------|
| JWT 库 | jjwt 0.12.5 | 业界标准，免配置 Jackson 序列化，API 简洁 |
| jjwt-impl/jjwt-jackson scope | runtime | 编译时只需 api，运行时自动加载实现 |

### 2.2 创建 JwtTokenProvider

**操作内容**: 新建 `security/JwtTokenProvider.java`

```java
@Component
public class JwtTokenProvider {
    private final SecretKey key;
    private final long expirationMs;

    public JwtTokenProvider(
            @Value("${jwt.secret}") String secret,
            @Value("${jwt.expiration}") long expirationMs) {
        this.key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.expirationMs = expirationMs;
    }

    public String generateToken(String userId) {
        Date now = new Date();
        return Jwts.builder()
                .subject(userId)
                .issuedAt(now)
                .expiration(new Date(now.getTime() + expirationMs))
                .signWith(key)
                .compact();
    }

    public String getUserIdFromToken(String token) {
        return parseClaims(token).getSubject();
    }

    public boolean validateToken(String token) {
        try { parseClaims(token); return true; }
        catch (JwtException | IllegalArgumentException e) { return false; }
    }

    private Claims parseClaims(String token) {
        return Jwts.parser().verifyWith(key).build()
                .parseSignedClaims(token).getPayload();
    }
}
```

| 决策项 | 选择 | 理由 |
|--------|------|------|
| 签名算法 | HS256（HMAC-SHA256） | 对称密钥，适合单体应用，无需管理公私钥对 |
| 过期时间 | 86400000ms（24小时） | 可在 yml 调整 |
| Token 载荷 | 仅存 userId（subject） | minimum viable —— 不存多余信息，减小 Token 体积 |

**错误与修复**：

| # | 错误现象 | 根因 | 修复 |
|---|---------|------|------|
| 1 | `WeakKeyException: 224 bits is not secure enough` | jjwt 0.12 要求 HMAC-SHA 密钥 >= 256 位（32字符），原密钥 `AgenthubJwtSecretKey2026P1B1` 只有 28 字符 | 加长到 35 字符：`AgenthubJwtSecretKey2026P1B1!Secure`（280 位） |

### 2.3 创建 JwtAuthenticationFilter

**操作内容**: 新建 `security/JwtAuthenticationFilter.java`

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) {
        String token = extractToken(request);

        if (StringUtils.hasText(token) && jwtTokenProvider.validateToken(token)) {
            String userId = jwtTokenProvider.getUserIdFromToken(token);
            UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(userId, null, Collections.emptyList());
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        filterChain.doFilter(request, response);
    }

    private String extractToken(HttpServletRequest request) {
        String bearer = request.getHeader("Authorization");
        if (StringUtils.hasText(bearer) && bearer.startsWith("Bearer ")) {
            return bearer.substring(7);
        }
        return null;
    }
}
```

| 决策项 | 选择 | 理由 |
|--------|------|------|
| 基类 | OncePerRequestFilter | 保证同一请求只执行一次过滤，避免 forward/include 时重复处理 |
| Bean 注册方式 | `@Component` | 可被 SecurityConfig 直接注入，无需手动 new |
| 无 Token 时行为 | 继续 chain，不抛异常 | 由 SecurityFilterChain 的 `.authorizeHttpRequests()` 决定是否拦截 |
| Authorities | `Collections.emptyList()` | MVP 不区分角色，P1 后期可扩展 RBAC |

### 2.4 创建 AuthController

**操作内容**: 新建 `controller/AuthController.java`

```java
@RestController
@RequestMapping("/auth")
@RequiredArgsConstructor
@Validated
public class AuthController {

    private final UserRepository userRepository;
    private final JwtTokenProvider jwtTokenProvider;
    private final BCryptPasswordEncoder passwordEncoder;

    @PostMapping("/register")
    public ResponseEntity<Map<String, Object>> register(@Valid @RequestBody RegisterRequest request) {
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new IllegalArgumentException("用户名已存在");
        }
        User user = new User();
        user.setUsername(request.getUsername().trim());
        user.setPassword(passwordEncoder.encode(request.getPassword()));
        userRepository.save(user);
        String token = jwtTokenProvider.generateToken(user.getId());
        // 返回 token + userId + username
    }

    @PostMapping("/login")
    public ResponseEntity<Map<String, Object>> login(@Valid @RequestBody LoginRequest request) {
        User user = userRepository.findByUsername(request.getUsername().trim())
                .orElseThrow(() -> new IllegalArgumentException("用户名或密码错误"));
        if (!passwordEncoder.matches(request.getPassword(), user.getPassword())) {
            throw new IllegalArgumentException("用户名或密码错误");
        }
        String token = jwtTokenProvider.generateToken(user.getId());
        // 返回 token + userId + username
    }
}
```

| 决策项 | 选择 | 理由 |
|--------|------|------|
| 路径前缀 | `/auth` | 语义清晰，SecurityConfig 中 `permitAll()` 一行搞定 |
| DTO 风格 | 静态内部类 | 遵循项目现有规范（ConversationController 同样做法） |
| 登录/注册错误统一信息 | "用户名或密码错误" | 不区分具体原因，防止用户名枚举攻击 |
| 密码编码 | BCryptPasswordEncoder（Bean 注入） | Spring Security 标准，自动加盐，迭代 1024 次 |

### 2.5 重写 SecurityConfig

**操作内容**: 把占位注释替换为生效的 `SecurityFilterChain`

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .headers(headers -> headers.frameOptions(frame -> frame.sameOrigin()))
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**").permitAll()
                .requestMatchers("/ws-chat/**").permitAll()
                .requestMatchers("/h2-console/**").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthenticationFilter,
                UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    @Bean
    public BCryptPasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

| 决策项 | 选择 | 理由 |
|--------|------|------|
| CSRF | 关闭 | JWT 天然防 CSRF（Token 在 Header 不在 Cookie），且是纯 API 无表单 |
| Session | STATELESS | JWT 无状态认证，不需要服务端 Session |
| `/ws-chat/**` | permitAll | WebSocket 握手走 HTTP，STOMP 自有认证机制，不能拦截 |
| `/h2-console/**` | permitAll | 开发调试需要，生产环境会删除 |
| Frame Options | `sameOrigin()` | H2 控制台使用 frame，禁用会导致无法访问 |
| Filter 位置 | `addFilterBefore(..., UsernamePasswordAuthenticationFilter.class)` | JWT 过滤器在 Spring 默认表单登录过滤器之前执行 |

### 2.6 添加 JWT 配置项

**操作内容**: `application.yml` 添加 `jwt.secret` 和 `jwt.expiration`

```yaml
jwt:
  secret: AgenthubJwtSecretKey2026P1B1!Secure
  expiration: 86400000
```

---

## 三、技术要点讲解

### 3.1 Spring Security 过滤器链

Spring Security 本质是一串过滤器，请求到达 Controller 之前要逐个通过：

```
请求 → CSRF Filter → Auth Filter → SecurityFilterChain 路由决策 → Controller
                        │                    │
                 JwtAuthenticationFilter     ├─ /auth/** → 放行
                 (我们插在这里)               ├─ /ws-chat/** → 放行
                                             └─ 其他 → 需认证 → 检查 SecurityContext
```

**SecurityContextHolder** 是线程绑定的容器，存"当前请求是谁"：

```java
// Filter 里设置：
SecurityContextHolder.getContext().setAuthentication(authentication);

// Controller 里读取：
String userId = SecurityContextHolder.getContext().getAuthentication().getName();
```

### 3.2 JWT 零状态认证原理

Session 模式：服务器存用户状态 → 需要 Redis/内存同步
JWT 模式：用户信息在 Token 自身 → 服务器不存任何东西

```
Token 结构:
  header.payload.signature
  │      │         │
  │      │         └─ HMAC-SHA256(base64(header)+"."+base64(payload), secret)
  │      └─ {"sub":"userId","iat":签发时间,"exp":过期时间}
  └─ {"alg":"HS256"}
```

任何人改了 payload（比如把 userId 改成别人的），签名对不上 → 拒绝。

### 3.3 BCrypt 密码存储

不使用 MD5/SHA256 直接哈希，因为彩虹表可以反向查询。BCrypt 自动加盐 + 故意慢：

- `$2a$10$...` — 版本 + 迭代次数（2^10=1024） + 随机盐 + 哈希值
- `encode("123456")` 每次结果不同（随机盐）
- `matches("123456", hash)` 自动从 hash 中提取盐做验证

### 3.4 OncePerRequestFilter vs Filter

普通 `Filter` 在一次 HTTP 请求中可能被调用多次（forward/include 时会重复过过滤链）。`OncePerRequestFilter` 保证每个请求的 `doFilterInternal` 只执行一次，避免重复解析 Token。

---

## 四、验证测试

### 4.1 自动化验证

| 测试项 | 命令 | 结果 |
|--------|------|------|
| 新增文件存在 | `ls security/*.java controller/AuthController.java` | ✅ |
| pom.xml 依赖 | `grep "security\|jjwt" pom.xml` | ✅ |
| yml 配置 | `grep -A2 "jwt:" application.yml` | ✅ |
| Maven 编译 | `mvn compile -q` | ✅ |
| Spring Boot 启动 | `mvn spring-boot:run` | ✅（密钥长度修复后） |

### 4.2 手动功能测试清单

| # | 测试项 | 请求 | 预期 | 结果 |
|---|--------|------|------|:--:|
| 1 | 无 Token 访问 | GET /agents | 403 | ✅ |
| 2 | 用户注册 | POST /auth/register | 201 + token | ✅ |
| 3 | 用户登录 | POST /auth/login | 200 + token | ✅ |
| 4 | 带 Token 访问 | GET /agents + Bearer | 200 + 正常数据 | ✅ |
| 5 | 错误密码 | POST /auth/login wrong pass | 400 | ✅ |
| 6 | 无效 Token | GET /agents + fake token | 403 | ✅ |
| 7 | 重复注册 | POST /auth/register same name | 400 | ✅ |
| 8 | 短密码 | POST /auth/register "12" | 400 | ✅ |

### 4.3 发现并修复的 Bug

| # | 错误现象 | 根因 | 修复 |
|---|---------|------|------|
| 1 | 启动时 `WeakKeyException: 224 bits is not secure enough` | jjwt 0.12 要求 HMAC 密钥 >= 256 位，原密钥 28 字符=224 位 | 加长到 35 字符（280 位） |

---

## 五、关键决策记录

| # | 决策 | 结论 | 理由 |
|---|------|------|------|
| 1 | 认证方案 | JWT 无状态认证 | 无需 Redis，前后端分离友好，重启不丢会话 |
| 2 | JWT 库 | jjwt 0.12.5 | 业界标准，jjwt 0.12 API 现代化 |
| 3 | 密码存储 | BCrypt | Spring Security 标准，自动加盐防彩虹表 |
| 4 | Session 管理 | STATELESS | JWT 不需要服务端 Session |
| 5 | CSRF | 关闭 | JWT 在 Header 不在 Cookie，天然防 CSRF |
| 6 | 登录错误信息 | 统一"用户名或密码错误" | 防止用户名枚举 |
| 7 | DTO 位置 | AuthController 静态内部类 | 遵循项目现有规范 |

---

## 六、产物清单

| 文件 | 操作 | 说明 |
|------|------|------|
| `backend-java/pom.xml` | 修改 | 添加 security + jjwt(api/impl/jackson) 共 4 个依赖 |
| `security/JwtTokenProvider.java` | 新建 | JWT 生成/验证/解析，HS256，24h 过期 |
| `security/JwtAuthenticationFilter.java` | 新建 | Bearer Token 提取 → SecurityContext 注入 |
| `controller/AuthController.java` | 新建 | POST /auth/register + /auth/login，BCrypt |
| `config/SecurityConfig.java` | 重写 | 占位→SecurityFilterChain，/auth/**放行 |
| `application.yml` | 修改 | 添加 jwt.secret(280位) + jwt.expiration(1天) |

### 代码统计

| 指标 | 数值 |
|------|------|
| 新增文件 | 3 |
| 修改文件 | 3 |
| 新增代码行 | 274 |
| 净减少代码行 | 34 |

---

## 七、数据流/架构影响

**认证前**：
```
所有请求 → Controller → "system" 硬编码用户
```

**认证后**：
```
未认证请求 → SecurityFilterChain 拦截 → 403
POST /auth/register → BCrypt 加密 → JPA save → 返回 JWT
POST /auth/login → 查 users 表 → BCrypt 验证 → 返回 JWT
后续请求 + Bearer Token → JwtAuthFilter → SecurityContextHolder
  → Controller 可从 SecurityContext 获取真实 userId
```

---

## 八、后续任务依赖

| 任务 | 可开始时机 | 关联 |
|------|----------|------|
| P1-C1 登录/注册 UI | 本 PR 合并后 | 前端 Auth API + Pinia auth store + 路由守卫 |
| P2 角色鉴权 | 需要时 | 当前 MVP 不区分角色，可扩展 RBAC |
| 现有 Controller 使用真实用户 | 本 PR 合并后 | 将 `"system"` 替换为 `SecurityContextHolder.getContext().getAuthentication().getName()` |
