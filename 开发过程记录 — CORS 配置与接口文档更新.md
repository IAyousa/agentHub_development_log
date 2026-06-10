# 开发过程记录 — CORS 配置与接口文档更新

> **日期**: 2026-05-26\

---

## 一、背景

在完成团队任务分配和群聊方案设计之后，成员B 需要负责 Java 后端的 CORS 跨域配置（任务 B1）。同时，之前的 API 文档与团队讨论确定的方案A（两种UI+共享消息引擎）存在多处不一致，需要补充缺失接口并对齐设计决策。

---

## 二、任务执行过程

### 阶段 1：CORS 配置实现

#### 2.1 初始实现（WebMvcConfigurer 方式）

**操作内容**：

1. 在 `application.yml` 中新增 `cors.allowed-origins` 配置项：
   ```yaml
   cors:
     allowed-origins: http://localhost:5173
   ```

2. 创建 `CorsConfig.java`，实现 `WebMvcConfigurer.addCorsMappings()`：
   - `allowedOrigins` 通过 `@Value` 从 yml 读取，不硬编码
   - `allowedMethods`: GET, POST, PUT, DELETE, OPTIONS
   - `allowedHeaders`: `*`
   - `allowCredentials`: true
   - `maxAge`: 3600s

3. 同步修改 `WebSocketConfig.java`，STOMP 端点 CORS 也从 `"*"` 改为读取 yml 配置

**验证方式**：启动后端（8080）和前端（5173），在浏览器 F12 Console 执行 fetch 跨域测试。

#### 2.2 问题发现与修复

**问题**：浏览器报 `No 'Access-Control-Allow-Origin' header is present on the requested resource.`

**根因分析**：`WebMvcConfigurer.addCorsMappings()` 仅对 Spring MVC 的 `DispatcherServlet` 有效。H2 Console（`/h2-console`）由独立 `WebServlet` 处理，不走 Spring MVC，因此不受 `CorsRegistry` 管理。

**修复方案**：改用 `CorsFilter` Bean，工作在 Servlet Filter 层，拦截**所有**请求（Spring MVC + 第三方 Servlet + 静态资源）。

```java
@Bean
public CorsFilter corsFilter() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of(allowedOrigins.split(",")));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return new CorsFilter(source);
}
```

**修复后验证**：

```powershell
# 后端 OPTIONS 请求验证
Status: 200
Allow-Origin: http://localhost:5173
Allow-Methods: GET,POST,PUT,DELETE,OPTIONS
Allow-Credentials: true
```

#### 2.3 设计决策：保留 CorsFilter 而非回退

| 对比维度 | CorsFilter（当前） | WebMvcConfigurer（废弃） |
|---------|--------------------|--------------------------|
| 覆盖范围 | Servlet Filter 层，所有请求 | 仅 Spring MVC |
| H2 Console | ✅ | ❌ |
| Artifact 静态资源 | ✅ | 不确定 |
| Spring 官方推荐 | ✅ | 简化用法 |

**结论**：CorsFilter 是 Spring 官方推荐的最佳实践，覆盖范围更广，保留不变。

---

### 阶段 2：API 文档补充与对齐

#### 2.4 对照分析

任务文档中确定的方案A 需要以下接口，但原有 API 文档缺失：

| 缺失接口 | 说明 |
|---------|------|
| `GET /conversations/{id}` | 会话详情（含 agents 列表） |
| `PUT /conversations/{id}/agents` | 修改会话关联的 Agent |
| `PUT /messages/{id}/pin` | 消息置顶/取消置顶 |
| `POST/GET /artifacts/*` | Artifact 产物管理模块 |
| WebSocket `agentId` 字段 | 区分 Agent 发言 |
| WebSocket `agent_switch` 事件 | Orchestrator 调度通知 |
| SSE `agentId/agentName` 字段 | 内部通信携带 Agent 身份 |

#### 2.5 文档更新内容

**新增接口**（6 个）：

| 文档节 | 接口 | 说明 |
|-------|------|------|
| 2.2.3 | `GET /conversations/{id}` | 返回会话详情，含 agents 数组（id/name/type/avatarUrl） |
| 2.2.6 | `PUT /conversations/{id}/agents` | 替换会话的 Agent 列表，传空数组移除所有 Agent |
| 2.2.8 | `PUT /messages/{id}/pin` | 请求体 `{pinned: boolean}` |
| 2.4.1 | `POST /artifacts/upload` | multipart/form-data 上传 |
| 2.4.2 | `GET /artifacts/{id}` | 下载/预览产物 |
| 2.4.3 | `GET /conversations/{id}/artifacts` | 会话下所有产物 |

**修改格式**（4 处）：

| 位置 | 变更 |
|------|------|
| WebSocket 3.3.1 发送 | 去掉 `agentType`/`systemPrompt` 字段，仅保留 `conversationId` + `content` |
| WebSocket 3.3.2 接收 | MessageChunk 新增 `agentId` 字段；新增 `agent_switch` 事件 |
| WebSocket 3.4 时序 | 加入 Orchestrator 切换 Agent 的多 Agent 协作示例 |
| SSE 4.2 chunk | 新增 `agentId`/`agentName` 字段，含读取说明 |

