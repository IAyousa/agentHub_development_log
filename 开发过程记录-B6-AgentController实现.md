# 开发过程记录 — B6 AgentController 实现

> **日期**: 2026-06-03
> **任务编号**: B6
> **所属模块**: backend-java
> **前置依赖**: B7 DTO ✅、AgentRepository ✅（JPA 层已就绪）

---

## 一、背景

B6 是 Agent 管理的 REST 控制器层，职责是对 `agents` 表提供基础的查询和创建接口。与其他 B-part 任务不同，B6 不需要独立的 Service 层——Agent 管理是简单 CRUD，直接操作 `AgentRepository` 即可，任务规划中明确 B6 无前置 Service 依赖。

API 契约文档（第 2.3 节）定义了 3 个端点：

| 端点 | 说明 |
|------|------|
| `GET /agents` | 获取可用 Agent 列表 |
| `POST /agents` | 创建自定义 Agent |
| `GET /agents/{id}` | 获取 Agent 详情 |

---

## 二、任务执行过程

### 2.1 kyo19c 自行完成初版

kyo19c 独立完成 AgentController 的完整实现，代码质量明显高于 B5 初版（B5 由 Trae 生成后需要 6 处修复）。

**初版正确实现了**：
- 3 个端点全部对齐 API 契约
- 错误响应四字段 `{error, message, timestamp, path}` 格式，与 B5 统一
- `capabilities` 字段的 JSON 序列化/反序列化（DB 存 JSON 字符串 ↔ API 返回 JSON 数组）
- `isBuiltin` 判定逻辑：`createdBy == null` 为内置 Agent
- 参数校验：`name` 和 `type` 非空检查
- `@SuppressWarnings("unchecked")` 处理 `Map<String, Object>` 到 `List<String>` 的类型转换

### 2.2 AI 审查发现 1 个问题

| # | 问题 | 严重程度 |
|---|------|----------|
| 1 | POST 响应比契约多了 `avatarUrl` 和 `isBuiltin` 字段 | 🟢 P2 |

**原因**：`create()` 方法返回 `toDetail(saved)`，而 `toDetail()` 继承自 `toSummary()` 的全部字段（含 `avatarUrl`、`isBuiltin`），但契约定义的 POST 响应只应包含 6 个字段：`id, name, type, systemPrompt, capabilities, createdAt`。

### 2.3 修复

将 POST 方法的响应改为独立构造 6 字段 Map，不再复用 `toDetail()`：

```java
// ✅ 修复后：精确匹配契约
Map<String, Object> response = new LinkedHashMap<>();
response.put("id", saved.getId());
response.put("name", saved.getName());
response.put("type", saved.getType());
response.put("systemPrompt", saved.getSystemPrompt());
response.put("capabilities", parseCapabilities(saved.getCapabilities()));
response.put("createdAt", saved.getCreatedAt() != null
        ? saved.getCreatedAt().toString() : null);
return ResponseEntity.status(HttpStatus.CREATED).body(response);
```

---

## 三、技术要点讲解

### 3.1 Agent 实体中 capabilities 的存储方式

`capabilities` 在 Agent 实体中定义为 `@Column(columnDefinition = "TEXT")`，存储 JSON 字符串（如 `["代码生成", "Debug"]`），而非数据库关系表。

| 方案 | 存储方式 | 优点 | 缺点 |
|------|---------|------|------|
| JSON 字符串 | `capabilities TEXT` | 读写简单，无需 JOIN | 无法按标签查询 |
| 多对多关系表 | `agent_capabilities` 关联表 | 可查询、可索引 | 多一次 JOIN |

MVP 选择 JSON 字符串方案，因为 Agent 数量少、标签仅用于前端展示，不需要数据库层面查询。

### 3.2 ObjectMapper 的双向转换

```java
// Java → JSON（存储前）
agent.setCapabilities(objectMapper.writeValueAsString(capabilities));
// List<String> → '["React开发", "组件设计"]'

// JSON → Java（返回前端时）
objectMapper.readValue(capabilitiesJson, new TypeReference<List<String>>() {});
// '["React开发", "组件设计"]' → List<String>
```

