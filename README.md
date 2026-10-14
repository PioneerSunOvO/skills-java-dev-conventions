# java-dev-conventions

跨项目通用的 Java 后端（**Spring Boot 2.x/3.x**、可选 **Spring Cloud**；持久层 **MyBatis / MyBatis-Plus / JPA**；及其他常见国内组件）编写与审查约束 Skill，供 Claude Code、Cursor、Codex 等 Agent 使用。

## 能力概览

- **分层边界**：Controller / Service / Mapper（或 Repository）、Param/DTO 入参、VO 出参
- **版本无关核心**：注解从短到长、长行换行、步骤分块、中文注释分级表
- **版本自适应**：`versions.md` 探测单体与 Cloud；项目 rules 已写明画像时可走捷径
- **存量大文件**：`incremental-change.md` 指导超大 ServiceImpl 局部修改
- **数据访问**：分页多分支、MP 动态条件、回调幂等、金额时间、并发模板
- **测试门禁**：`tests-policy.md` 规定何时必须补验证、优先补哪层
- **安全门禁**：`security.md` 约束鉴权、数据范围、脱敏、动态字段白名单
- **速查入口**：`quick-check.md` 提供按改动类型的 30 秒自检
- **模块落点 / 工程骨架**：`architecture.md`（含 **多模块工程骨架**：Starter / SPI / 改底座门槛）
- **横切高阶**：`framework-patterns.md`（Context+Utils、注解+Aspect+策略、Handler、CAS 封装、下沉门槛）
- **YAML**：`config-yml.md` 段分割符与配置项中文注释
- **交付自检**：`pitfalls.md` 完整清单
- **迁移升级**：`migration.md` 处理 Boot / Cloud / `javax`→`jakarta`

## 安装

### 从 GitHub（推荐）

```bash
npx skills add <owner>/skills-java-dev-conventions -g --agent claude-code cursor -y --copy
```

### 手动复制

| 范围 | 路径 |
|------|------|
| 全局 | `~/.agents/skills/java-dev-conventions/` 或 `~/.cursor/skills/` |
| 项目 | `<repo>/.cursor/skills/java-dev-conventions/`（可软链到全局） |

## 使用

```
@java-dev-conventions
```

或自然语言：「按 java-dev-conventions 新增 OrderController」。

## 多项目协作：推荐分层

本 Skill **保持跨项目通用**。每个业务仓库另建**项目专属 skill**（可选），职责划分：

| 层级 | 放什么 | 示例 |
|------|--------|------|
| **本 Skill** | 分层、注释、判空、SQL、事务、Cloud 分支原则 | `java-dev-conventions` |
| **项目 skill** | 版本锁定、统一响应类名、模块禁改、业务词汇、Git scope | `xxx-quickref` |
| **项目 rules** | `alwaysApply` 门禁、调用顺序、作者署名 | `.cursor/rules/*.mdc` |
| **CLAUDE.md / AGENTS.md** | 硬性禁改、环境说明 | 仓库根目录 |

**推荐 invoke 顺序**（有项目 skill 时）：

```text
Superpowers 流程 skill（若启用）→ 项目 skill → java-dev-conventions
```

冲突时：**项目 rules / 项目 skill > 本 Skill > Superpowers 默认行为**。

## 支持矩阵

| 类别 | 状态 | 说明 |
|------|------|------|
| Spring Boot 2.x / 3.x | **支持** | 默认主场景 |
| Spring Cloud | **支持** | Feign / Gateway / 注册中心 / 配置中心 / 熔断按画像分支处理 |
| MyBatis / MyBatis-Plus / JPA | **支持** | `versions.md` + `data-access.md` 分支 |
| Swagger2 / springdoc | **支持** | 不混用，跟项目探测结果；**接口 Param/Vo 字段必须有文档注解** |
| Jackson / Fastjson / Gson | **支持** | 跟项目现有主序列化体系 |
| Hutool / Guava / Commons | **支持** | 跟邻代码，不引入第二套重叠依赖 |
| Dubbo | **不直接覆盖** | 以项目规则与邻代码为准 |
| Quarkus / Micronaut / Vert.x | **不直接覆盖** | 非 Spring 容器，不强套本 Skill |
| 纯 WebFlux 全链路 | **部分覆盖** | 注释/分层/边界可复用；响应式 API 细节不在本文规范 |

