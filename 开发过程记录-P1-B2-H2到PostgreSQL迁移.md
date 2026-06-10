# 开发过程记录 — P1-B2 H2 → PostgreSQL 迁移

> **日期**: 2026-06-07 ~ 2026-06-08
> **任务编号**: P1-B2
> **所属模块**: backend-java / 基础设施
> **前置依赖**: P0 全部完成 ✅（5 个 Controller + Service + DTO 全部就绪）

---

## 一、背景

### 1.1 为什么要做

MVP 阶段使用 H2 文件模式数据库（零部署成本），但 H2 本质是嵌入式测试数据库，不适合生产环境：

| | H2 | PostgreSQL |
|---|---|---|
| 运行方式 | 嵌入 Java 进程 | 独立服务器进程 |
| 并发 | 文件锁，高并发受限 | MVCC，高并发读写 |
| 适用场景 | 开发/测试 | 生产环境 |
| 生态 | Java 专属 | 所有语言/框架 |

P1 阶段后续要接入 Redis 共享缓存（P1-A1）、实现 JWT 用户认证（P1-B1），H2 无法支撑多服务共享访问。

### 1.2 迁移策略

Spring Data JPA 通过 JDBC 标准接口 + Hibernate Dialect 自动适配底层数据库。迁移只需改 yml 配置 + SQL 方言，**Java 和 Python 代码零改动**。

依据文档：`docs/基础设施配置.md` 第 2.1.1 节迁移指南、`docs/团队任务分配.md` P1-B2 任务描述。

---

## 二、任务执行过程

### 2.1 创建 docker-compose.yml

**操作内容**：项目根目录新建 `docker-compose.yml`，配置 PostgreSQL 15-alpine 容器。

```yaml
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

**设计决策**：

| 决策项 | 选择 | 理由 |
|--------|------|------|
| PG 版本 | 15-alpine | Alpine 镜像体积小（~80MB），15 是稳定版本 |
| 数据持久化 | named volume `postgres_data` | 容器删除后数据不丢失 |
| `version` 字段 | 删除 | Docker Compose v2 起 version 已废弃，当前 Docker 29.x 会警告 |

### 2.2 切换 application.yml 数据源

**操作内容**：修改 `backend-java/src/main/resources/application.yml`，涉及以下变更：

| 配置项 | H2（旧） | PostgreSQL（新） |
|--------|----------|-------------------|
| `h2.console.*` | `enabled: true` | **整块删除** |
| `datasource.url` | `jdbc:h2:file:./shared-data/agenthub;AUTO_SERVER=TRUE` | `jdbc:postgresql://localhost:5432/agenthub?charSet=UTF-8` |
| `datasource.username` | `sa` | `agenthub` |
| `datasource.password` | （空） | `agenthub123` |
| `driver-class-name` | `org.h2.Driver` | `org.postgresql.Driver` |
| `hibernate.dialect` | `org.hibernate.dialect.H2Dialect` | **删除**（Hibernate 6 自动检测） |

**错误与修复**：

| # | 错误现象 | 根因 | 修复 |
|---|---------|------|------|
| 1 | Docker 启动时 `version is obsolete` 警告 | `version: '3.8'` 在新版 Docker 已废弃 | 删除 `version` 字段 |
| 2 | Spring Boot 启动报 `HHH90000025` 警告 | 显式指定 `PostgreSQLDialect`，Hibernate 6 已自动检测 | 删除 `dialect` 配置项 |
| 3 | 中文存到 PG 变成 `PG????` | JDBC 连接未指定字符编码 | URL 加 `?charSet=UTF-8` |
| 4 | Docker 拉镜像失败 `dial tcp 103.39.76.66:443` | 国内网络无法直连 Docker Hub | 配置 `registry-mirrors`（19 个国内镜像源） |

### 2.3 转换 data.sql 语法

**操作内容**：将 H2 特有的 `MERGE INTO ... KEY (id)` 改为 PostgreSQL 兼容的 `INSERT ... ON CONFLICT (id) DO NOTHING`。

```sql
-- H2（旧）
MERGE INTO agents (id, name, type, ...) KEY (id) VALUES (...);

-- PostgreSQL（新）
INSERT INTO agents (id, name, type, ...) VALUES (...)
ON CONFLICT (id) DO NOTHING;
```

