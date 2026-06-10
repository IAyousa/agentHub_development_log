# 开发过程记录 — B5 ArtifactController 实现

> **日期**: 2026-06-03
> **任务编号**: B5
> **所属模块**: backend-java
> **前置依赖**: B4 ArtifactService ✅（已完成）、B7 DTO ✅（已完成）、B3 ArtifactConfig ✅（已完成）

---

## 一、背景

B5 是产物管理的 REST 控制器层，职责是接收前端 HTTP 请求并转发给 ArtifactService 处理。B4 已完成全部业务逻辑（上传/查询/下载/删除），B5 只需做参数接收 + 异常转换 + 返回响应。

API 契约文档（第 2.4 节）定义了 3 个端点，B5 额外补充了 DELETE 端点：

| 端点 | 说明 | 契约来源 |
|------|------|---------|
| `POST /artifacts/upload` | 上传产物文件 | 契约 2.4.1 |
| `GET /artifacts/{id}` | 下载/预览产物 | 契约 2.4.2 |
| `GET /conversations/{id}/artifacts` | 获取会话下所有产物 | 契约 2.4.3 |
| `DELETE /artifacts/{id}` | 删除产物 | B4 Service 已有 `delete()` 方法，补全 REST 接口 |

---

## 二、任务执行过程

### 2.1 Trae 生成初版代码

kyo19c 使用 Trae 生成 ArtifactController 初版。

**初版存在 6 个问题**（AI 审查发现）：

| # | 问题 | 严重程度 |
|---|------|----------|
| 1 | `@RequestMapping("/api")` 给所有路径加了 `/api` 前缀，契约文档无此前缀 | 🔴 P0 |
| 2 | 错误响应缺少 `timestamp` 和 `path` 字段 | 🔴 P0 |
| 3 | 手写 `determineContentType()` 硬编码 MIME 类型，仅覆盖 8 种 | 🟡 P1 |
| 4 | Content-Disposition 未用 RFC 5987 编码 | 🟡 P1 |
| 5 | 缺少 DELETE 端点 | 🟡 P1 |
| 6 | 上传端点未声明 `consumes = MULTIPART_FORM_DATA_VALUE` | 🟢 P2 |

### 2.2 AI 修复

**修复 1 — 去掉 `/api` 前缀**：

```java
// ❌ 修复前
@RestController
@RequestMapping("/api")
public class ArtifactController {
    @PostMapping("/artifacts/upload")  // → /api/artifacts/upload
}

// ✅ 修复后
@RestController
public class ArtifactController {
    @PostMapping("/artifacts/upload")  // → /artifacts/upload
}
```

**修复 2 — 提取 `error()` 统一错误方法**：

```java
private ResponseEntity<?> error(HttpStatus status, String errorCode,
                                 String message, String path) {
    return ResponseEntity.status(status).body(Map.of(
            "error", errorCode,
            "message", message,
            "timestamp", LocalDateTime.now().toString(),
            "path", path
    ));
}
```

所有异常处理统一调用此方法，错误响应四字段 `{error, message, timestamp, path}` 完全对齐契约文档。

**修复 3 — `Files.probeContentType()` 替代硬编码**：

```java
// ❌ 修复前：手写 if-else，仅 8 种类型
private String determineContentType(String filename) {
    if (filename.endsWith(".html")) return "text/html";
    ...
}

// ✅ 修复后：Java 标准方法，自动探测所有已注册 MIME 类型
String contentType = Files.probeContentType(path);
if (contentType == null) contentType = "application/octet-stream";
```

**修复 4 — RFC 5987 文件名编码**：

```java
String encoded = URLEncoder.encode(file.getName(), StandardCharsets.UTF_8)
        .replace("+", "%20");
"inline; filename*=UTF-8''" + encoded
```

**修复 5 — 补充 DELETE 端点**：

```java
@DeleteMapping("/artifacts/{id}")
public ResponseEntity<?> delete(@PathVariable String id) {
    artifactService.delete(id);
    return ResponseEntity.noContent().build();  // 204
}
```

---

## 三、技术要点讲解

### 3.1 `@RestController` vs `@Controller`

`@RestController` = `@Controller` + `@ResponseBody`，所有方法返回值自动序列化为 JSON。B5 是纯 REST API，不需要视图渲染，使用 `@RestController`。

项目中的 `WebSocketController` 使用的是 `@Controller`（STOMP 消息处理），两者分工不同。

### 3.2 `ResponseEntity<?>` 统一响应包装

每个端点返回 `ResponseEntity<?>`，可以精确控制 HTTP 状态码：

