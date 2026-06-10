# 开发过程记录 — PR/11 审查、有用部分提取与 Redis 缓存集成

> **日期**: 2026-06-09  
> **关联 PR**: PR/11 (Liu-Weizhi1219)  
> **关联提交**: `6a4d23d`（PR/11 提取）、`9d1c7ef`（接线 + 序列化修复）  
> **涉及范围**: Redis 缓存基础设施、Agent 元数据缓存、Jackson 序列化、前后端对接规划

---

## 一、背景

Liu-Weizhi1219 提交了 PR/11，标题为 "P1-A1 + A4-A7"。实际包含 34 个文件、+6500/-1342 行。审查发现该 PR 存在严重问题：基于旧版 dev 分支开发，混入了倒退性改动和无关项目文件。

---

## 二、PR/11 审查

### 2.1 全量分析

| 类别 | 文件数 | 判定 |
|------|:--:|------|
| 倒退 — 删除已有功能 | 2 | pom.xml 删除 JWT/Security/Springdoc 依赖；application.yml 删除 spring.config.import + JWT 配置 |
| 重复 — dev 已有更完整版 | 6 | ConversationController、MessageController、ConversationService、MessageService、WebSocketSessionManager 等，均已在 PR/7 squash merge 时合入 |
| 无关项目文件 | ~22 | `.hyperion/skills/iuap-c-*` 目录，非 AgentHub 项目代码 |
| **有用 — 新增** | **2** | RedisConfig.java、AgentCacheService.java |
| **有用 — 配置追加** | **2** | pom.xml: spring-boot-starter-data-redis；application.yml: Redis 连接池 |

### 2.2 倒退性改动示例

```diff
# PR/11 的 pom.xml 会删除这些已实现的依赖：
- spring-boot-starter-security   # JWT 认证
- jjwt-api / jjwt-impl           # Token 生成
- springdoc-openapi               # Swagger UI

# PR/11 的 application.yml 会删除：
- spring.config.import: optional:classpath:application-secret.yml
- jwt.secret / jwt.expiration
```

如果直接合并 PR/11，后端编译失败 + JWT 认证全部报废。

### 2.3 有用部分评估

**RedisConfig.java**：Lettuce 连接工厂 + RedisTemplate 序列化配置。代码结构清晰，直接可用。

**AgentCacheService.java**：Agent 元数据 Redis 缓存 CRUD。Key 命名规范 `agent:metadata:{id}` / `agent:list`，支持单条读写和批量写入。设计合理但不能单独工作——没有任何 Controller 调用它。

### 2.4 审查结论

> **不可直接合并**。仅提取 Redis 相关 2 个新文件 + 2 处配置追加。其余全部丢弃。

---

## 三、提取与接线

### 3.1 第一次提交（`6a4d23d`）

| 文件 | 操作 | 来源 |
|------|:--:|------|
| `RedisConfig.java` | 新建 | PR/11 |
| `AgentCacheService.java` | 新建 | PR/11 |
| `pom.xml` | +spring-boot-starter-data-redis | PR/11 提取 |
| `application.yml` | +Redis 连接池配置 | PR/11 提取 |

### 3.2 接线修复（`9d1c7ef`）

#### 问题 1：AgentCacheService 无人调用

`AgentCacheService` 定义了完整的缓存方法，但没有任何 Controller 或 Service 调用它。Redis 中 `agent:*` key 始终为空。

**修复**：在 `AgentController.getAgents()` 中注入 `AgentCacheService`，每次读取 DB 后同步调用 `saveAgentsBatch()` 写入 Redis。采用 read-through 模式——先读 DB，再写缓存，失败仅打 warn 不阻塞。

```java
// AgentController.java
private final AgentCacheService agentCacheService;

@GetMapping("/agents")
public ResponseEntity<?> list() {
    List<Agent> agentList = agentRepository.findAll();
    try {
        agentCacheService.saveAgentsBatch(agentList);
    } catch (Exception e) {
        log.warn("Agent 缓存写入 Redis 失败: {}", e.getMessage());
    }
    // ...
}
```

#### 问题 2：LocalDateTime 序列化失败

Redis 写入时 Jackson 报错：
```
Java 8 date/time type java.time.LocalDateTime not supported by default
```

根因分析链路：
1. `Agent` 实体含 `LocalDateTime createdAt/updatedAt` 字段
2. `RedisConfig` 使用 `RedisSerializer.json()` 序列化
3. `RedisSerializer.json()` 内部使用 Spring Boot 自动配置的 `ObjectMapper`
4. 但该 ObjectMapper **未注册** `JavaTimeModule`
5. 即使 pom.xml 添加了 `jackson-datatype-jsr310` 依赖，`RedisSerializer.json()` 也不会自动加载该模块

**修复**：

```java
// 旧：RedisSerializer.json() — 不支持 LocalDateTime
RedisSerializer<Object> jsonSerializer = RedisSerializer.json();

// 新：手动注册 JavaTimeModule
ObjectMapper objectMapper = new ObjectMapper();
objectMapper.registerModule(new JavaTimeModule());
GenericJackson2JsonRedisSerializer jsonSerializer = 
    new GenericJackson2JsonRedisSerializer(objectMapper);
```

