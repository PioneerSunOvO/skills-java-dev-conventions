---
name: java-dev-conventions
description: >-
  Java 企业级后端通用编写与审查约束，适用于 Spring Boot / Spring Cloud 单体或多模块项目。
  新增代码须自然中文注释（去 AI/机翻味）、类注释含创建人/时间/职责；
  Mapper XML 每个 SQL 方法上方写业务口径注释；YAML 按段分割符划模块并注释非自解释项。
  覆盖分层、Param/DTO/VO、MapStruct、MyBatis/MyBatis-Plus/JPA、方法步骤分块、数据访问、Git 提交说明。
  动手前从 pom/gradle 探测 Java 版本、Spring 代际、是否 Spring Cloud、持久层栈并对齐邻代码。
  项目 CLAUDE.md、AGENTS.md、.cursor/rules 及项目专属 skill 优先于本 skill。
---

# Java 开发约束

国内企业级 Java 后端（Spring Boot / Spring Cloud、MyBatis/MyBatis-Plus/JPA）的**跨项目通用**编写与审查约束。供 Cursor、Claude Code、Codex 等 Agent 及人工 Code Review 使用。

**通用分层与质量底线**写在本文件；版本差异见 [versions.md](versions.md)；分层细节见 [architecture.md](architecture.md)。项目专属约定（包名、响应类、提交风格细节）放在各仓库 `CLAUDE.md` 或项目专属 skill，**不写入本 skill 正文**。

## 适用边界

| 项 | 说明 |
|----|------|
| 默认栈 | Java 8–21、Spring Boot 2.x/3.x、可选 Spring Cloud；MyBatis / MyBatis-Plus / JPA |
| 部署形态 | 单体、多模块 Maven/Gradle、微服务（Feign / Gateway 等，见 versions.md） |
| 不假设 | 包名、基类、统一响应类、依赖版本——以**当前仓库**为准 |
| 项目规则优先 | `CLAUDE.md`、`AGENTS.md`、`.cursor/rules`、项目专属 skill 冲突时服从项目 |
| 改动范围 | 只动任务相关业务模块；不动框架底层、安全底层、敏感配置，除非用户明确要求 |

## 规则分层

| 类型 | 说明 | 代表条目 |
|------|------|----------|
| **版本无关** | 换 JDK、Spring 代际、持久层仍适用 | 分层、注解从短到长、步骤分块、Java/XML/YAML 注释、禁止 N+1 |
| **版本相关** | 须先完成 [versions.md](versions.md) 环境画像 | Java 语法上限、javax/jakarta、MP/MyBatis/JPA、Spring Cloud 组件、注入与分页 |

画像未明确前，不生成高版本语法、jakarta 包名或项目未使用的 API。

## 执行顺序

```
- 读 versions.md     → 环境画像（Java、SB 代际、Spring Cloud、持久层、javax/jakarta）
- 读邻代码           → 命名、分层、注入、判空、注释详略、返回体
- 定边界             → 入参 Param/DTO、出参 VO、Entity 走 MapStruct
- 写实现             → architecture.md + data-access.md
- 方法结构           → method-structure.md
- 补注释             → comments.md + data-access.md（XML）+ config-yml.md
- 写提交说明         → 本节「Git 提交说明」+ examples.md（用户要求提交时）
- 交付自检           → pitfalls.md 完整清单
```

## 新增类注释（硬性门禁）

