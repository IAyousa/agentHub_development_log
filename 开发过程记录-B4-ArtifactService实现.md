# 开发过程记录 — B4 ArtifactService 实现

> **日期**: 2026-06-02
> **任务编号**: B4
> **所属模块**: backend-java
> **前置依赖**: B3 ArtifactConfig ✅（已完成）、B7 DTO ✅（已完成）

---

## 一、背景

成员 B 负责 AgentHub 后端基础设施层 + 产物管理 + Agent 管理，共 7 个任务（B1~B7）。B3 ArtifactConfig 已完成静态资源映射配置，B4 是产物管理的核心服务层——负责文件上传/查询/下载/删除的全部业务逻辑。

B4 完成后，B5（ArtifactController）即可开始开发。

任务分配文档中 B4 的定位：`ArtifactService`，依赖 ArtifactConfig 的存储路径，承接 Controller 层的调用请求。

---

## 二、任务执行过程

### 2.1 初版实现

按照任务规划创建 `ArtifactService.java`，初版使用 `ConcurrentHashMap` 作为内存注册表，`@PostConstruct` 扫描文件系统加载已有文件。

**核心设计**：
- 文件存储路径：`{storagePath}/{conversationId}/{messageId}/{filename}`
- 内存 Map 维护产物元数据（id → ArtifactDTO）
- `@Value("${artifact.storage.path}")` 读取配置的存储路径

### 2.2 代码审查（AI）

AI 对照项目现有 Service 模式（ConversationService、MessageService）进行审查，发现 8 个问题，按严重程度分级：

**P0 — 严重问题**：

| # | 问题 | 说明 |
|---|------|------|
| 1 | 重启后 ID 失效 | `@PostConstruct init()` 每次重启用 `UUID.randomUUID()` 重新生成 ID，前端持有的 artifact ID 全部失效 |
| 2 | `createdAt` 时间丢失 | 存量文件导入时设为 `LocalDateTime.now()`（当前时间），而非文件的真实创建时间 |
| 3 | 无持久化 | 元数据只存在内存 `ConcurrentHashMap`，重启丢失。对比 ConversationService/MessageService 均使用 JPA Repository → H2 |

**P1 — 中等问题**：

| # | 问题 | 说明 |
|---|------|------|
| 4 | 同名文件静默覆盖 | 同 conversationId+messageId 下上传同名文件，旧文件直接被覆盖 |
| 5 | 参数无校验 | `save()` 对 `MultipartFile`、`conversationId`、`messageId` 均无 null 检查 |
| 6 | 路径穿越风险 | `conversationId`/`messageId` 含 `../` 可写入任意目录 |

**P2 — 小问题**：

| # | 问题 | 说明 |
|---|------|------|
| 7 | 无 `delete` 方法 | 无法删除产物，ArtifactController 将缺少对应功能 |
| 8 | `@Value` 无默认值 | 缺少 `:fallback` 语法，配置缺失时启动直接报错 |

### 2.3 修复实施

**新增 Artifact JPA Entity**：

```java
@Data
@Entity
@Table(name = "artifacts")
public class Artifact {
    @Id @Column(length = 36)
    private String id;
    private String filename;        // 原始文件名（前端看到的名字）
    private String storedName;      // 磁盘文件名（UUID + 扩展名）
    private Long fileSize;
    private String conversationId;
    private String messageId;
    private LocalDateTime createdAt;

    @PrePersist
    public void prePersist() {
        if (this.id == null) this.id = UUID.randomUUID().toString();
        if (this.createdAt == null) this.createdAt = LocalDateTime.now();
    }
}
```

**新增 ArtifactRepository**：

```java
@Repository
public interface ArtifactRepository extends JpaRepository<Artifact, String> {
    List<Artifact> findByConversationId(String conversationId);
    List<Artifact> findByConversationIdAndMessageId(String conversationId, String messageId);
}
```

**重写 ArtifactService**：

