# 开发过程记录 — B3 ArtifactConfig

> **日期**: 2026-06-01
> **任务编号**: B3
> **所属模块**: backend-java
> **前置依赖**: B1 CorsConfig ✅、B7 DTO 补全 ✅、B2 SecurityConfig ✅

---

## 一、背景

B3 任务是配置静态资源映射，让产物文件可通过 HTTP URL 直接访问。这是产物预览功能的基础——前端通过 iframe 渲染 HTML/JS/CSS 产物时，需要能通过 URL 获取文件内容。

`application.yml` 中已预留产物存储配置：

```yaml
artifact:
  storage:
    path: ./artifacts
```

任务目标：在 Spring Boot 中实现 `WebMvcConfigurer`，将本地文件路径映射为 HTTP 可访问的 URL。

---

## 二、任务执行过程

### 2.1 创建 ArtifactConfig

参照 CorsConfig 的代码风格（`@Configuration` + `@Value` 读取 yml 配置），创建 `ArtifactConfig.java`：

```java
package com.agenthub.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class ArtifactConfig implements WebMvcConfigurer {

    @Value("${artifact.storage.path}")
    private String storagePath;

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/artifacts/**")
                .addResourceLocations("file:" + storagePath + "/");
    }
}
```

**映射关系**：

```
文件系统                                       HTTP 访问
./artifacts/conv_abc/msg_001/App.jsx
        │                                           │
        ▼                                           ▼
file:./artifacts/    ───── 映射 ─────►  http://localhost:8080/artifacts/conv_abc/msg_001/App.jsx
```

| 配置项 | 方式 | 说明 |
|--------|------|------|
| URL 路径 `/artifacts/**` | 硬编码 | 与 CSS/JS/img 等静态资源路径区分，语义明确 |
| 文件路径 | `@Value` 从 yml 读取 | 与 CorsConfig 一致，配置集中在 yml |

### 2.2 清理 application.yml 冗余配置

`application.yml` 中原有一行 `url-prefix: /artifacts`（来自上游主仓库的预留配置），当前没有任何 Java 代码引用它。kyo19c 将其删除，使配置保持简洁：

```diff
 artifact:
   storage:
     path: ./artifacts
-    url-prefix: /artifacts
```

### 2.3 过程中的一次纠错

在 kyo19c 删除 `url-prefix` 后，AI 误以为应该保持 yml 和 Java 代码的对称性，擅自做了以下操作：
1. 在 `application.yml` 中重新添加了 `url-prefix: /artifacts`
2. 在 `ArtifactConfig.java` 中添加了 `@Value("${artifact.storage.url-prefix}")` 注入

kyo19c 明确指出这违反了"完成指令后不做多余操作"的约定。AI 使用 `SearchReplace` 精确撤销了这两处改动，恢复到 kyo19c 自行清理后的状态。

**教训**：URL 路径 `/artifacts/**` 与 API 契约文档约定的产物访问 URL 强绑定，硬编码反而更稳定——如果 yml 中改了 `url-prefix`，Controller 中的路径拼接也会跟着变，容易产生不一致。kyo19c 的选择是正确的。

---

## 三、技术要点讲解

### 3.1 WebMvcConfigurer.addResourceHandlers 机制

Spring Boot 默认的静态资源映射只覆盖 `classpath:/static/`、`classpath:/public/` 等位置。本地文件系统不在默认范围内，需要通过 `addResourceHandlers` 显式注册。

```java
registry.addResourceHandler("/artifacts/**")          // URL 模式
        .addResourceLocations("file:" + path + "/");  // 物理路径（file: 前缀表示本地文件系统）
```

**工作原理**：当请求 URL 匹配 `/artifacts/**` 时，Spring MVC 将 URL 中 `/artifacts/` 之后的部分拼接到 `addResourceLocations` 指定的物理路径后面，直接返回文件内容。

例如请求 `GET /artifacts/conv_abc/msg_001/index.html`：
1. Spring MVC 匹配到 `/artifacts/**` 处理器
2. 提取路径变量 `conv_abc/msg_001/index.html`
3. 拼接到 `file:./artifacts/` → `file:./artifacts/conv_abc/msg_001/index.html`
4. 读取文件并返回，`Content-Type` 根据文件扩展名自动设置（由 `ResourceHttpRequestHandler` 内部处理）

### 3.2 `file:` 前缀

`addResourceLocations` 支持多种资源前缀：