同时追加 `jackson-datatype-jsr310` 依赖（Spring Boot 父 POM 已管理版本号）。

---

## 四、Redis Key 命名规范

通过本次集成确立了项目 Redis Key 规范：

```
{领域}:{类型}:{标识}
```

| Key 模式 | 示例 | 值类型 |
|----------|------|--------|
| `agent:metadata:{agentId}` | `agent:metadata:agent_claude_001` | JSON（Agent 对象） |
| `agent:list` | `agent:list` | JSON 数组（List\<Agent\>） |

未来扩展沿用此规范：
```
session:ws:{conversationId}     # WebSocket 会话状态
artifact:preview:{artId}        # 产物预览缓存  
user:token:{userId}             # Token 黑名单
```

---

## 五、测试验证

### 5.1 测试环境

- Redis: Memurai（Windows 原生 Redis 兼容版），端口 6379，无密码
- Spring Boot: 默认 H2 模式启动

### 5.2 测试结果

| # | 测试 | 预期 | 结果 |
|----|------|------|:--:|
| 1 | Maven 编译 | spring-boot-starter-data-redis + jackson-datatype-jsr310 正确拉取 | ✅ |
| 2 | 启动（Redis 未运行） | 惰性连接，不阻塞启动 | ✅ |
| 3 | 启动（Redis 运行中） | Lettuce 静默连接成功 | ✅ |
| 4 | GET /agents（JWT） | 已有功能不受影响 | ✅ |
| 5 | GET /agents → Redis 写入 | `agent:list` + `agent:metadata:*` 出现 | ✅ |
| 6 | `redis-cli keys "agent:*"` | 正确返回缓存 key | ✅ |

### 5.3 调试过程

`AgentCacheService` 写入失败时有清晰日志：

```
WARN Agent 缓存写入 Redis 失败: Could not write JSON: ...
```

从日志中定位到 `LocalDateTime` 序列化问题 → 发现 `RedisSerializer.json()` 不加载 `JavaTimeModule` → 改用 `GenericJackson2JsonRedisSerializer` + 显式注册模块。

---

## 六、代码质量评估

### RedisConfig.java — 修正后 ✅

- Lettuce 连接工厂（Spring Boot 自动配置优先，此处作为显式声明）
- Key 用 StringSerializer，Value 用 GenericJackson2JsonRedisSerializer
- `JavaTimeModule` 显式注册，避免 LocalDateTime 序列化失败
- `afterPropertiesSet()` 正确调用

### AgentCacheService.java — 设计合理 ✅

- `AGENT_METADATA_PREFIX` + `AGENT_LIST_KEY` 常量集中管理
- `saveAgentsBatch()` 使用 `multiSet()` 批量写入，效率优于逐个 `set()`
- `getAllAgents()` 有 `@SuppressWarnings("unchecked")` + `instanceof` 保护
- 未引入额外依赖——纯 Spring Data Redis

---

## 七、后续建议

1. **Agent 数据源切换**：P2 阶段可改为 `AgentCacheService.getAllAgents()` 优先读 Redis（miss 时回退 DB），减轻 H2 数据库压力
2. **Python 端消费**：Python Agent Service 可通过 Redis 直接读取 Agent 列表，替代当前 `AGENT_REGISTRY` 静态缓存，实现 Java DB → Redis → Python 的实时同步
3. **TTL 设置**：当前缓存永不过期，建议 Agent 更新时主动失效 + 设置合理的 TTL
4. **PR 创建规范培训**：建议 Liu-Weizhi1219 在创建 PR 前执行 `git fetch upstream dev && git merge upstream/dev`，避免基于旧版本开发

---

## 八、经验总结

### 8.1 部分提取优于整体合并

当 PR 包含倒退改动和无关文件时，`git cherry-pick --no-commit` + 手动筛选文件的模式优于直接 merge。本次仅提取了 2/34 的文件，其余全部丢弃。

### 8.2 序列化库的模块化陷阱

Jackson 的模块是插件式架构——添加 Maven 依赖不等于自动注册。`RedisSerializer.json()` 使用的 ObjectMapper 与 Spring Boot 自动配置的 ObjectMapper 是**不同实例**，需要显式 `registerModule(new JavaTimeModule())`。

### 8.3 基础设施先行的价值

RedisConfig 和 AgentCacheService 虽然当前只是基础设施（P2 才大规模使用），但提前集成有实际收益：
- 验证了 Redis 连接、序列化、读写全链路
- 确立了 Key 命名规范
- AgentController 接入后立即可见缓存效果

### 8.4 惰性连接的设计智慧

`LettuceConnectionFactory` 默认不在启动时校验连接，Redis 不可用时仅警告不崩溃。这种设计让 P2 基础设施可以提前合入而不影响日常开发（无需 Redis 也能正常启动）。
