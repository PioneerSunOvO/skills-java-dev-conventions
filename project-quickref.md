# 项目 quickref 创建指南

本文件说明：**何时**需要项目专属 skill、**如何探测**、**缺失时 Agent 该怎么做**、**如何从零脚手架**。

通用编码底线仍在 [SKILL.md](SKILL.md) / [quick-check.md](quick-check.md)；**不要把某一仓库的类名、业务词写进通用 skill 正文**。

---

## 何时需要 quickref

| 情况 | 建议 |
|------|------|
| 仅有 Spring Boot，结构简单，无固定响应/分页封装 | `CLAUDE.md` + 本 skill 可能够用 |
| 有统一响应类、分页方式、多模块、禁改目录 | **建议建** `<project>-quickref` |
| 有业务术语、金额口径、中文 Git scope、团队 `@author` 约定 | **强烈建议建** |
| 团队多人 / 多 Agent 工具共用同一仓库 | **建议建**，并配 `.cursor/rules` 或根目录 `AGENTS.md` |

命名约定：目录与 YAML `name` 用小写 + 连字符，后缀 `-quickref`，例如 `myshop-quickref`。

---

<a id="agent-未发现-quickref"></a>

## Agent：未发现 quickref 时（必读）

在**新增或修改**本仓库 Java、Mapper XML、Spring 配置（yml/yaml/properties）**之前**，先探测项目 quickref（见下节路径）。

### 探测结果与动作

