# AgentHub 开发过程记录 — JPA 数据层搭建

> **参与 Agent**: Trae AI（Claude 后端）\
> **开发人员**: IAyousa\
> **日期**: 2026-05-26\
> **分支**: dev\
> **涉及模块**: backend-java（Spring Boot JPA 数据层）\
> **目的**: 记录一次完整的 AI 辅助开发过程，帮助团队理解如何与 AI Agent 高效协作

---

## 目录

1. [整体流程概览](#1-整体流程概览)
2. [阶段一：项目状态评估与任务规划](#2-阶段一项目状态评估与任务规划)
3. [阶段二：Git 分支管理与冲突解决](#3-阶段二git-分支管理与冲突解决)
4. [阶段三：JPA Entity 层设计与实现](#4-阶段三jpa-entity-层设计与实现)
5. [阶段四：JPA Repository 层实现](#5-阶段四jpa-repository-层实现)
6. [阶段五：数据初始化方案](#6-阶段五数据初始化方案)
7. [阶段六：编译启动与测试验证](#7-阶段六编译启动与测试验证)
8. [阶段七：中文编码问题排查与修复](#8-阶段七中文编码问题排查与修复)
9. [与 AI Agent 协作的方法论总结](#9-与-ai-agent-协作的方法论总结)

---

## 1. 整体流程概览

本次开发任务的起点是 CLAUDE.md 项目上下文文件，终点是验证 JPA 数据层 + 种子数据中文显示正常。整个过程遵循了标准的后端开发流程：

```
读取项目上下文 ──→ 评估当前状态 ──→ 制定推进计划
                                          │
                                    ┌─────┘
                                    ▼
                            创建 Entity 层（4 个实体）
                                    │
                                    ▼
                            创建 Repository 层（4 个接口）
                                    │
                                    ▼
                            编写种子数据（data.sql）
                                    │
                                    ▼
                            编译验证 → 启动测试 → 发现乱码
                                    │
                                    ▼
                            排查编码问题 → 尝试修复 → 失败重试
                                    │
                                    ▼
                            最终方案 → 验证通过 ✅
```

**关键数据**：

| 指标 | 数值 |
|------|------|
| 对话轮次 | ~10 轮 |
| 新建文件 | 10 个（4 Entity + 4 Repository + 1 data.sql + 1 文档） |
| 修改文件 | 1 个（application.yml） |
| 删除文件 | 0 个（删除后恢复不算） |
| 环境问题排查 | 2 次（CHARACTER_SET 兼容性、sql.init.encoding 配置） |

---

## 2. 阶段一：项目状态评估与任务规划

### 2.1 触发方式

用户指令：

> "请你读取 CLAUDE.md，说一下下一步要从哪里开始"

### 2.2 AI 的处理方式

AI 首先读取了项目上下文文件 CLAUDE.md，该文件包含：

```
├── 项目身份（名称、概念、团队、仓库信息）
├── 3 层架构图（Frontend ↔ Backend ↔ Agent Service）
├── 实际代码目录结构
├── 各模块完成度（前端 60% / 后端 15% / Agent 25%）
└── 关键技术决策
```

然后 AI 基于完成度数据做出了**优先级判断**：

```
前端 60% ── UI 原型可用，可暂时延后
后端 15% ── 最大短板，无 Entity/Repository/Service/Controller
Agent 25% ── 依赖后端网关服务，必须先完成后端

结论：优先打通 Spring Boot JPA 数据层
```

### 2.3 推荐的分阶段计划

AI 给出了 4 个阶段的推进路线：

1. **JPA 数据层** — Entity + Repository（本次完成）
2. **Service 层** — 封装业务逻辑
3. **REST API 层** — Controller + 前后端联调
4. **Agent 集成** — 对接 FastAPI + LLM

### 2.4 协作要点

- **提供上下文**：CLAUDE.md 让 AI 无需额外搜索就能理解项目全貌
- **分阶段推进**：不要一次指令包含太多任务，每个阶段完成后验证再继续
- **AI 擅长做状态评估**：把客观信息（完成度百分比、缺失项清单）交给 AI 分析，它能给出合理的优先级建议

---

## 3. 阶段二：Git 分支管理与冲突解决

### 3.1 场景

用户指令：

> "现在我的本地 git 为什么会有两个提交"

问题背景：本地分支和远程分支各自独立创建了 CLAUDE.md，导致分叉：

```
* a13ec2d (共同起点)
├── ec549ba (本地 CLAUDE.md 提交)
└── 2e94af7 (远程 CLAUDE.md 提交) — origin/dev 中也有人提交了同文件
```

### 3.2 AI 的处理方式

AI 执行了以下命令来诊断：

```bash
git status                          # 发现 unmerged paths
git log --oneline --graph --all -10 # 可视化分支分叉
git ls-files -u                     # 列出冲突状态文件（stage 2 vs stage 3）
git show :2:CLAUDE.md               # 读取本地版本
git show :3:CLAUDE.md               # 读取远程版本
```

然后对比两个版本，发现远程版多了 3 处改进（Agent Service 进度 25%、health 端点、requirements.txt 描述），工作区文件恰好已包含全部改进，直接 `git add` 标记解决。

### 3.3 协作要点

- **优先用命令而非 GUI**：`git show :2:file` / `git show :3:file` 能精确对比两个冲突版本
- **让 AI 帮你分析差异**：AI 可以同时读取两个版本并总结差异点，比人工逐行对比高效
- **IDE 自带 Git 的三种合并方式**：内联合并按钮、三向合并编辑器、Source Control 面板右键菜单

---

## 4. 阶段三：JPA Entity 层设计与实现

### 4.1 场景

用户指令：

> "现在请你协助完成第一阶段目标，以数据模型设计.md 为基础，生成 JPA 数据层"

### 4.2 AI 的处理方式

AI 首先读取了现有项目结构和数据模型设计文档：

```bash
# 读取的文件
- backend-java/pom.xml                          # 确认已有 JPA、Lombok、H2 依赖
- backend-java/src/main/resources/application.yml # 确认 H2 配置
- backend-java/src/main/java/com/agenthub/*      # 确认已有代码结构
- docs/数据模型设计.md                              # 获取 ER 图和表定义
```

然后基于文档中的表定义创建了 4 个 JPA Entity：

| Entity | 对应表 | 关联关系 |
|--------|--------|----------|
| `User.java` | `users` | — |
| `Conversation.java` | `conversations` | `@ManyToMany` → `agents` |
| `Message.java` | `messages` | 含 `idx_messages_conversation_id` 和 `idx_messages_pinned` 索引 |
| `Agent.java` | `agents` | — |

**关键设计决策**：

```java
// UUID 主键 + @PrePersist 自动生成（避免自增 ID 的安全问题）
@Id
@Column(length = 36)
private String id;

@PrePersist
public void prePersist() {
    if (this.id == null) {
        this.id = java.util.UUID.randomUUID().toString();
    }
}

// Lombok @Data 替代手写 getter/setter（减少样板代码）
@Data
@Entity
@Table(name = "agents")
public class Agent { ... }

// 多对多关系维护
@ManyToMany(fetch = FetchType.LAZY)
@JoinTable(
    name = "conv_agents",
    joinColumns = @JoinColumn(name = "conversation_id"),
    inverseJoinColumns = @JoinColumn(name = "agent_id")
)
private Set<Agent> agents = new HashSet<>();
```

### 4.3 协作要点

- **让 AI 阅读设计文档**：将数据模型设计.md 作为输入，AI 能直接将其转换为代码
- **AI 会遵循现有代码风格**：使用 Lombok `@Data`、已有包名 `com.agenthub.model`、Jakarta 注解等
- **简化设计文档中的代码骨架**：文档中的手写 getter/setter 被替换为 `@Data`，提高了代码简洁性

---

## 5. 阶段四：JPA Repository 层实现

### 5.1 创建内容

4 个 Repository 接口，扩展 `JpaRepository<Entity, String>`：

```java
// UserRepository — 按用户名查询（登录验证用）
Optional<User> findByUsername(String username);
boolean existsByUsername(String username);

// ConversationRepository — 按创建者查询，支持归档过滤和时间排序
List<Conversation> findByCreatedByOrderByUpdatedAtDesc(String createdBy);
List<Conversation> findByCreatedByAndIsArchivedOrderByUpdatedAtDesc(String createdBy, Boolean isArchived);

// MessageRepository — 按会话查询消息，支持固定消息过滤
List<Message> findByConversationIdOrderByCreatedAtAsc(String conversationId);
List<Message> findByConversationIdAndIsPinnedTrue(String conversationId);

// AgentRepository — 按类型/创建者查询
List<Agent> findByType(String type);
List<Agent> findByCreatedBy(String createdBy);
```

### 5.2 AI 讲解的交互流程

AI 在此阶段讲解了 **Spring Boot 标准四层交互模型**：

```
HTTP 请求 → Controller（接收请求）
                ↓ 调用
            Service（业务逻辑）
                ↓ 调用
            Repository（数据访问，JPA 接口）
                ↓ 操作
            Entity（映射到数据库表）
                ↓
            H2 数据库
```

---

## 6. 阶段五：数据初始化方案

### 6.1 初始方案：data.sql

最初使用 `data.sql` 文件 + `MERGE INTO` 语句预置 Agent 种子数据：

```sql
MERGE INTO agents (id, name, type, system_prompt, capabilities, ...)
KEY (id) VALUES
('agent_claude_001', 'Claude Code', 'claude_code',
 '你是一个经验丰富的软件工程师，擅长前端开发、代码审查和问题调试。请用中文回复。',
 '["代码生成", "代码审查", "Debug", "重构建议", "文档编写"]', CURRENT_TIMESTAMP);
```

配置文件需要两项配合：

```yaml
spring:
  sql:
    init:
      mode: always               # 每次启动都执行
      encoding: UTF-8            # 指定 data.sql 文件编码 ← 关键！
  jpa:
    defer-datasource-initialization: true  # 等 Hibernate 建完表再跑
```

**`defer-datasource-initialization: true` 的作用**：
- Spring Boot 默认先执行 `data.sql`，再执行 Hibernate DDL（建表）
- 如果颠倒顺序，`data.sql` 中的 INSERT/MERGE 会因表不存在而失败
- `defer-datasource-initialization: true` 强制 Hibernate 先建表，再执行 `data.sql`

### 6.2 中间偏差：DataInitializer.java

问题排查过程中，AI 曾用 `CommandLineRunner` 替代 `data.sql`（[DataInitializer.java](#) 已被删除）：

```java
@Component
public class DataInitializer implements CommandLineRunner {
    @Override
    public void run(String... args) {
        if (agentRepository.count() > 0) return;
        // 用 Java 代码插入数据...
    }
}
```

**为什么被否决**：
- SQL 脚本更利于迁移（可直接在 PostgreSQL/MySQL 上执行）
- DBA 可直接审阅 SQL，不需要改代码
- 符合业界标准实践

### 6.3 最终方案确认

用户表达了正确的工程判断后，AI 立即回归 SQL 方案并保留了核心修复（`sql.init.encoding: UTF-8`）。

---

## 7. 阶段六：编译启动与测试验证

### 7.1 验证步骤

```bash
# 第一步：编译验证
cd backend-java && mvn compile -q
# 输出：BUILD SUCCESS（无输出表示编译通过）

# 第二步：启动应用
mvn spring-boot:run -q
```

### 7.2 启动日志关键信息

```
-- 第 1 步：Repository 扫描（Spring Data JPA 自动发现接口）
Found 4 JPA repository interfaces.

-- 第 2 步：Hibernate 自动建表（ddl-auto: update）
Hibernate: create table agents (...)
Hibernate: create table conv_agents (...)    -- 关联表，联合主键
Hibernate: create table conversations (...)
Hibernate: create table messages (...)       -- 含索引
Hibernate: create table users (...)

-- 第 3 步：创建索引
Hibernate: create index idx_messages_conversation_id on messages (conversation_id)
Hibernate: create index idx_messages_pinned on messages (conversation_id, is_pinned)

-- 第 4 步：外键约束
Hibernate: alter table conv_agents add constraint FK... foreign key (agent_id) references agents

-- 第 5 步：data.sql 执行（默认 DEBUG 级别，需要调整日志级别才会显示）

-- 第 6 步：应用就绪
Started AgenthubApplication in 7.105 seconds
```

### 7.3 H2 控制台验证

浏览器访问 `http://localhost:8080/h2-console`：

| 字段 | 值 |
|------|-----|
| JDBC URL | `jdbc:h2:mem:agenthub` |
| User Name | `sa` |
| Password | (空) |

验证 SQL：

```sql
SHOW TABLES;           -- 应显示 5 张表
SELECT * FROM agents;  -- 应显示 2 条预置 Agent 数据
```

---

## 8. 阶段七：中文编码问题排查与修复

### 8.1 问题发现

用户反馈：

> "AGENTS 表里的 CAPABILITIES 字段和 SYSTEM_PROMPT 字段的中文显示乱码"

### 8.2 排查过程（体现了 AI Agent 协作中的典型试错流程）

#### 尝试 1：在 JDBC URL 加 `CHARACTER_SET=UTF-8`（失败）

```yaml
# 修改前
url: jdbc:h2:mem:agenthub;DB_CLOSE_DELAY=-1

# 修改后（错误）
url: jdbc:h2:mem:agenthub;DB_CLOSE_DELAY=-1;CHARACTER_SET=UTF-8
```

**结果**：H2 v2.x 不支持 `CHARACTER_SET` 参数，启动报错：
```
JdbcSQLNonTransientConnectionException: Unsupported connection setting "CHARACTER_SET"
```

**教训**：不同版本 H2 的 JDBC 参数不通用，应优先考虑 Spring Boot 层面的编码控制。

#### 尝试 2：同时回滚 URL + 删除 data.sql + 用 Java 代码初始化

**结果**：数据可正常写入（Java 字符串天然 Unicode），但失去了 SQL 脚本的迁移便利性。

#### 尝试 3（正确方案）：仅添加 `sql.init.encoding: UTF-8`

```yaml
# 最终 application.yml 中的关键配置
spring:
  sql:
    init:
      mode: always
      encoding: UTF-8            # ← 告诉 Spring 用 UTF-8 读 data.sql
```

### 8.3 根因分析

```
┌─────────────────────────────────────────────────────────┐
│  data.sql 文件的物理编码                                  │
│  ┌──────────────────────────────────────┐                │
│  │ 你是一个经验丰富的软件工程师...          │ ← UTF-8 字节  │
│  └──────────────────────────────────────┘                │
│                         ↓                                │
│  Spring Boot 读取文件时使用的编码                          │
│  ┌──────────────────────────────────────┐                │
│  │ 默认使用 Charset.defaultCharset()     │ ← 中文 Win GBK │
│  │ → 用 GBK 解析 UTF-8 文件 → 乱码      │                │
│  └──────────────────────────────────────┘                │
│                         ↓                                │
│  写入 H2 数据库 → 数据已经是乱码了                         │
└─────────────────────────────────────────────────────────┘

修复后：
┌─────────────────────────────────────────────────────────┐
│  sql.init.encoding: UTF-8                               │
│  → 用 UTF-8 解析 UTF-8 文件 → 正确中文 → 正常写入 H2     │
└─────────────────────────────────────────────────────────┘
```

### 8.4 用户提出的正确工程判断

> "是不是原来使用 SQL 脚本的方式加载数据库比较方便迁移，现在使用 Java 编码的方式得经过编译后才能加载数据"

AI 立即认同了用户的判断，这是本次协作中 **人机互补** 的典型案例：
- AI 擅长技术实现和调试
- 开发者保留对工程实践的判断权
- 好的协作结果 = AI 的能力 + 开发者的工程判断

---

## 9. 与 AI Agent 协作的方法论总结

### 9.1 有效协作的核心原则

| 原则 | 说明 | 本次实践体现 |
|------|------|-------------|
| **提供充分上下文** | 让 AI 读取 CLAUDE.md、设计文档等 | 阶段一直接基于项目状态评估任务优先级 |
| **分阶段推进** | 一个指令一个目标，完成验证再继续 | Entity → Repository → data.sql → 测试 → 修 bug |
| **保留判断权** | AI 的方案可以质疑和修正 | 用户否决 Java 初始化方案，回归 SQL |
| **即时验证** | 每步完成后编译/运行 | 编译 → 启动 → H2 控制台验证 |
| **记录过程** | 通过对话自然形成文档 | 即本文档的产生过程 |

### 9.2 高效指令模式

```markdown
# ✅ 好的指令（具体、有上下文、有明确目标）
"以 `docs/数据模型设计.md` 为基础，生成 JPA 数据层，
 并讲解后端如何与这些实体交互"

# ✅ 好的指令（发现问题时描述现象）
"AGENTS 表的 CAPABILITIES 和 SYSTEM_PROMPT 字段中文显示乱码，
 这是正常现象吗"

# ❌ 差的指令（过于模糊）
"帮我写后端代码"

# ❌ 差的指令（一次要求太多）
"帮我把整个后端从 Entity 到 Controller 全部写完"
```

### 9.3 为 AI 准备项目上下文

推荐在项目根目录维护类似 CLAUDE.md 的上下文文件，包含：

```
1. 项目身份（名称、技术栈、团队背景）
2. 架构图（层级关系、端口、通信协议）
3. 当前目录结构（实际代码，非规划）
4. 各模块完成度（百分比 + 缺失项清单）
5. 关键技术决策（MVP 简化选项、已推迟功能）
```

这样每次新对话时，AI 无需重新探索代码库就能立即进入工作状态。

### 9.4 常见协作模式

```
模式 1：诊断型
  用户描述现象 → AI 分析根因 → AI 提出修复 → 用户确认 → 实施

模式 2：实现型
  用户提供设计文档 → AI 生成代码 → 编译验证 → 运行测试

模式 3：教学型
  用户提问原理 → AI 讲解 + 代码示例 → 用户追问细节 → 深入解释

模式 4：优化型
  用户提出改进建议 → AI 评估合理性 → 调整方案 → 重新验证
```

本次开发过程覆盖了以上全部 4 种模式。

---

## 附录 A：本次开发的最终文件清单

```
backend-java/src/main/java/com/agenthub/
├── model/
│   ├── User.java                       # 新增
│   ├── Conversation.java               # 新增
│   ├── Message.java                    # 新增
│   └── Agent.java                      # 新增
├── repository/
│   ├── UserRepository.java             # 新增
│   ├── ConversationRepository.java     # 新增
│   ├── MessageRepository.java          # 新增
│   └── AgentRepository.java            # 新增
├── config/
│   ├── WebSocketConfig.java            # 已有
│   └── (DataInitializer.java)          # 已删除
├── controller/
│   └── WebSocketController.java        # 已有
├── dto/
│   └── SendMessageRequest.java         # 已有
└── service/
    └── AgentGatewayService.java        # 已有

backend-java/src/main/resources/
├── application.yml                     # 已修改（添加 sql.init.encoding）
└── data.sql                            # 新增（预置 Agent 种子数据）

docs/
└── 开发过程记录-JPA数据层搭建.md         # 新增（本文档）
```

## 附录 B：application.yml 最终完整配置

```yaml
spring:
  application:
    name: agenthub
  h2:
    console:
      enabled: true
      path: /h2-console
      settings:
        trace: false
        web-allow-others: false
  datasource:
    url: jdbc:h2:mem:agenthub;DB_CLOSE_DELAY=-1
    username: sa
    password: 
    driver-class-name: org.h2.Driver
  sql:
    init:
      mode: always
      encoding: UTF-8
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    defer-datasource-initialization: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.H2Dialect
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 10MB

server:
  port: 8080

agent:
  service:
    url: http://localhost:8000
    timeout: 120000

artifact:
  storage:
    path: ./artifacts
    url-prefix: /artifacts
```
