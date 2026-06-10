# 开发过程记录 — 数据库持久化机制与 H2→PostgreSQL 迁移分析

> **参与 Agent**: Trae AI（Claude 后端）\
> **开发人员**: IAyousa\
> **日期**: 2026-05-28\
> **分支**: dev\
> **涉及模块**: backend-java（Spring Boot JPA 数据层）、agent-service（FastAPI 数据流）\
> **目的**: 理清 Java 端与 Python 端各自的数据读写链路，记录 H2 内存数据库迁移到 PostgreSQL 的完整方案

---

## 目录

1. [背景与触发](#1-背景与触发)
2. [Java 端数据持久化机制](#2-java-端数据持久化机制)
3. [Python 端数据持久化机制](#3-python-端数据持久化机制)
4. [完整数据流全景](#4-完整数据流全景)
5. [H2 → PostgreSQL 迁移方案](#5-h2--postgresql-迁移方案)
6. [JPA 兼容性分析](#6-jpa-兼容性分析)
7. [总结与对团队的影响](#7-总结与对团队的影响)

---

## 1. 背景与触发

### 1.1 触发方式

用户指令：

> "请你现在为我讲讲 Java 端和 Python 端是怎么读取到数据库中数据的，生成新的数据后怎么回写到数据库中，数据最后保存在哪里？"

> "那后续怎么把 H2 数据库迁移到 PostgreSQL 数据库呢"

### 1.2 问题背景

团队已经完成了 JPA 数据层搭建、CORS 配置和前端 Axios 实例配置，但尚未有人系统梳理过数据在三个子系统（前端、Java 后端、Python Agent 服务）之间的流转路径，也没有明确 H2 → PostgreSQL 的迁移策略。这两个问题对后续开发方向有直接影响。

---

## 2. Java 端数据持久化机制

### 2.1 当前数据库：H2 内存模式

数据实际存储在 **JVM 内存** 中，配置文件位于 `backend-java/src/main/resources/application.yml`：

```yaml
datasource:
  url: jdbc:h2:mem:agenthub;DB_CLOSE_DELAY=-1
  username: sa
  password: 
  driver-class-name: org.h2.Driver
```

`jdbc:h2:mem:agenthub` 中的 `mem` 表示纯内存模式，**不落盘**。`DB_CLOSE_DELAY=-1` 让数据库在最后一个连接断开后不立即销毁，只在 JVM 进程退出时才丢失。

> ⚠️ **影响**：Java 进程每次重启，所有会话记录和消息全部清空。文档中已规划 P1 阶段切换为 PostgreSQL。

### 2.2 读取链路

```
Java 代码调用 repository.findById("xxx")
    → Spring Data JPA 代理拦截
    → Hibernate (ORM) 生成 SQL
    → JDBC Driver 执行
    → H2 内存数据库返回结果
    → Hibernate 将 ResultSet 映射为 Java 实体对象
    → 返回 Agent/Message/Conversation/User 实体
```

4 个 Repository 接口（均继承 `JpaRepository`）：

| Repository | 自定义查询方法 |
|------------|---------------|
| AgentRepository | `findByType(type)`, `findByCreatedBy(userId)` |
| ConversationRepository | `findByCreatedByOrderByUpdatedAtDesc(userId)` |
| MessageRepository | `findByConversationIdOrderByCreatedAtAsc(convId)`, `findByConversationIdAndIsPinnedTrue(convId)` |
| UserRepository | `findByUsername(username)`, `existsByUsername(username)` |

Spring Data JPA 会根据方法名自动生成对应 SQL，无需手写任何实现代码。

### 2.3 写入链路

```
Java 代码调用 repository.save(entity)
    → JPA 判断是 INSERT 还是 UPDATE（根据主键是否存在）
    → Hibernate 生成 SQL
    → JDBC Driver 执行
    → H2 内存数据库存储
```

所有实体类通过 `@PrePersist` 钩子自动处理主键生成和时间戳：

```java
@PrePersist
public void prePersist() {
    if (this.id == null) {
        this.id = java.util.UUID.randomUUID().toString();  // UUID 主键
    }
    if (this.createdAt == null) {
        this.createdAt = LocalDateTime.now();               // 创建时间
    }
}
```

### 2.4 种子数据

`data.sql` 在每次应用启动时自动执行（`spring.sql.init.mode: always`），使用 H2 特有的 `MERGE INTO` 语法预填充两个 Agent：

```sql
MERGE INTO agents (id, name, type, avatar_url, system_prompt, capabilities, created_at)
KEY (id) VALUES (...)
```

### 2.5 制品文件存储

Agent 生成的 HTML/CSS/JS 产物不存数据库，而是存入本地文件系统：

```yaml
artifact:
  storage:
    path: ./artifacts
    url-prefix: /artifacts
```

### 2.6 当前实际状态

Repository 层已完备，但 `WebSocketController.handleUserMessage()` 中读写数据库的代码尚未实现（仅有注释占位），所以目前 Java 端 **有完整的读写能力但尚未有业务代码调用**。

---

## 3. Python 端数据持久化机制

### 3.1 结论：Python 端没有自己的数据库

当前 4 个 API 端点全部返回硬编码的 mock 数据：

| 端点 | 返回内容 | 文件 |
|------|---------|------|
| `GET /api/v1/agents` | `[{"id": "1", "name": "Mock Agent"}]` | `agents.py` |
| `GET /api/v1/conversations` | `[]` (空数组) | `conversations.py` |
| `GET /api/v1/artifacts` | `[]` (空数组) | `artifacts.py` |
| `POST /api/v1/messages/chat/stream` | 假的 SSE 流（`asyncio.sleep` 模拟逐字输出） | `messages.py` |

`requirements.txt` 中虽有 `sqlalchemy` 和 `aiosqlite`，但代码中**没有任何地方 import 它们**，纯粹是预留依赖。

### 3.2 Python 端的架构定位

Python Agent 服务在架构中是一个**无状态的中转节点**，不持有任何业务数据：

```
Java Backend → HTTP POST → Python Agent Service → HTTP POST → Claude/Codex API
Java Backend ← SSE Stream ← Python Agent Service ← SSE Stream ← LLM 响应
```

它的唯一职责是：
1. 接收 Java 后端发来的用户消息
2. 根据 Agent 类型选择对应的 LLM Adapter
3. 调用外部 LLM API，获取流式响应
4. 将响应实时逐块转发回 Java 后端

所有持久化工作由 Java 端完成，Python 端不存储任何数据。

---

## 4. 完整数据流全景

```
┌──────────────┐   WebSocket (STOMP)    ┌──────────────────┐   HTTP + SSE    ┌──────────────────┐   HTTP    ┌─────────────┐
│  Vue 前端      │ ◄───────────────────► │  Spring Boot      │ ◄──────────────► │  FastAPI Agent    │ ◄───────► │ Claude API  │
│  (Port 5173)  │                       │  (Port 8080)      │                  │  (Port 8000)      │          │ Codex API   │
└──────────────┘                       │                   │                  │                   │          └─────────────┘
                                       │  ┌─────────────┐  │                  │  (无数据库)        │
                                       │  │ H2 内存数据库 │  │                  │                   │
                                       │  │  (5 张表)     │  │                  └──────────────────┘
                                       │  └─────────────┘  │
                                       │  ┌─────────────┐  │
                                       │  │ ./artifacts/ │  │  ← 制品文件存本地文件系统
                                       │  └─────────────┘  │
                                       └──────────────────┘
```

### 4.1 消息流转时序（设计意图）

| 步骤 | 动作 | 方向 |
|------|------|------|
| 1 | 用户在前端发送消息 | Vue → Java (STOMP WebSocket) |
| 2 | Java 将用户消息写入 H2 `messages` 表 | Java → H2 (JPA save) |
| 3 | Java 调用 Agent 服务 | Java → Python (HTTP POST) |
| 4 | Python 选择合适的 Adapter 调用 LLM | Python → Claude/Codex (HTTP) |
| 5 | Python 将 LLM 流式响应转发回 Java | Python → Java (SSE) |
| 6 | Java 将 Agent 回复写入 H2 `messages` 表 | Java → H2 (JPA save) |
| 7 | Java 将 Agent 回复推送给前端 | Java → Vue (STOMP WebSocket) |
| 8 | 如果 LLM 生成了制品，Java 将其写入 `./artifacts/` | Java → 本地文件系统 |

> ⚠️ **当前状态**：步骤 2-8 均未实现，仅完成了各层的基础设施搭建。

---

## 5. H2 → PostgreSQL 迁移方案

### 5.1 迁移复杂度评估

**结论：低风险、低工作量**。因为 JPA 抽象层隔离了数据库差异，绝大部分代码无需改动。

| 组件 | 是否需改 | 原因 |
|------|---------|------|
| 4 个 JPA Entity 类 | ❌ 不改 | 全是 JPA 标准注解，无 H2 特有语法 |
| 4 个 JPA Repository 接口 | ❌ 不改 | 只依赖 `JpaRepository` 标准接口 |
| AgentGatewayService | ❌ 不改 | 用 WebClient 调 Python，与数据库无关 |
| WebSocketController | ❌ 不改 | 消息收发不直接依赖数据库方言 |
| pom.xml | ❌ 不改 | `postgresql` 依赖已存在 |
| Python Agent 服务 | ❌ 不改 | Python 端无数据库 |
| application.yml | ✅ **需改 5 行** | datasource + dialect 配置 |
| data.sql | ✅ **需改语法** | H2 `MERGE INTO` → PostgreSQL `INSERT ON CONFLICT` |

### 5.2 迁移步骤

#### Step 1：启动 PostgreSQL Docker 容器

将文档中的 Docker Compose 配置创建为项目根目录的 `docker-compose.yml`：

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    container_name: agenthub-postgres
    environment:
      POSTGRES_DB: agenthub
      POSTGRES_USER: agenthub
      POSTGRES_PASSWORD: agenthub123
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped
volumes:
  postgres_data:
    driver: local
```

执行 `docker-compose up -d postgres` 启动。

#### Step 2：修改 application.yml

| 配置项 | H2（旧） | PostgreSQL（新） |
|--------|----------|-----------------|
| `spring.h2.console.enabled` | `true` | 删除整个 h2 块 |
| `datasource.url` | `jdbc:h2:mem:agenthub;DB_CLOSE_DELAY=-1` | `jdbc:postgresql://localhost:5432/agenthub` |
| `datasource.username` | `sa` | `agenthub` |
| `datasource.password` | （空） | `agenthub123` |
| `driver-class-name` | `org.h2.Driver` | `org.postgresql.Driver` |
| `hibernate.dialect` | `org.hibernate.dialect.H2Dialect` | `org.hibernate.dialect.PostgreSQLDialect` |

#### Step 3：修改 data.sql

将 H2 特有的 `MERGE INTO` 改为 PostgreSQL 兼容的 `INSERT ... ON CONFLICT`：

```sql
-- 旧语法（H2）
MERGE INTO agents (...) KEY (id) VALUES (...);

-- 新语法（PostgreSQL）
INSERT INTO agents (id, name, type, avatar_url, system_prompt, capabilities, created_at)
VALUES (...)
ON CONFLICT (id) DO NOTHING;
```

#### Step 4：验证

- 重启 Spring Boot，观察控制台日志出现 PostgreSQL 建表语句
- 确认 `http://localhost:8080/h2-console` 已不可访问（正常现象）
- 使用 DBeaver/pgAdmin 连接 `localhost:5432` 查看 `agenthub` 数据库中的 5 张表

### 5.3 关于数据迁移

MVP 阶段 H2 是内存模式，**每次重启数据自动清空**，因此不存在数据迁移需求。如果未来需要保留 H2 数据，可以采用：
- **方案 A**：导出 H2 SQL → 导入 PostgreSQL（适用于开发阶段积累的测试数据）
- **方案 B**：直接丢弃 H2 数据，从 seed data 重新初始化（适用于生产数据尚未产生的 MVP 阶段）

---

## 6. JPA 兼容性分析

### 6.1 为什么 JPA 代码不需要改

JPA（Java Persistence API）是一套标准的 ORM 接口规范。当前代码只用到了 JPA 标准注解和标准 API：

| 使用的特性 | 标准来源 | H2 兼容 | PostgreSQL 兼容 |
|-----------|---------|---------|----------------|
| `@Entity`, `@Table`, `@Column`, `@Id` | JPA 2.2+ | ✅ | ✅ |
| `@ManyToMany` + `@JoinTable` | JPA 2.2+ | ✅ | ✅ |
| `@PrePersist`, `@PreUpdate` | JPA 2.2+ | ✅ | ✅ |
| `JpaRepository.save()`, `.findById()` | Spring Data JPA | ✅ | ✅ |
| `@Index` 注解 | JPA 2.1+ | ✅ | ✅ |
| `@Column(columnDefinition = "TEXT")` | JPA | ✅ | ✅ |

唯一依赖数据库方言的是 `application.yml` 中的 `hibernate.dialect` 和 JDBC driver——两者都通过配置切换，不涉及代码改动。

### 6.2 潜在风险点：Lombok @Data + JPA 代理

`Conversation.java` 的 `@ManyToMany` 关联使用了 `FetchType.LAZY`（延迟加载）。Lombok 的 `@Data` 自动生成的 `equals()` 和 `hashCode()` 在 Hibernate 代理对象上可能触发意外行为。切换到 PostgreSQL 后延迟加载行为不变，这个问题两种数据库都存在，属于 JPA 通用问题，与数据库迁移无关。

---

## 7. 总结与对团队的影响

### 7.1 关键结论

| 问题 | 答案 |
|------|------|
| Java 端数据存在哪？ | 当前：H2 内存数据库（JVM 进程中）；P1 后：PostgreSQL（Docker 卷持久化） |
| Python 端数据存在哪？ | 不存数据。Python 是无状态中转，只做 LLM 调度和流式转发 |
| 数据怎么读？ | `JpaRepository` 方法 → Hibernate → JDBC → 数据库 |
| 数据怎么写？ | `JpaRepository.save()` → Hibernate → JDBC → 数据库 |
| 迁移要改多少代码？ | **改 1 个 YAML 文件（5 行）+ 1 个 SQL 文件（2 条语句）**，Java/Python 代码零改动 |

### 7.2 对团队各成员的影响

| 成员 | 影响 |
|------|------|
| **成员 A**（后端 Service/Controller） | 写业务代码时直接 `@Autowired` Repository 即可，无需关心底层是 H2 还是 PostgreSQL |
| **成员 B**（后端基础设施） | P1 阶段负责执行迁移，参照本文档 Step 1-4 |
| **成员 C**（前端） | 无影响。前端只通过 REST/WebSocket 与 Java 通信，数据库对前端完全透明 |
| **成员 D**（Python Agent） | 无影响。Python 端不需要也不应该直连数据库，一切数据通过 Java 中转 |

### 7.3 一个容易混淆的点

`requirements.txt` 中有 `sqlalchemy` 和 `aiosqlite`，但这是**预留依赖**，当前代码完全未使用。Python Agent 服务的架构定位是**无状态网关**，不应该直连数据库。如果未来有需求（如缓存 Agent 配置），也应该由 Java 后端统一管理数据，Python 端通过 HTTP API 从 Java 获取。

---

> **文档关联**：
> - 本记录的分析结论已同步更新至 `docs/基础设施配置.md`（补充完整迁移步骤）和 `docs/数据模型设计.md`（补充 SQL 兼容性说明）。