| 场景 | 状态码 |
|------|--------|
| 上传成功 | `201 Created` |
| 查询/下载成功 | `200 OK` |
| 删除成功 | `204 No Content` |
| 参数校验失败 | `400 Bad Request` |
| 文件不存在 | `404 Not Found` |
| 服务器异常 | `500 Internal Server Error` |

### 3.3 `@PathVariable` vs `@RequestParam`

- `@PathVariable`：从 URL 路径中取值，如 `/artifacts/{id}` 中的 `id`
- `@RequestParam`：从 Query String 或 Form 中取值，如 `?conversationId=xxx` 或 `multipart/form-data` 中的字段

### 3.4 Controller 层职责边界

B5 遵循「Controller 只做路由 + 参数传递 + 异常转 HTTP 状态码」原则：

```
Controller 层:  接收请求 → 校验 → 调用 Service → 异常转 HTTP 状态码 → 返回响应
Service 层:    业务逻辑 → 参数校验 → 磁盘操作 → DB 操作 → 返回结果
```

路径穿越防护、文件名生成、事务管理等业务规则全部在 B4 Service 中完成，Controller 不重复实现。

---

## 四、验证测试

### 4.1 自动化验证

| 测试项 | 命令 | 结果 |
|--------|------|------|
| Spring Boot 编译 | `mvn compile -q` | ✅ 零错误 |

### 4.2 交互式端到端测试（8 步）

| # | 测试项 | 命令/方式 | 预期 | 结果 |
|---|--------|----------|------|:--:|
| 1 | 文件完整性 | `test -f` 检查 4 个文件 | 全部存在 | ✅ |
| 2 | 编译 | `mvn compile -q` | 无输出 | ✅ |
| 3 | 启动服务 | `mvn spring-boot:run` | Started on 8080 | ✅ |
| 4 | 上传 | `curl -X POST /artifacts/upload -F ...` | 201 + 6 字段 JSON | ✅ |
| 5 | 下载 | `curl -o NUL -w "%{http_code}" /artifacts/{id}` | 200 | ✅ |
| 6 | 列表 | `curl /conversations/{id}/artifacts` | 200 + `artifacts` 数组 | ✅ |
| 7 | 删除 | `curl -X DELETE -w "%{http_code}" /artifacts/{id}` | 204 | ✅ |
| 8 | 删除后查询 | `curl /artifacts/{id}` | 404 + 四字段错误 | ✅ |

**测试数据**：上传 `CLAUDE.md`（13421 字节），artifact ID `24dd0f03-eb58-43dc-a266-e12bb8abef65`，conversationId `conv-test-001`。

---

## 五、关键决策记录

| # | 决策 | 结论 | 理由 |
|---|------|------|------|
| 1 | Controller 路径格式 | 无 `/api` 前缀，直接 `/artifacts/**` | 契约文档和前端 Axios baseURL 均无 `/api` |
| 2 | Content-Type 探测 | `Files.probeContentType()` | Java 标准方法，覆盖所有已注册 MIME 类型 |
| 3 | 错误统一方法 | 提取 `error(status, code, msg, path)` | 四字段格式统一，避免重复代码 |
| 4 | DELETE 端点 | 补充实现 | B4 Service 已支持 `delete()`，补全 REST 语义 |
| 5 | 参数校验位置 | Service 层做校验 | Controller 不重复校验，即使未来有 WebSocket 上传也复用 |

---

## 六、产物清单

| 文件 | 操作 | 行数 | 说明 |
|------|------|------|------|
| `controller/ArtifactController.java` | 新增 | ~112 | 4 个 REST 端点 + 统一错误处理 + Content-Type 自动探测 |

### 代码统计

| 指标 | 数值 |
|------|------|
| 新增文件 | 1 |
| 新增代码行 | ~112 |

---

## 七、数据流/架构影响

```
前端请求:
  POST /artifacts/upload (multipart)
    → ArtifactController.upload()
    → ArtifactService.save(file, convId, msgId)
    → 返回 201 + ArtifactDTO

  GET /artifacts/{id}
    → ArtifactController.download()
    → ArtifactService.getFile(id)
    → Files.probeContentType() 自动探测 MIME
    → 返回 200 + 文件流 + Content-Type + Content-Disposition

  DELETE /artifacts/{id}
    → ArtifactController.delete()
    → ArtifactService.delete(id)
    → 删除 DB 记录 + 磁盘文件
    → 返回 204 No Content
```

B5 与 B3（ArtifactConfig 静态资源映射）互补：B3 处理 iframe 多文件预览场景（相对路径资源自动加载），B5 处理单文件上传/下载/删除。

---

## 八、后续任务依赖

| 任务 | 可开始时机 | 关联 |
|------|----------|------|
| B6 AgentController | ✅ 即刻 | 直接操作 AgentRepository |