**设计决策**：`ON CONFLICT DO NOTHING` 保证幂等性——每次 Spring Boot 重启时 data.sql 都会执行（`sql.init.mode: always`），但已存在的 Agent 记录不会被重复插入。

### 2.4 Docker 环境配置（本机适配）

**操作内容**：本机 Docker Desktop 29.3.0 配置 19 个国内镜像加速器解决 Docker Hub 拉取失败。

```json
{
  "builder": { "gc": { "defaultKeepStorage": "20GB", "enabled": true } },
  "experimental": false,
  "registry-mirrors": [
    "https://docker.1panelproxy.com",
    "https://docker.m.daocloud.io",
    "..."
  ]
}
```

**错误与修复**：

| # | 错误现象 | 根因 | 修复 |
|---|---------|------|------|
| 5 | `Access is denied` 修改 Disk image location | Docker 无权限写 D 盘 | 保持默认路径 |
| 6 | `Expected ',' or '}' after property value` | `"experimental": false` 后面缺少逗号 | 补上 `,` |
| 7 | docker-compose 头部多了 `{ {` | 粘贴错误 | 改为单个 `{` |

---

## 三、技术要点讲解

### 3.1 为什么 Java 代码不用改一行

**JDBC 抽象层**：

```
Java 代码 → JDBC 接口 (javax.sql.DataSource)
  → JDBC Driver (H2 Driver 或 PostgreSQL Driver)
    → 数据库
```

JDBC 是 Java 操作数据库的统一标准接口。换数据库只需换 Driver jar 包 + 改连接字符串，Java 代码只跟 `DataSource`/`Connection` 打交道，不感知底层是什么数据库。

**Spring Data JPA 方法名推导 SQL**：

```java
// MessageRepository.java
List<Message> findByConversationIdOrderByCreatedAtDesc(String id, Pageable pageable);
```

Spring Data JPA 根据方法名推导出：
- `findByConversationId` → `WHERE conversation_id = ?`
- `OrderByCreatedAtDesc` → `ORDER BY created_at DESC`

Hibernate 根据配置的 `hibernate.dialect`（或自动检测）生成对应数据库的 SQL。H2 和 PG 的标准 CRUD SQL 几乎一样，LIMIT/OFFSET 分页语法也相同。这就是为什么**零代码迁移**可行。

**data.sql 为什么必须手动改**：

`data.sql` 不走 JPA，是 Spring Boot 启动时通过 `DataSourceInitializer` 直接执行的原生 SQL。不经过 Hibernate 方言转换，因此 H2 特有的 `MERGE INTO` 语法不会自动转成 PG 的 `ON CONFLICT`。

### 3.2 PostgreSQL 连接字符串解读

```
jdbc:postgresql://localhost:5432/agenthub?charSet=UTF-8
│                  │         │    │       │
│                  │         │    │       └─ 数据库名
│                  │         │    └─ 端口（PG 默认 5432，类比 MySQL 3306）
│                  │         └─ 主机地址
│                  └─ PG JDBC 协议
└─ JDBC 协议头
```

`?charSet=UTF-8` 确保客户端和服务器之间以 UTF-8 传输中文字符。不像 H2 是进程内调用自动继承 JVM 编码，PG 通过 TCP 连接，必须显式指定。

### 3.3 ON CONFLICT vs MERGE INTO

两者都是幂等插入（id 已存在就跳过），但语法属于不同数据库：

| 数据库 | 语法 |
|--------|------|
| H2 | `MERGE INTO ... KEY (id) VALUES (...)` |
| PostgreSQL | `INSERT INTO ... VALUES (...) ON CONFLICT (id) DO NOTHING` |
| MySQL | `INSERT IGNORE INTO ...` 或 `REPLACE INTO ...` |

---

## 四、验证测试

### 4.1 自动化验证