`TypeReference` 是 Jackson 处理泛型的机制——Java 的类型擦除导致 `List<String>.class` 不可用，必须用 `TypeReference` 保留泛型信息。

### 3.3 isBuiltin 的设计

```java
map.put("isBuiltin", agent.getCreatedBy() == null);
```

种子数据（`data.sql`）中 `created_by` 故意不写，默认为 NULL → `isBuiltin = true`。用户通过 API 创建时，`createdBy` 设为 `"system"` → `isBuiltin = false`。前端据此区分内置 Agent（不可删除/修改）和自定义 Agent。

### 3.4 为什么 B6 不需要 Service 层

| 决策因素 | 说明 |
|---------|------|
| 业务逻辑简单 | 仅 CRUD，无复杂事务或跨表操作 |
| 与任务规划一致 | 任务文档明确 B6 直接操作 AgentRepository |
| 避免过度设计 | 3 个端点、无磁盘操作、无外部调用 |

对比 B4（ArtifactService）需要 Service 层：涉及文件系统操作 + DB 操作的原子性、路径穿越防护、UUID 文件名生成等复杂逻辑。B6 无此类需求。

---

## 四、验证测试

### 4.1 自动化验证

| 测试项 | 命令 | 结果 |
|--------|------|------|
| Spring Boot 编译 | `mvn compile -q` | ✅ 零错误 |

### 4.2 端到端测试（3 步）

| # | 测试项 | 命令 | 预期 | 结果 |
|---|--------|------|------|:--:|
| 1 | Agent 列表 | `curl /agents` | 200 + `agents` 数组，包含种子数据 | ✅ |
| 2 | 创建 Agent | `curl -X POST /agents -d '{...}'` | 201 + 6 字段 | ✅ |
| 3 | Agent 详情 | `curl /agents/{id}` | 200 + 含 `isBuiltin: false` | ✅ |

**测试数据**：创建了名为「测试Agent」的自定义 Agent，id `e21cac1e-be98-474f-8bb3-eaefa6116767`。

---

## 五、关键决策记录

| # | 决策 | 结论 | 理由 |
|---|------|------|------|
| 1 | Service 层 | 不需要，Controller 直接操作 Repository | 简单 CRUD 无复杂业务逻辑，与任务规划一致 |
| 2 | capabilities 存储 | JSON 字符串（TEXT 字段） | MVP Agent 数量少，仅前端展示用 |
| 3 | isBuiltin 判定 | `createdBy == null` | 种子数据不设 created_by，API 创建设为 "system" |
| 4 | 请求体解析 | `Map<String, Object>` 手动提取字段 | 字段少，避免为创建请求单独建 DTO |
| 5 | POST 响应字段 | 仅返回契约定义的 6 字段 | 不多返回 `avatarUrl`（null）和 `isBuiltin`（false） |

---

## 六、产物清单

| 文件 | 操作 | 行数 | 说明 |
|------|------|------|------|
| `controller/AgentController.java` | 新增 | ~131 | 3 个 REST 端点 + JSON 双向转换 + 统一错误处理 |

### 代码统计

| 指标 | 数值 |
|------|------|
| 新增文件 | 1 |
| 新增代码行 | ~131 |

---

## 七、数据流/架构影响

```
获取列表:
  GET /agents → AgentController.list()
  → agentRepository.findAll()
  → 逐条 toSummary()（capabilities JSON 反序列化 + isBuiltin 判定）
  → {"agents": [...]}

创建 Agent:
  POST /agents → AgentController.create()
  → 参数校验（name/type 非空）
  → capabilities 序列化为 JSON 字符串
  → agentRepository.save()
  → 201 + 6 字段响应

获取详情:
  GET /agents/{id} → AgentController.detail()
  → agentRepository.findById()
  → 404 或 200 + toDetail()（含 systemPrompt + createdAt）
```

B6 是 B-part 的最后一个任务，完成后成员 B 的全部 7 个任务均已交付。

---

## 八、后续任务依赖

B-part（B1~B7）全部完成，无剩余任务依赖。后续如需扩展：
- Agent 修改/删除接口（PATCH/DELETE）
- Agent 列表分页/搜索
- Agent 头像上传
