---
name: java-dev-conventions
description: >-
  新增 Java 代码必须写自然中文注释（去 AI/机翻味）、类注释含创建人/时间/职责；
  企业级 Java 后端编写与审查约束，覆盖 Spring Boot 2.x/3.x、MyBatis/MyBatis-Plus/JPA
  分层、Param/DTO 入参、VO 出参、MapStruct 转换、方法步骤分块与数据访问。
  动手前从 pom/gradle 探测 Java 版本、javax/jakarta 与持久层栈并对齐邻代码。
  版本无关核心（分层、注解排列、长行换行、注释、N+1 底线）跨项目/JDK/Spring 代际不变；
  版本相关条目见 versions.md 分支对照。
  适用于新增或重构 Controller/Service/Mapper、编写 SQL/XML、代码审查、
  写 Git 提交说明、提交代码，或用户提到 Java 规范、分层、MyBatis、注释规范、
  提交风格时。项目 CLAUDE.md、AGENTS.md、.cursor/rules 优先于本 skill。
---

# Java 开发约束

国内企业级 Java 后端（toB/toC）的编写与审查约束。**通用分层与质量底线**写在本文件；版本差异、持久层分支见 [versions.md](versions.md)；分层细节见 [architecture.md](architecture.md)。

## 适用边界

| 项 | 说明 |
|----|------|
| 默认栈 | Java 8–21、Spring Boot 2.x/3.x 或等价 Spring 栈、MyBatis / MyBatis-Plus / JPA |
| 不假设 | 包名、基类全限定名、统一响应类名、依赖版本——以**当前仓库**为准 |
| 项目规则优先 | `CLAUDE.md`、`AGENTS.md`、`.cursor/rules` 冲突时服从项目 |
| 改动范围 | 只动任务相关业务模块；不动框架底层、安全底层、敏感配置，除非用户明确要求 |

## 规则分层（跨项目 / 跨版本）

约束分两类。**切换项目、JDK、Spring Boot / Cloud 版本时，版本无关规则始终有效**，不因栈变化而废弃或改写。

| 类型 | 说明 | 代表条目 |
|------|------|----------|
| **版本无关（核心）** | 与 JDK、Spring 代际、持久层选型无关；换仓库仍适用 | 分层边界、注解从短到长、长行换行、步骤分块、注释五层、禁止 N+1、禁止 Entity 穿透 API |
| **版本相关（自适应）** | 须先完成 [versions.md](versions.md) 环境画像，再选具体 API / 包名 | Java 语法上限、javax/jakarta、Swagger2/springdoc、MP/MyBatis/JPA、注入风格、分页封装 |

**执行原则**：先套用版本无关规则；涉及具体注解类名、import、持久层写法时，以**当前仓库画像**为准。**不得**把某一项目的专属注解、基类或响应类写进 skill 正文示例。

## 执行顺序

```
1. 读 versions.md  → 从 pom/gradle 探测并完成环境画像（Java、SB 代际、持久层、javax/jakarta）
2. 读邻代码        → 命名、分层、注入、判空、注释详略、返回体
3. 定边界          → 入参 Param/DTO、出参 VO、Entity 转换走 MapStruct
4. 写实现          → architecture.md + data-access.md（按画像选 API）
5. 方法结构        → method-structure.md（步骤分块、按块分层、按需抽私有方法）
6. 补注释          → comments.md（五层注释、去 AI/机翻味自检）
7. 写提交说明      → 本节「Git 提交说明」+ examples.md 提交示例（用户要求提交时）
8. 交付自检        → pitfalls.md 清单
```

**第 1 步不可跳过**：画像未明确前，不生成高版本语法、jakarta 包名或项目未使用的持久层 API。

## 新增文件硬性门禁

**每新增一个 Java 类**，类注释不可缺项。项目有模板时以 `CLAUDE.md` 为准；标准三要素：

```java
/**
 * @description: 订单服务实现
 * @author: 开发者
 * @since: 2026-06-09 10:00
 */
```

| 标签 | 要求 |
|------|------|
| `@description` | 一句话说清**类职责**；写业务口径或边界，不写「查询、更新、删除」列表 |
| `@author` | 创建人 |
| `@since` | 创建时间，格式 `YYYY-MM-DD HH:mm` |

**变体**（切面、工具类、口径复杂的服务）：首行业务说明 + `@author` / `@since`。示例见 [coding-patterns.md](coding-patterns.md)。

缺任一要素、或 `@description` 为空 → 视为不合格。

## 分层速查