每新增 Java 类必须含创建人、时间、职责。**模板与变体以 [comments.md#类注释硬性](comments.md#类注释硬性) 为唯一正文**；项目 `CLAUDE.md` 有专属模板时服从项目。

缺 `@description`（或首行业务说明）、`@author`、`@since` → 不合格。

## 分层速查

| 层 | 只做 | 禁止 |
|----|------|------|
| Controller | 入参校验、调 Service、封装统一响应 | 业务判断、直调 Mapper/Repository、拼复杂 VO |
| Service | 业务编排、事务、领域规则 | 解析 HTTP、拼 SQL 字符串 |
| ServiceImpl | 实现接口；可调其他 Service、Mapper | 越层访问 Web 上下文（除非项目已有封装） |
| Mapper / Repository | 数据读写 | 业务分支、事务注解 |
| Convert | DTO ↔ Entity、简单字段映射 | 含业务 if/else |
| Entity | 表字段映射 | API 入参或出参 |
| DTO / Param | 写操作或非分页入参 | 仅出参字段 |
| VO | 读操作出参 | 写库对象 |

微服务间调用走 Feign / RestTemplate 等项目既有方式，不在 Controller 直调外库 Mapper。详见 [architecture.md](architecture.md)。

## 编码硬性约束

### 空值与集合

跟随项目既有写法，见 [coding-patterns.md](coding-patterns.md)。

### 异常与日志

- 可预期业务失败：抛项目统一业务异常（如 `BusinessException`）
- 禁止空 `catch`、禁止 `catch` 后 `return null` 吞异常
- `@Slf4j`；`log.error("说明, key={}", val, e)` 保留堆栈；禁止 `System.out` / `printStackTrace`

### 事务

- 写操作：`@Transactional(rollbackFor = Exception.class)`（或项目等价配置）
- 分布式场景注意本地事务边界；跨服务一致性在注释中说明（消息、幂等、补偿）

### 依赖注入

见 [versions.md](versions.md)——**不引入项目没有的新注入风格**。

### API 与校验

- HTTP 方法、路径风格跟同模块 Controller
- 校验：`@Validated` 分组 + `@Valid` 嵌套
- 统一响应包装；**禁止直接返回 Entity**
- `javax` / `jakarta` 与 Spring Boot 代际一致

### 注解排列与长行换行

见 [coding-patterns.md](coding-patterns.md)。

## 数据访问要点

- 单表简单条件：按画像选 Wrapper / Repository / XML
- 多表、聚合、动态展示：**Mapper XML（或等价）一次返 VO**
- 分页走项目封装，禁止全量查再内存截取
- 禁止 `for` / `stream` 内单条查库（N+1）

详见 [data-access.md](data-access.md)。

## 中文注释（摘要）

注释覆盖 **Java、Mapper XML、YAML** 三类产物。语气、去 AI/机翻味以 [comments.md](comments.md) 为**唯一权威**。

| 产物 | 位置 | 写什么 |
|------|------|--------|
| Java | 类 / Javadoc / 步骤块 / 块内行 | 职责、边界、幂等、编排步骤 |
| Mapper XML | SQL 标签正上方 | 查什么、过滤口径、表关联与性能取舍 |
| YAML | 段落分割符 + key 旁 | 本段作用、用途、单位、环境差异 |

**何时可省略**：纯重命名、单行 typo、仅格式化且无语义变化时，不必强加业务注释。

<a id="git-提交说明"></a>

## Git 提交说明（通用）

与代码注释互补。具体 type 前缀、模块命名以**仓库既有 `git log` 风格**为准；无惯例时用下列通用结构。

### 标题

```
<type>(<scope>): <一句话说明变更结果>
```

| 字段 | 说明 |
|------|------|
| `type` | 常见：`feat` `fix` `refactor` `perf` `docs` `chore`；跟邻 commit |
| `scope` | 模块或子系统名；可用英文包段或中文业务域，与仓库一致 |
| 一句话 | 说清**对外可见的变化**，避免堆砌类名 |

### 正文

多项独立改动时，正文用 **`-` 列表**分条（不用阿拉伯数字编号）：

```
- <能力或规则变化 1>
- <能力或规则变化 2>
```

每条写：**做了什么 → 带来什么效果**。不要复述 diff 行号。

### 用语

- 优先业务可读表述；技术术语仅在读者是开发且必要时保留
- 可保留：表名、接口路径、业务状态码——便于对号入座
- 避免：无信息量类名堆砌、空泛「优化代码」「修复 bug」

### 范围与安全

- 用户指定「只提交 Java」时，提醒配套 `Mapper.xml`、迁移脚本等是否遗漏
- 不提交：`.env`、密钥、含真实密码的配置、无关二进制
- 未要求不主动 commit

正反例见 [examples.md#git-提交说明](examples.md#git-提交说明)。项目专属提交风格见各仓库 `CLAUDE.md` 或项目 skill。

## 交付前自检（精简）

完整清单见 [pitfalls.md#十交付清单完整](pitfalls.md#十交付清单完整)。核心项：

- 环境画像已完成（含 Spring Cloud 与否）
- 分层正确；无 Entity 穿透 API；无 N+1
- 新增类注释、方法 Javadoc、步骤分块达标
- Mapper XML / YAML 注释与分割符达标
- 注释自然中文，无机翻味、无 AI 套话
- 未改敏感配置、未升依赖、未动用户未要求模块

## 参考文件索引

| 文件 | 内容 |
|------|------|
| [versions.md](versions.md) | 环境探测、Java 语法、SB2/SB3、**Spring Cloud**、持久层/注入/分页 |
| [architecture.md](architecture.md) | 分包、Controller/Service、DTO/VO、URL、审计、测试、定时任务、安全边界 |
| [data-access.md](data-access.md) | SQL/XML 注释、分页、事务、金额时间 |
| [config-yml.md](config-yml.md) | YAML 段落分割符、段内注释 |
| [comments.md](comments.md) | Java 注释、类注释模板、语气与 AI 套话 |
| [method-structure.md](method-structure.md) | 步骤分块、私有方法抽取 |
| [coding-patterns.md](coding-patterns.md) | 判空、Stream、MapStruct、注解排列 |
| [pitfalls.md](pitfalls.md) | 陷阱表、反面教材、**完整交付清单** |
| [examples.md](examples.md) | 版本探测、注释润色、Git 提交示例 |

## 跨项目复用

复制整个 `java-dev-conventions` 目录到目标项目的 `.cursor/skills/`、`.claude/skills/` 或团队约定的 skills 目录。项目专属内容单独维护项目 skill，不并入本 skill。