| 结果 | Agent 动作 |
|------|------------|
| 存在 `*-quickref/SKILL.md` | **先读项目 quickref**，再读 [quick-check.md](quick-check.md) → 本 skill 细则 |
| 不存在 quickref，但根目录 `CLAUDE.md` / `AGENTS.md` **已写明**响应类、分页、禁改模块、根包 | **向用户说明**缺少 quickref，建议按本文脚手架；用户未反对时可**临时**以 CLAUDE 为画像继续，**并在回复中提醒**尽快补 quickref |
| 不存在 quickref，且画像信息不足（陌生仓、无 CLAUDE、仅靠猜） | **暂停业务代码改动**；**明确提示用户**需创建 quickref（或至少补全 `CLAUDE.md`）；可提供按 [最小模板](#最小-skillmd-模板) 生成的草稿路径，**等用户确认后再改代码** |
| 用户明确说「无 quickref，直接改」 | 可读 [versions.md](versions.md) + `pom`/`gradle` + 邻代码；仍须在交付说明中标注「未配置项目 quickref，类名/分页以邻代码为准」 |

**禁止**：在缺少项目画像时，把 magApi 或其他仓库的 `ApiResult`、`PageInfo`、包名、业务词**硬套**到当前仓库。

### 建议提示用户的原文（可略改）

```text
本仓库尚未发现项目 quickref（.cursor/skills/*-quickref/SKILL.md）。
继续改 Java/XML/YAML 容易猜错响应类、分页与禁改模块。

建议：
1. 按 java-dev-conventions/project-quickref.md 创建 <项目名>-quickref
2. 或先补全根目录 CLAUDE.md 中的技术栈与核心 API

需要我按模板生成 quickref 草稿吗？确认前我将暂停业务代码改动。
```

---

## 探测路径

按优先级查找（任一命中即视为「已有 quickref」）：

```text
<repo>/.cursor/skills/*-quickref/SKILL.md
<repo>/.agents/skills/*-quickref/SKILL.md
<repo>/.codex/skills/*-quickref/SKILL.md
```

也可读 `.cursor/rules/*.mdc` 是否已声明具体 quickref  skill 名。

**不算** quickref：单独的 `java-dev-conventions`、全局 `~/.agents/skills/` 下仅有通用 skill、其他项目复制来的 `magapi-quickref`（应为本仓新建，勿跨仓复制）。

---

## 创建步骤

### 1. 安装通用 skill

```text
<repo>/.cursor/skills/java-dev-conventions/   # 复制或软链到 ~/.agents/skills/java-dev-conventions
```

### 2. 建项目 quickref 目录

```text
<repo>/.cursor/skills/<project>-quickref/SKILL.md
```

`<project>` 用小写短名（仓库名或产品名），如 `myshop-quickref`。

### 3. 从仓库收集信息（填模板前）

| 来源 | 提取什么 |
|------|----------|
| `pom.xml` / `build.gradle` | JDK、Spring Boot、ORM、安全、文档工具版本 |
| 目录结构 | 可改模块 vs **禁改**模块 |
| 现有 Controller / 基类 | 统一响应、异常、登录态、分页入参/出参 |
| `git log` | `type(scope)` 与 scope 是否中文业务域 |
| 团队约定 | `@author` 默认值、金额单位（元/分） |

### 4. 编写 `SKILL.md`

- YAML `description`：写清 **【必触发】** 范围、**触发词**、**须再加载 java-dev-conventions**
- 正文：只写**本仓独有**信息；通用分层/注释/N+1 **链到**本 skill 对应文件，勿整段复制
- 末尾：**其他 Java 项目不要复制本 skill**

完整结构见 [正文建议章节](#正文建议章节) 与 [最小模板](#最小-skillmd-模板)。

### 5. （推荐）Cursor rule

```text
<repo>/.cursor/rules/java-skills-required.mdc
```

`globs` 覆盖 `**/*.{java,xml,yml,yaml}`；强制顺序：项目 quickref → `java-dev-conventions`（先 `quick-check`）。示例见 [Cursor rule 模板](#cursor-rule-模板)。

### 6. （可选）根目录画像

```text
<repo>/CLAUDE.md   或   <repo>/AGENTS.md
```

5～20 行即可：JDK、启动方式、禁改模块一句话；与 quickref 冲突时 **quickref / CLAUDE 优先于** 通用 skill。

---

## 目录树（最小）

```text
<repo>/
├─ CLAUDE.md                          # 可选
├─ AGENTS.md                          # 可选（非 Cursor 工具）
├─ .cursor/
│  ├─ rules/
│  │  └─ java-skills-required.mdc     # 可选，推荐
│  └─ skills/
│     ├─ java-dev-conventions/        # 通用（软链或复制）
│     └─ <project>-quickref/
│        └─ SKILL.md                  # 项目专属，通常单文件即可
```

---

## 正文建议章节

按需裁剪；第一版 **80～150 行** 通常够用。

| 章节 | 内容 |
|------|------|
| 适用范围 | 链到 `java-dev-conventions`；声明仅本仓库 |
| 技术栈 | 版本锁定表（禁止 Agent 擅自升级） |
| 模块与禁改 | 哪些目录可改、哪些非必要不改 |
| 包结构与核心 API | 根包、响应类、异常、登录态、分页写法（**真实类名**） |
| Controller / 事务 / 注入 | 跟邻代码的硬性约定 |
| 本仓差异 | 金额口径、幂等、权限、注释 `@author`、Git 中文 scope 等 |
| 业务词汇 | 术语表 |
| 命名约定 | Entity / Service / Vo 示例 |
| 何时使用 / 30 秒自检 | 5～6 条勾选项 |

---

<a id="最小-skillmd-模板"></a>

## 最小 SKILL.md 模板

复制后全局替换：`MyApp`、`myapp`、`com.example`、`backend/`、`Result` 等。

```markdown
---
name: myapp-quickref
description: >-
  【必触发】在 MyApp 动手写/改 backend/、common/ 下 Java、Mapper XML、
  Spring 配置之前必须加载；未加载不得改代码。
  撰写本仓中文业务域 Git 提交说明时也必须使用。
  触发词：MyApp, com.example, Result, PageQuery, OrderService,
  订单, 支付, Git 提交, .java, Mapper.xml。
  加载后须再加载 java-dev-conventions（先其 quick-check）。
  不适用：其他 Java 仓库。
---

# MyApp 项目速查

**仅适用于本仓库**。通用 Java 约束见 [java-dev-conventions/SKILL.md](../java-dev-conventions/SKILL.md)——不在此重复判空表、Stream 细则、注释分级表。

## 技术栈（版本锁定）

| 项 | 值 |
|---|---|
| Java | （填） |
| Spring Boot | （填） |
| ORM | （填） |

## 模块与禁改

| 模块 | 说明 |
|------|------|
| `backend/` | 业务主模块 |
| `legacy/` | **非必要不改** |

## 包结构与核心 API

- 根包：`com.example.app`
- 响应：`com.example.common.Result` → `Result.ok(data)` / `Result.fail(msg)`
- 异常：（填）
- 分页：（填真实写法，链到 java-dev-conventions/examples.md#分页 若适用）

## Controller / 事务（本仓）

- （URL 前缀、注入方式、写操作事务注解）
- **禁止 Entity 直接作为 API 入参/出参**

## Git 提交说明（本仓库）

- `scope`：（跟 git log，如中文业务域）
- 格式见 [java-dev-conventions#Git 提交说明](../java-dev-conventions/SKILL.md#git-提交说明)

## 业务词汇

| 术语 | 含义 |
|------|------|
| （填） | （填） |

## 30 秒本仓自检

1. 响应/分页/异常类名是否跟邻代码一致？
2. 是否误改禁改模块？
3. 金额/状态/幂等是否按本仓口径？
4. 注释与 Git scope 是否符合本仓约定？

其他 Java 项目只复制 `java-dev-conventions/`，**不要复制本 skill**。
```

---

<a id="cursor-rule-模板"></a>

## Cursor rule 模板

保存为 `.cursor/rules/java-skills-required.mdc`（skill 名换成实际 quickref）：

```markdown
---
description: 新增/修改 Java、Mapper XML、YAML 时必须先 invoke 项目 quickref 与 java-dev-conventions
globs: "**/*.{java,xml,yml,yaml}"
alwaysApply: false
---

# 代码变更：规范 Skills 必读（强制）

在本仓库**新增或修改**匹配上述 glob 的文件时，动笔前 **MUST** 按序 Read / invoke：

1. `.cursor/skills/<project>-quickref/SKILL.md` — 本仓画像、禁改模块、核心 API、业务词
2. `.cursor/skills/java-dev-conventions/SKILL.md` — 先 `quick-check.md`，再按改动类型打开 1～2 个细则

冲突时：`CLAUDE.md` / 项目 quickref / 本 rule **优先于** `java-dev-conventions`。

纯概念问答、不改代码：可不加载。
```

非 Cursor 工具：在 `AGENTS.md` 写一句「改 Java/XML/YAML 前先读 `.cursor/skills/<project>-quickref/SKILL.md`」。

---

## 与 CLAUDE.md 的分工

| 载体 | 适合放什么 |
|------|------------|
| **quickref** | 类名、分页代码片段、模块禁改、业务词、本仓自检、Git scope |
| **CLAUDE.md** | 如何启动、测试命令、环境变量、「这仓库是什么」一句话 |
| **java-dev-conventions** | 跨项目不变的编码与审查底线 |

有 quickref 后，Agent **优先 quickref**；`CLAUDE.md` 作补充，不重复整段 API 表。

---

## 验收清单

创建或更新 quickref 后自检：

- [ ] YAML `description` 含【必触发】、触发词、须加载 `java-dev-conventions`
- [ ] 技术栈版本与 `pom`/`gradle` 一致
- [ ] 响应类、分页、异常、登录态为**本仓真实全限定名**
- [ ] 未把其他项目（如 magApi）的类名/业务词写进来
- [ ] 通用规则用链接指向 `java-dev-conventions`，未大段重复
- [ ] （推荐）已配 `.mdc` 或 `AGENTS.md`
- [ ] 末尾声明「其他项目勿复制本 skill」

---

## 维护

- 新发现 Agent 常猜错的点 → 补进 quickref，**不要**补进 `java-dev-conventions`
- quickref 单文件过长（>300 行）时，再拆子文件并在 `SKILL.md` 索引；多数项目单文件即可