| 层 | 只做 | 禁止 |
|----|------|------|
| Controller | 入参校验、调 Service、封装统一响应 | 业务判断、直调 Mapper/Repository、拼复杂 VO |
| Service | 业务编排、事务、领域规则 | 解析 HTTP、拼 SQL 字符串 |
| ServiceImpl | 实现接口；可调其他 Service、Mapper | 越层访问 Web 上下文（除非项目已有封装） |
| Mapper / Repository | 数据读写 | 业务分支、事务注解 |
| Convert / Assembler | DTO ↔ Entity、简单字段映射 | 含业务 if/else |
| Entity | 表字段映射 | API 入参或出参 |
| DTO / Param | 写操作或非分页入参 | 仅出参字段 |
| VO | 读操作出参、展示字段 | 写库对象 |

URL 约定、对象选用、命名见 [architecture.md](architecture.md)。

## 编码硬性约束

### 空值与集合

跟随项目既有写法；推荐习惯见 [coding-patterns.md](coding-patterns.md)。常见模式：

```java
Objects.isNull(x) / Objects.nonNull(x)   // 部分项目禁止 == null，以邻代码为准
StringUtils.isBlank(s)                   // commons-lang3
CollectionUtils.isEmpty(list)            // collections4（以项目为准）
```

链式取值可能为 null 时用 `Optional` 或提前返回。JDK 特性边界见 [versions.md](versions.md)。

### 异常与日志

- 可预期业务失败：`throw new BusinessException("简短中文")`（或项目等价类）
- 禁止空 `catch`、禁止 `catch` 后 `return null` 吞异常
- `@Slf4j`；`log.error("说明, key={}", val, e)` 保留堆栈；禁止 `System.out` / `printStackTrace`

### 事务

- 写操作：`@Transactional(rollbackFor = Exception.class)`（或项目等价配置）
- 不手动 `commit/rollback`；注意同类自调用导致事务失效

### 依赖注入

见 [versions.md](versions.md)——**不引入项目没有的新注入风格**。

### API 与校验

- HTTP 方法、路径风格跟同模块 Controller
- 校验：`@Validated` 分组 + `@Valid` 嵌套，与项目 `validator.groups` 一致
- 统一响应包装；**禁止直接返回 Entity**
- `javax` / `jakarta` 校验包与 Spring Boot 代际一致（见 versions.md）

### 注解排列（版本无关）