**最终接口清单**（P0 阶段）：

| 分类 | 数量 | 具体接口 |
|------|------|---------|
| 会话管理 | 8 | CRUD + 历史消息 + Agent关联 + 置顶 |
| Agent 管理 | 3 | 列表 + 创建 + 详情 |
| 产物管理 | 3 | 上传 + 下载 + 列表 |
| WebSocket | 2 | 发送 + 接收（含 agent_switch） |
| 内部 HTTP | 2 | SSE 流式 + 健康检查 |

---

### 阶段 3：团队任务文档更新

#### 2.6 变更内容

根据接口文档更新 `团队任务分配.md`（v1.0 → v1.1）：

| 变更项 | 说明 |
|--------|------|
| 新增 2.7 节 | 接口范围总览表，列出 P0 全部接口 |
| 成员 A 表格 | A1 细化方法名；A5 明确 6 个 REST 路径；A6 增加 pin 接口 |
| 成员 A 新增 | 接口对齐清单（8 行表格，映射 API 文档章节号） |
| 成员 B 表格 | B1 标记已完成；B4/B5 对齐 Artifact 路径；B7 增加 PinRequest DTO |
| 成员 B 新增 | 接口对齐清单（6 行表格） |
| 成员 C 表格 | C4/C5 强调不传 agentType/systemPrompt，处理 agent_switch 事件 |
| 成员 D 表格 | D6 SSE 格式对齐 API 文档（agentId/agentName/finish） |
| 第 6 节 WS 格式 | 发送简化为 2 字段；新增 agent_switch 事件定义 |
| 冲刺目标 | CorsConfig 标记 ✅ 已完成 |
| 进度总览 | 后端完成度 25% → 30%；CORS 配置标记 ✅ |

#### 2.7 文件位置调整

`团队任务分配.md` 从 `docs/` 移至 `agent_collaboration/` 目录，作为内部协调文档，不进入代码仓库。

---

## 三、关键决策记录

| # | 决策 | 结论 | 理由 |
|---|------|------|------|
| 1 | CORS 实现方式 | `CorsFilter` Bean | 覆盖全路径（Spring MVC + 第三方 Servlet + 静态资源），Spring 官方推荐 |
| 2 | CORS 地址管理 | yml 读取，不硬编码 | 方便切换环境，多地址用逗号分隔 |
| 3 | 团队任务文档位置 | `agent_collaboration/` | 内部协调文档，不推送到远程仓库 |
| 4 | 接口文档范围 | P0 阶段 14 REST + 2 WS + 2 内部 HTTP | 聚焦最小可用产品，P1/P2 接口后续补充 |
| 5 | WebSocket 发送格式 | 仅传 `conversationId` + `content` | Orchestrator 自动调度，前端不感知 Agent 选择 |

---

## 四、涉及文件清单

| 文件 | 操作 | 说明 |
|------|------|------|
| `backend-java/.../config/CorsConfig.java` | **新建** | CorsFilter Bean |
| `backend-java/.../config/WebSocketConfig.java` | **修改** | CORS 改为 yml 读取 |
| `backend-java/.../resources/application.yml` | **修改** | 新增 `cors.allowed-origins` |
| `docs/API 契约定义与通信协议规范.md` | **大幅修改** | +157 行，补充 6 接口 + 格式对齐 |
| `agent_collaboration/团队任务分配.md` | **重建** | v1.1，对齐接口文档 |

---

## 五、对团队开发的启示

1. **CORS 配置要覆盖 Filter 层**：只用 `CorsRegistry` 不够，H2 Console、Artifact 静态资源、第三方 Servlet 都会绕过 Spring MVC。`CorsFilter` Bean 一次性解决。

2. **地址从 yml 读取**：开发/测试/生产环境的 CORS 地址不同，直接改配置而非改代码。

3. **接口文档是开发基准**：所有成员的任务描述中新增了"接口对齐清单"表格，将任务编号直接映射到 API 文档的具体章节和接口路径，避免理解偏差。

4. **Orchestrator 对前端透明**：前端发送消息不传 `agentType`/`systemPrompt`，这简化了前端逻辑，但也意味着成员 A 的 `WebSocketController` 和成员 D 的 Orchestrator 必须在 SSE 链路中正确传递 `agentId`/`agentName`。

---

## 六、后续行动项

| 任务 | 负责人 | 状态 |
|------|--------|------|
| B2 SecurityConfig | 成员B | 待开始 |
| B3 ArtifactConfig | 成员B | 待开始 |
| A1-A7 核心 Service/Controller | 成员A | 待开始 |
| C1-C11 前端 API 对接 | 成员C | 待开始 |
| D1-D8 Agent 适配器 | 成员D | 待开始 |