| 方法 | 改动 |
|------|------|
| `save()` | 注入 ArtifactRepository，UUID 命名文件防碰撞，参数校验 + 路径穿越防护，写磁盘 + 写 DB 在 `@Transactional` 下原子执行 |
| `getByConversation()` | 从内存过滤改为 `artifactRepository.findByConversationId()` |
| `getFile()` | 从 registry.get() 改为 `artifactRepository.findById()`，查 DB 后读磁盘 |
| `delete()` | **新增**：删 DB 记录 + 删磁盘文件 |
| `init()` | 仅导入 DB 中不存在的孤儿文件，用 `Files.readAttributes()` 读取真实创建时间，用 `storedName` 作为确定性 ID |
| `validatePathSegment()` | **新增**：拒绝含 `..` `/` `\` 的路径参数 |

---

## 三、技术要点讲解

（本次中尚未完整讲解，计划下次会话继续）

### 3.1 @Service 与 Spring Bean 生命周期

`@Service` 标记业务服务类 → Spring 启动时扫描并创建单例 → 通过 `@RequiredArgsConstructor` + `final` 字段注入给 Controller。

### 3.2 @Transactional 事务边界

写操作（save/delete）使用默认 `@Transactional`（读写事务），只读操作使用 `@Transactional(readOnly = true)`（跳过脏检查，性能更好）。如果 save 过程中磁盘写入成功但 DB 写入失败，Spring 自动回滚 JPA 操作——但磁盘文件不会自动回滚，这是当前设计的已知局限。

### 3.3 UUID 文件名防碰撞

磁盘文件以 `{UUID}.{扩展名}` 命名（如 `a1b2c3d4.js`），原始文件名存在 DB 的 `filename` 字段。API 返回原始文件名，Controller 通过 ID 查找 DB 记录 → 获取 `storedName` → 定位磁盘文件。

### 3.4 路径穿越防护

`validatePathSegment()` 检查参数是否包含 `..`、`/`、`\`，防止恶意输入跳出 `storagePath` 根目录。这是文件系统的「SQL 注入防护」。

### 3.5 @PostConstruct 孤儿文件恢复

每次服务启动时扫描 `storagePath` 目录树，对于磁盘存在但 DB 无记录的文件，自动创建 DB 记录。这是防御性设计——相当于备份恢复机制。

---

## 四、验证测试

| 测试项 | 命令 | 结果 |
|--------|------|------|
| Spring Boot 编译 | `mvn compile -q` | ✅ 零错误 |

---

## 五、关键决策记录

| # | 决策 | 结论 | 理由 |
|---|------|------|------|
| 1 | 元数据持久化方式 | JPA Entity + ArtifactRepository → H2 | 与 ConversationService/MessageService 保持一致 |
| 2 | 磁盘文件名 | UUID + 原始扩展名 | 消除同名文件覆盖风险，原始名存 DB 的 filename 字段 |
| 3 | DTO 与 Entity 分离 | Entity 含 7 字段（含 storedName），DTO 含 6 字段（不含 storedName） | storedName 是内部实现细节，不暴露给前端 |
| 4 | init() 策略 | 仅恢复孤儿文件，不覆盖已有记录 | 避免重启修改已有 artifact ID |
| 5 | 参数校验 | Service 层做校验，Controller 只做异常转换 | 即使未来有其他调用方（如 WebSocket 上传），校验逻辑不重复 |
| 6 | @Value 默认值 | `@Value("${artifact.storage.path:./artifacts}")` | 防御性编程，配置缺失时不至于启动崩溃 |

---

## 六、产物清单

| 文件 | 操作 | 行数 | 说明 |
|------|------|------|------|
| `model/Artifact.java` | 新增 | ~40 | JPA 实体，artifacts 表 |
| `repository/ArtifactRepository.java` | 新增 | ~15 | JPA 仓库接口 |
| `service/ArtifactService.java` | 重写 | ~160 | 产物服务：上传/查询/下载/删除 |

### 代码统计

| 指标 | 数值 |
|------|------|
| 新增文件 | 3 |
| 新增代码行 | ~215 |

---

## 七、数据流/架构影响

```
改造前：
  ArtifactService 元数据 = ConcurrentHashMap（内存）
  → 重启丢失全部元数据
  → 产物 ID 不稳定

改造后：
  ArtifactService 元数据 = JPA/H2 持久化
  → 元数据与文件生命周期一致
  → 产物 ID 稳定（UUID 持久化在 DB）
  → 架构与其他 Service 统一
```

```
上传流程：
  前端 multipart → ArtifactController
  → ArtifactService.save(file, convId, msgId)
  → validatePathSegment() 安全检查
  → Files.createDirectories() 创建目录
  → file.transferTo({UUID}.ext) 写磁盘
  → artifactRepository.save() 写 DB
  → 返回 ArtifactDTO（仅含前端需要的字段）
```

---

## 八、后续任务依赖

| 任务 | 可开始时机 | 关联 |
|------|----------|------|
| B5 ArtifactController | ✅ 即刻 | 依赖 ArtifactService 的全部方法 |
| B6 AgentController | ✅ 即刻（与 B5 并行） | 直接操作 AgentRepository |