| 测试项 | 命令 | 结果 |
|--------|------|------|
| 静态检查（H2 配置已移除） | `Select-String -Path application.yml -Pattern "h2\|H2"` | ✅ 无结果 |
| 静态检查（PG 配置已写入） | `Select-String -Path application.yml -Pattern "postgresql"` | ✅ 命中 |
| 静态检查（MERGE 已替换） | `Select-String -Path data.sql -Pattern "MERGE"` | ✅ 无结果 |
| 静态检查（ON CONFLICT 已写入） | `Select-String -Path data.sql -Pattern "ON CONFLICT"` | ✅ 命中 2 行 |
| Maven 编译 | `mvn compile -q` | ✅ 通过 |
| Docker 运行 | `docker run hello-world` | ✅ Hello from Docker! |
| PG 容器启动 | `docker-compose up -d postgres` | ✅ Up |
| Spring Boot 启动 | `mvn spring-boot:run` | ✅ Started AgenthubApplication |

### 4.2 手动功能测试清单

| # | 测试项 | 预期行为 | 结果 |
|---|--------|---------|------|
| 1 | GET /agents | 返回 Claude Code + Codex 两个 Agent | ✅ |
| 2 | POST /conversations 创建会话 | 返回新会话 ID + 201 状态码 | ✅ |
| 3 | 中文标题存储 | PG 中 `title` 字段正确显示中文 | ✅（`?charSet=UTF-8` 修复后） |
| 4 | PG 直连查询 | `psql -c "SELECT * FROM conversations"` 能看到数据 | ✅ |
| 5 | data.sql 幂等 | 重启 Spring Boot 不报 Agent 重复插入 | ✅ |

---

## 五、关键决策记录

| # | 决策 | 结论 | 理由 |
|---|------|------|------|
| 1 | PG 版本 | 15-alpine | 稳定版本 + 最小镜像体积 |
| 2 | `hibernate.dialect` | 删除，不显式指定 | Hibernate 6 从 JDBC 驱动自动检测，手动指定会产生 `HHH90000025` 警告 |
| 3 | `sql.init.mode` | 保持 `always` | 开发阶段每次重启自动执行 seed data，`ON CONFLICT DO NOTHING` 保证幂等 |
| 4 | Docker Compose `version` | 删除 | Docker 29.x 已废弃此字段 |
| 5 | JDBC URL 编码 | 添加 `?charSet=UTF-8` | PG TCP 连接不继承 JVM 编码，中文会变成 `?` |
| 6 | 中文测试 | 使用 PSQL 直查而非依赖 PowerShell 输出 | PowerShell `Invoke-RestMethod` 终端编码不可靠，PG 内数据正常 |

---

## 六、产物清单

| 文件 | 操作 | 说明 |
|------|------|------|
| `docker-compose.yml` | **新建** | PG 15-alpine 容器，端口 5432，数据卷持久化 |
| `backend-java/src/main/resources/application.yml` | **修改** | H2 块删除，datasource/driver 切到 PG，dialect 删除 |
| `backend-java/src/main/resources/data.sql` | **修改** | `MERGE INTO` → `INSERT ON CONFLICT (id) DO NOTHING` |

### 代码统计

| 指标 | 数值 |
|------|------|
| 新增文件 | 1（docker-compose.yml） |
| 修改文件 | 2（application.yml, data.sql） |
| Java/Python 代码改动 | 0 |

---

## 七、数据流/架构影响

**迁移前**：
```
Spring Boot --JPA--> H2 内嵌数据库（进程内）
           文件路径: ./shared-data/agenthub.mv.db
```

**迁移后**：
```
Spring Boot --JDBC--> PostgreSQL 独立容器 (localhost:5432)
           数据库: agenthub, 用户: agenthub/agenthub123
           数据卷: agenthub_demo_postgres_data
```

**不变的部分**：所有 REST API、WebSocket、Agent SSE 协议完全不变。Hibernate 自动将 JPA 实体映射到 PG 表结构，业务代码零改动。

---

## 八、后续任务依赖

| 任务 | 可开始时机 | 关联 |
|------|----------|------|
| P1-A1 Redis 共享缓存 | PG 就绪后 | Java 将 Agent 元数据写入 Redis |
| P1-B1 JWT 认证 | PG 就绪后 | `users` 表已有 JPA 实体，需实现登录/注册接口 |
| P1-D3 Agent Redis 读取 | P1-A1 完成后 | Python 从 Redis 读取 Agent 列表 |
