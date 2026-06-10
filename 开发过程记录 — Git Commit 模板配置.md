# 开发过程记录 — Git Commit 模板配置

> **日期**: 2026-05-28\
> **所属模块**: project（项目工程配置）\
> **前置依赖**: 无（纯工程配置，不影响任何业务代码）

---

## 一、背景

在 C8 任务完成后编写 git commit 注释时，用户（成员C）注意到每次提交都需要手动遵循统一的格式规范。团队成员如果各自使用不同的 commit message 风格（如有的写中文、有的写英文、有的写 task 编号、有的不写），会导致提交历史混乱、changelog 难以自动生成。

为了规范化团队的提交习惯，决定创建一个 Git Commit 模板文件，并通过 Git 配置使其在每次 `git commit` 时自动预填充。

**模板来源**：从 C6 和 C8 两次任务的实际 commit message 中提炼出来的共性格式。

---

## 二、任务执行过程

### 2.1 从实际 commit 中提炼格式

回顾 C6 和 C8 的 commit message，发现共性结构：

```
feat(frontend-C6): ChatList 改为展示会话列表，支持新建会话    ← type(module-task): 描述
                                                                ← 空行
- 改动点 1                                                      ← 列表式详情
- 改动点 2
- 改动点 3
```

**格式要素**：
- 第一行：`<type>(<module>): <简短描述>` — 遵循 Conventional Commits 规范
- 空行分隔标题与正文
- 正文以 `- ` 开头的列表形式逐条列出改动点

### 2.2 第一版：固定模块名

初始模板内容：

```
<type>(frontend): <简短描述>

- <改动点 1>
- <改动点 2>
- <改动点 3>
```

**问题**：`frontend` 是硬编码的，项目实际有 `backend`、`agent-service`、`docs` 等多个模块，其他成员无法直接使用。

### 2.3 第二版：通用占位符 + 注释提示

修改为通用模板：

```
# type: feat / fix / refactor / style / docs / chore
# module: frontend / backend / agent-service / docs
<type>(<module>): <简短描述>

- <改动点 1>
- <改动点 2>
- <改动点 3>
```

**改动要点**：
- `(frontend)` → `(<module>)`：改为占位符，适配所有模块
- 新增两行 `#` 注释：列出可选的 type 和 module 值，作为填写提示
- `#` 开头的行在 Git 提交时会被自动忽略，不出现在最终的 commit message 中

**type 取值说明**：

| type | 含义 | 示例 |
|------|------|------|
| `feat` | 新功能 | `feat(frontend): 添加会话列表页面` |
| `fix` | 修复 bug | `fix(backend): 修复消息时间戳时区错误` |
| `refactor` | 代码重构 | `refactor(frontend): 提取 MessageInput 组件` |
| `style` | 样式调整 | `style(frontend): 统一按钮圆角为 12px` |
| `docs` | 文档变更 | `docs(project): 添加开发记录模板` |
| `chore` | 工程配置 | `chore(project): 添加 Git Commit 模板` |

### 2.4 激活模板配置

创建 `.gitmessage` 文件后，通过 Git 配置命令激活：

```powershell
git config commit.template .gitmessage
```

**配置原理**：`commit.template` 是 Git 的内置配置项。设置后，执行 `git commit`（不带 `-m` 参数）时，Git 会用模板文件内容预填充 COMMIT_EDITMSG 文件，在编辑器中打开供用户直接填写。

**验证生效**：

```powershell
git config commit.template
# 输出: .gitmessage
```

### 2.5 团队使用方式

`.gitmessage` 提交到远程仓库后，其他成员克隆项目后在根目录执行一次即可：

```powershell
git config commit.template .gitmessage
```

---

## 三、技术要点详解

### 3.1 Git commit.template 机制

`git commit` 的工作流程：

```
不带 -m:  git commit → 打开编辑器 → EDITOR 加载 COMMIT_EDITMSG → 用户编辑 → 保存提交
带 -m:    git commit -m "msg" → 直接使用参数作为 message → 提交

设置 template 后:
不带 -m:  git commit → 编辑器打开 → COMMIT_EDITMSG 预填充模板内容 → 用户在此基础上编辑 → 保存提交
```

模板只影响"不带 `-m`"的模式。使用 `-m` 时不受影响，适合简单的一次性提交。

### 3.2 # 注释行的自动忽略

Git 处理 COMMIT_EDITMSG 时，会**自动去除所有以 `#` 开头的行**。这正是 Git 内置的 `COMMIT_EDITMSG` 生成逻辑的一部分（`git status` 的输出也是用 `#` 标注的）。

因此模板中顶部的两行：
```
# type: feat / fix / refactor / style / docs / chore
# module: frontend / backend / agent-service / docs
```

在最终提交时不会出现，仅作为用户填写时的参考提示。

### 3.3 Conventional Commits 规范

采用的格式 `<type>(<scope>): <description>` 遵循 [Conventional Commits](https://www.conventionalcommits.org/) 规范。

**好处**：
- 可自动生成 CHANGELOG
- 可自动确定语义版本号（fix → patch，feat → minor，BREAKING CHANGE → major）
- 提交历史一目了然，便于 code review
- 与主流工具链兼容（semantic-release、commitlint 等）

---

## 四、验证测试

| # | 测试项 | 操作 | 预期结果 | 结果 |
|---|--------|------|----------|------|
| 1 | 模板文件存在 | 检查项目根目录 | `.gitmessage` 文件存在 | ✅ |
| 2 | 配置正确 | `git config commit.template` | 输出 `.gitmessage` | ✅ |
| 3 | 注释行在 `#` 开头 | 查看文件内容 | 前两行以 `#` 开头 | ✅ |
| 4 | module 可替换 | 查看模板内容 | 使用 `<module>` 占位符，非固定值 | ✅ |

---

## 五、关键决策与取舍

| 决策 | 选择 | 理由 |
|------|------|------|
| 模板存放位置 | 项目根目录 `.gitmessage` | Git 约定俗成的位置，克隆即可用 |
| 配置方式 | `git config commit.template` | 本地配置，不侵入 `.git/config` 远程共享部分 |
| 模板格式 | Conventional Commits | 主流规范，工具链兼容性好 |
| module 字段 | 通用 `<module>` 占位符 | 适配 frontend/backend/agent-service/docs 四个模块 |
| type 取值 | 6 种标准类型 | 覆盖团队日常开发场景，不过度细分 |

---

## 六、产物清单

| 文件 | 变更类型 | 说明 |
|------|----------|------|
| `.gitmessage` | 新增 | Git Commit 模板文件，7 行，含 type/module 注释提示 |

**配置变更**（非文件）：
- `git config commit.template .gitmessage`：激活模板（本地配置，需每个成员执行一次）

---

## 七、架构影响

```
改造前：
  团队成员各自编写 commit message → 格式不统一 → 提交历史混乱

改造后：
  git commit → 编辑器预填充 .gitmessage 模板 → 团队成员在此基础上填写
            → 格式统一(type(module): 描述) → 提交历史规范 → 可自动生成 CHANGELOG
```

纯工程配置变更，不影响任何业务代码逻辑。

---

## 八、后续建议

| 建议 | 说明 |
|------|------|
| 强制校验（可选） | 后续可引入 `commitlint` + `husky`，在 pre-commit hook 中校验 message 格式，防止不规范提交 |
| 团队成员告知 | 在团队群中通知 `.gitmessage` 的存在和激活方式 |
| CI 自动检查 | 可在 CI pipeline 中添加 commit message 格式校验步骤 |