### 新项目接入清单

1. 复制或软链 `java-dev-conventions` 到全局或项目 `.cursor/skills/`
2. 在仓库写 `CLAUDE.md`：JDK、Spring Boot/Cloud、持久层、禁改模块（可选但推荐）
3. **创建项目 quickref**（业务仓推荐、非可选）：`<project>-quickref/SKILL.md` — 步骤、模板、缺失时 Agent 行为见 **[project-quickref.md](project-quickref.md)**
4. （推荐）`.cursor/rules/` 声明 quickref 与 java-dev-conventions 的加载顺序
5. 首次改 Java 前：有 quickref 则先读它；**没有则 Agent 应提示用户创建**（见 project-quickref.md），勿硬套其他仓库类名

### 最小模板目录树

```text
<repo>/
├─ CLAUDE.md
├─ .cursor/
│  ├─ rules/
│  │  └─ java-dev.mdc
│  └─ skills/
│     ├─ java-dev-conventions/
│     └─ <project>-quickref/
│        └─ SKILL.md
```

建议分工：

- `CLAUDE.md`：JDK、Spring Boot/Cloud、持久层、禁改模块、运行方式
- `java-dev-conventions`：通用编码与审查底线
- `<project>-quickref`：响应包装类、分页类、包名、业务词、模块别名
- `java-dev.mdc`：默认启用顺序与优先级

### 示例：项目 rules 片段（`.mdc`）

```markdown
---
description: 默认启用 Java 规范与项目速查
alwaysApply: true
---

# Java 开发

会话默认遵循：

1. **`<project>-quickref`** — 本仓库 API、模块、业务词汇
2. **`java-dev-conventions`** — 通用分层与编码底线

冲突时：`CLAUDE.md` / 项目 quickref > java-dev-conventions。
```

将 `<project>-quickref` 换成实际 skill 名；**不要把某一仓库的类名写进 `java-dev-conventions` 正文**。

完整创建指南（探测路径、最小模板、Cursor rule、缺失时提示用户）：**[project-quickref.md](project-quickref.md)**。

## 文件结构

```
SKILL.md                  # 入口：路由 + 硬门禁
versions.md               # 环境探测：Java、SB2/3、Cloud、持久层、老旧依赖
architecture.md           # 分层、DTO/VO、模块落点、多模块工程骨架、契约、Swagger、错误码
incremental-change.md     # 存量大文件增量修改
coding-patterns.md        # 判空、Stream、多工具库并存、MapStruct 权威
comments.md               # 注释权威（规则）
comments-examples.md      # 占位词/套话/形状模板/润色对照（按需）
method-structure.md       # 步骤分块、私有方法
data-access.md            # SQL/XML、分页引用、MP 动态条件、回调幂等
tests-policy.md           # 测试触发条件、验证层级、手工验证退化条件
security.md               # 鉴权、数据范围、脱敏、动态字段白名单
quick-check.md            # 默认入口：按改动类型选读
config-yml.md             # application*.yml
pitfalls.md               # 陷阱与完整交付清单
examples.md               # 版本/语法/分页/MapStruct 对照（按需）
examples-method-structure.md # 步骤分块完整范本
migration.md              # 升级与迁移
project-quickref.md       # 项目 quickref 创建；未发现时提示用户
```

## 维护与更新

本目录可能是独立 Git 仓库（如 `skills-java-dev-conventions`）。建议：

| 操作 | 说明 |
|------|------|
| **跟踪上游** | `git remote -v` 确认来源；`git pull` 合并上游通用改进 |
| **本地定制** | 在 fork 或分支上改；避免直接改全局后又用 `npx skills add` 覆盖 |
| **项目差异** | 业务词、类名、禁改模块写在**项目 skill / CLAUDE.md**，不写进通用正文 |
| **版本记录** | 较大改动在 README 或 commit message 说明，便于多仓同步 |

若使用 `npx skills add` 安装，更新前备份本地对 `java-dev-conventions` 的修改。

## 与 Superpowers 的关系

本 Skill 管**编码质量**；Superpowers 管**流程**（brainstorming、writing-plans、TDD 等）。局部小改例外见 [SKILL.md#局部小改例外](SKILL.md#局部小改例外与官方-brainstorming-并存)。

## License

MIT