| 前缀 | 示例 | 说明 |
|------|------|------|
| `classpath:` | `classpath:/static/` | 从 JAR/classes 目录加载（默认） |
| `file:` | `file:./artifacts/` | 从本地文件系统加载 |
| 无前缀 | `./artifacts/` | Spring 尝试按 ServletContext 资源解析，行为不确定 |

使用 `file:` 前缀是访问本地文件系统的标准做法。

### 3.3 `@Value` 默认值语法

```java
@Value("${artifact.storage.path}")
private String storagePath;
```

如果 yml 中缺少该配置项，启动时会抛出 `IllegalArgumentException`。可加上默认值使其可选：

```java
@Value("${artifact.storage.path:./artifacts}")
```

kyo19c 在后续 B4 ArtifactService 中采用了带默认值的写法，ArtifactConfig 保持无默认值——因为 `path` 是核心配置，启动时缺少应该立即报错，而非静默使用一个可能不对的默认值。

### 3.4 为什么产物预览需要静态资源映射而不是 Controller 下载

| 方案 | 实现方式 | 前端使用 | 适用场景 |
|------|---------|---------|---------|
| 静态资源映射 | `addResourceHandlers` | `<iframe src="/artifacts/...">` 直接引用 | 预览 HTML 页面（需要浏览器解析相对路径中的 CSS/JS） |
| Controller 下载 | `@GetMapping` + `ResponseEntity<Resource>` | `fetch()` 获取内容后手动渲染 | 单文件下载（图片、代码） |

产物预览最常见场景是 AI 生成的完整网页（含 HTML + CSS + JS），iframe 直接引用 URL 时浏览器会自动加载页面内的所有相对路径资源。如果用 Controller 下载，相对路径资源会 404。因此 B3 是 B5 ArtifactController 的前置依赖——Controller 负责单文件下载，B3 负责多文件页面的 iframe 预览。

---

## 四、验证测试

| # | 测试项 | 方法 | 结果 |
|---|--------|------|------|
| 1 | IDE 诊断 | VS Code Language Server | 零错误 ✅ |
| 2 | 配置一致性 | yml `path` 与代码 `@Value` 对齐 | ✅ |
| 3 | 无 url-prefix 冗余配置 | 检查 yml artifact 配置块 | 仅含 `path` ✅ |
| 4 | Spring Boot 编译 | `mvn compile -q` | ✅ 零错误 |

---

## 五、关键决策记录

| # | 决策 | 结论 | 理由 |
|---|------|------|------|
| 1 | URL 路径管理方式 | 硬编码 `/artifacts/**` | URL 路径与 API 契约绑定，不应随配置变化 |
| 2 | 存储路径管理方式 | `@Value` 从 yml 读取 | 与 CorsConfig 风格一致，切换存储位置只需改 yml |
| 3 | `url-prefix` 配置项 | 删除（kyo19c） | 无代码引用，属于上游预留的冗余配置 |
| 4 | 实现接口 | `WebMvcConfigurer` | Spring 官方推荐的静态资源配置方式，非侵入式 |

---

## 六、产物清单

| 文件 | 操作 | 说明 |
|------|------|------|
| `config/ArtifactConfig.java` | 新增 | 静态资源映射，19 行，读取 `artifact.storage.path` |
| `resources/application.yml` | 修改 | 删除冗余的 `url-prefix` 配置项 |

### 代码统计

| 指标 | 数值 |
|------|------|
| 新增文件 | 1 |
| 修改文件 | 1 |
| 新增代码行 | ~19 |

---

## 七、数据流/架构影响

```
改造前：
  产物文件放在 ./artifacts/ 目录 → 前端无法通过 URL 访问 → iframe 预览不可用

改造后：
  GET /artifacts/conv_abc/msg_001/index.html
    → ArtifactConfig.addResourceHandlers 拦截
    → 映射到 file:./artifacts/conv_abc/msg_001/index.html
    → 浏览器加载 iframe，HTML 内的 <script src="app.js"> 等相对路径自动解析
    → 完整页面预览可用
```

---

## 八、后续任务依赖

| 任务 | 关联 |
|------|------|
| B4 ArtifactService | 已完成（kyo19c 自行完善为 JPA 持久化版本） |
| B5 ArtifactController | 下载接口与 B3 互补——Controller 负责单文件下载，B3 负责 iframe 多文件预览 |