同一类、方法、字段、参数等处多个注解时，**按注解文本从短到长**自上而下排列；与 JDK、Spring 代际、文档注解体系无关。详见 [coding-patterns.md](coding-patterns.md#注解排列)。

### 长行换行

续行用固定缩进，不在 `(` 后立刻断行堆参数，不用「与首参数对齐」式换行。详见 [coding-patterns.md](coding-patterns.md#长行换行)。

## 数据访问要点

- 单表简单条件：按画像选 Wrapper / Repository / XML（见 versions.md 持久层分支）
- 多表、聚合、动态展示：**Mapper XML（或等价）一次返 VO**，避免 Service 循环查库拼 Map
- 分页走项目封装，禁止全量查再内存截取
- 禁止 `for` / `stream` 内单条查库（N+1）

详见 [data-access.md](data-access.md)。

## 中文注释（摘要）

注释是**硬性约束**。新增或修改业务代码默认必须补注释。

**必须写**：类职责与边界、业务规则、幂等与冲突、非直观分支、单位与枚举含义。

**不要写**：复述代码、AI 模板句、机翻被动语态、空洞 CRUD 列表。

注释分五层：类 → 方法 Javadoc → **步骤分块**（`// 动词+对象+边界`，块间空一行）→ 块内行注释 → 字段。模板见 [method-structure.md](method-structure.md)、[comments.md](comments.md)。

## Git 提交说明（magApi）

提交说明面向**业务可读**：领导与非后端同事能看懂「改了什么、解决什么问题」，不必懂实现细节。与代码注释规范互补，不互相替代。

### 标题格式

```
<type>(<模块>): <一句话说明业务结果>
```

| 字段 | 要求 |
|------|------|
| `type` | `feat` 新能力；`fix` 修 bug；`refactor` 行为不变的重构；`perf` 性能优化 |
| `模块` | 中文业务域：`订单`、`来款`、`对账`、`结算`、`发行计划`、`全局异常` 等 |
| 一句话 | 说清**用户/业务侧变化**，不写类名、不写技术方案 |

### 正文格式

有**两项及以上**独立改动时，正文用**编号列表**分条（优先 `1. 2. 3.`）：

```
1. <具体能力或规则变化>
2. <第二条>
3. <第三条>
```

单点、偏技术的小改动可用 `-` 列表。每条回答：**做了什么 → 带来什么效果**，不要复述 diff 行号。

### 用语要求

| 推荐 | 避免 |
|------|------|
| 自动同步汇总表、按明细重算 | AOP、切面、注解驱动 |
| 新增汇总表、便于列表和统计查询 | 冗余表、物化视图 |
| 一次关联查询、减少两步查询 | EXISTS、IN 列表、N+1（除非读者是开发且必要） |
| 订单保存、修改、删除、作废 | save/update/remove 等方法名堆砌 |
| 管理端全量重建接口、历史数据补算 | rebuildAll、游标批处理 |

**可保留**：表名（`imo_magorder_subscribe_summary`）、接口路径（`/onlineRefund`）、业务字段（`moType=11`）——便于对号入座。

### 撰写流程

1. `git diff` / `git diff --cached` 看清改动范围
2. 定 `type` 与 `模块`（以主要影响面为准）
3. 标题写**一条最重要的业务结果**
4. 正文按**独立能力**拆条；Mapper XML 与 Java 属同一功能时写在同一条或同一 commit
5. 用户要求「先看一下 message 再提交」时，**先展示全文，确认后再 `git commit`**

### 提交范围

- 用户指定「只提交 Java」时，仍须提醒：配套 `Mapper.xml` 未提交会导致运行失败
- 不提交：`.env`、密钥、`application-*.yml` 中的敏感项、无关 docx（除非用户明确要求）
- 未要求不主动 commit；遵循仓库 `committing-changes-with-git` 规则

正反例与模板见 [examples.md#git-提交说明示例](examples.md#git-提交说明示例)。

## 交付前自检

```
[ ] 已按 versions.md 完成环境画像（Java 语法、javax/jakarta、持久层 API）
[ ] 分层正确；Controller 无业务、无直调 Mapper/Repository
[ ] 入参 Param/DTO；出参 VO；Entity 不进 API；MapStruct 做边界转换
[ ] 无 N+1；复杂查询走 XML/等价方式返 VO
[ ] 新增 Java 类含 @description（或首行业务说明）+ @author + @since
[ ] 方法 Javadoc 含职责、边界、`@param`/`@return`
[ ] 方法体按步骤分块；块间空一行；步骤注释含边界条件
[ ] 复杂分支/口径有块内单行注释；公开 Service 方法以编排为主
[ ] 注释无 AI/机翻味；与邻类风格一致
[ ] 同类声明处多注解已按文本从短到长排列（见 coding-patterns.md）
[ ] 未改敏感配置、未升依赖、未动用户未要求的模块
[ ] 提交说明（若需提交）：标题业务可读、正文编号分条、无 AOP/切面等专业术语堆砌
```

完整清单：[pitfalls.md](pitfalls.md)。

## 参考文件索引

| 文件 | 内容 |
|------|------|
| [versions.md](versions.md) | 环境探测、Java 8–21 语法、SB2/SB3、持久层/注入/分页分支 |
| [architecture.md](architecture.md) | 分包、Controller/Service、DTO/VO、URL、审计字段 |
| [data-access.md](data-access.md) | SQL/XML、分页、事务一致性、金额时间 |
| [comments.md](comments.md) | 五层注释、去 AI/机翻味、正反例 |
| [method-structure.md](method-structure.md) | 步骤分块模板、按块分层、私有方法抽取 |
| [coding-patterns.md](coding-patterns.md) | 判空工具、Stream、MapStruct、注解排列、Service/AOP 习惯 |
| [pitfalls.md](pitfalls.md) | 陷阱表、AI 代码气味、交付清单 |
| [examples.md](examples.md) | 版本探测、语法降级、步骤分块、MapStruct、Git 提交示例 |

## 跨项目复用

复制整个 `java-dev-conventions` 目录到：

- 项目：`<project>/.cursor/skills/`
- 全局：`~/.cursor/skills/`

调用：`/java-dev-conventions` 或 `@java-dev-conventions`。

| 放在 skill 里 | 放在目标项目 `CLAUDE.md` / rules 里 |
|---------------|-------------------------------------|
| 版本无关核心（分层、排版、注释、数据访问底线） | 模块禁改、版本锁定、注入风格、类注释模板 |
| 版本探测流程与分支对照（versions.md） | 具体包名、基类、统一响应类名 |
| 占位类名示例（`OrderController`、`XxxDto`） | 业务域命名、子域分包约定 |

正文示例**禁止**绑定某一仓库的 Controller、自定义框架注解或固定响应类型；换 JDK 8↔21、SB2↔SB3、有无 Spring Cloud 时，核心约束与执行顺序不变，仅 versions.md 所指分支按需切换。
