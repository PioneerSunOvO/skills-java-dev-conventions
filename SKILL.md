---
name: java-dev-conventions
description: >-
  Use when adding or modifying Java/Spring Boot backend code: Controller, Service,
  Mapper, Repository, Entity, DTO/Param/VO, SQL/XML, or REST endpoints. Also when
  reviewing Java changes, refactoring layers, or user mentions Java规范, 分层,
  中文注释, 判空, Optional, Objects.isNull, CollectionUtils, StringUtils, == null,
  集合判空, Stream, stream流, 流式处理, MyBatis, MapStruct, or Spring Boot 2→3 migration. Covers MyBatis,
  MyBatis-Plus, and JPA stacks. Not for pure Java Q&A without code edits. Project
  CLAUDE.md, AGENTS.md, and .cursor/rules override this skill.
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
7. 交付自检        → pitfalls.md 清单
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
| `@author` | 创建人真实姓名或项目约定署名；**禁止** Cursor、Codex、Auto、Claude、GPT、Copilot 等 AI/工具名（见 [comments.md#author-取值](comments.md#author-取值)） |
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

### 空值与集合（硬性）

**新代码禁止**手写 `== null` / `!= null` 判对象；**禁止**对可能为 null 的集合/Map 直接 `.isEmpty()` / `.size() == 0`。一律用工具类，细则见 [coding-patterns.md#判空与工具类](coding-patterns.md#判空与工具类)。

| 场景 | 必须用 | 禁止 |
|------|--------|------|
| 对象 null | `Objects.isNull()` / `Objects.nonNull()` | `== null`、`!= null` |
| 字符串空白 | `StringUtils.isBlank()` / `isNotBlank()` | `str == null \|\| str.isEmpty()` |
| 集合/Map 空 | `CollectionUtils.isEmpty()` / `isNotEmpty()` | `list == null \|\| list.isEmpty()` |
| 等值比较 | `Objects.equals(a, b)` | `a != null && a.equals(b)` |
| 查无则抛 | `Optional.ofNullable(x).orElseThrow(...)` | 多层 if 后 throw |
| 查无给默认 | `Optional.ofNullable(x).orElse(def)` | 冗长三元 `x != null ? x : def`（简单字面量三元可保留） |

```java
if (Objects.isNull(order)) {
    throw new BusinessException("订单不存在");
}
if (CollectionUtils.isEmpty(ids)) {
    return Collections.emptyList();
}
Order order = Optional.ofNullable(getById(id))
        .orElseThrow(() -> new BusinessException("订单不存在"));
```

- `StringUtils` / `CollectionUtils` 的 **import 包名跟邻文件**（`lang3`、`collections` vs `collections4`）
- `Optional` 仅用于「可能为空的一次取值 + 抛异常/默认值」；**禁止**层层嵌套防御
- 改老代码时邻段若已是 `== null` 或 Hutool `ObjectUtil`，可对齐邻段，但**新写片段**仍按上表
- JDK 语法上限见 [versions.md](versions.md)（如 Java 8 禁用 `String.isBlank()`）

### Stream 流式处理

**可读性优先**：能明显简化集合转换/过滤/聚合时用 Stream；不要为了「函数式」牺牲可读性。细则见 [coding-patterns.md#stream-流式处理](coding-patterns.md#stream-流式处理)。

| 适合 Stream | 保留 for / 普通循环 |
|-------------|---------------------|
| `map` / `filter` / `collect` 做列表转换 | 含复杂分支、多处 `continue`/`break` |
| `groupingBy` / `toMap` / `sum` 聚合统计 | 循环体内有副作用（写库、改外部变量） |
| 邻代码已普遍用 Stream 的同类转换 | 2～3 行简单遍历，for 更直观 |
| 纯内存、无 I/O 的数据变换 | 需要下标、或逐步早返回的业务校验 |

- **禁止**在 `stream` / `forEach` 内单条查库或远程调用（N+1）
- 链式操作 **≤ 4 步**为宜；更长则拆变量或抽私有方法
- 不用 `peek` 写业务逻辑；不用嵌套 Stream 炫技

### 异常与日志

- 可预期业务失败：`throw new BusinessException("简短中文")`（或项目等价类）
- 禁止空 `catch`、禁止 `catch` 后 `return null` 吞异常
- `@Slf4j`；`log.error("说明, key={}", val, e)` 保留堆栈；禁止 `System.out` / `printStackTrace`
- `try-catch` 统一捕获 `Exception`，不捕获具体子异常（除非项目或用户明确指定）
- 捕获后需使用异常工具类打印完整堆栈（如 `ExceptionUtils.printRootCauseStackTrace(e)`），禁止只打印 `e.getMessage()`

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

注释是**硬性约束**，**权威细则见 [comments.md](comments.md)**（一页速查、分级表、三层分工、占位词禁用法、30 秒自检）。

**必须写**：类三要素、分级表中的「必须」项（公开业务方法、配置/Document/对外 VO 字段等）。

**不要写**：复述代码、AI 套话/占位词（下面/补齐/构建索引）、机械全覆盖 Javadoc、用英文字段名当字段注释。

**步骤分块**：触发条件见 [comments.md#步骤分块](comments.md#步骤分块)；一行 Controller 委托不必分块。

## 交付前自检

```
[ ] 已按 versions.md 完成环境画像（Java 语法、javax/jakarta、持久层 API）
[ ] 分层正确；Controller 无业务、无直调 Mapper/Repository
[ ] 入参 Param/DTO；出参 VO；Entity 不进 API；MapStruct 做边界转换
[ ] 无 N+1；复杂查询走 XML/等价方式返 VO
[ ] 注释：已按 comments.md 分级表、30 秒自检与占位词禁用法
[ ] 判空：无 `== null`/`!= null` 判对象；集合用 `CollectionUtils`；字符串用 `StringUtils`
[ ] Stream：转换/聚合能简化则用；无 N+1、无过度嵌套；可读性不如 for 时保留 for
[ ] 同类声明处多注解已按文本从短到长排列（见 coding-patterns.md）
[ ] 未改敏感配置、未升依赖、未动用户未要求的模块
```

完整清单：[pitfalls.md](pitfalls.md)。

## 参考文件索引

| 文件 | 内容 |
|------|------|
| [versions.md](versions.md) | 环境探测、Java 8–21 语法、SB2/SB3、持久层/注入/分页分支 |
| [architecture.md](architecture.md) | 分包、Controller/Service、DTO/VO、URL、审计字段 |
| [data-access.md](data-access.md) | SQL/XML、分页、事务一致性、金额时间 |
| [comments.md](comments.md) | **注释权威**：一页速查、分级表、三层分工、占位词禁用法、并行查询模板、30 秒自检 |
| [method-structure.md](method-structure.md) | 步骤分块模板、按块分层、私有方法抽取 |
| [coding-patterns.md](coding-patterns.md) | 判空工具、Stream、MapStruct、注解排列、Service/AOP 习惯 |
| [pitfalls.md](pitfalls.md) | 陷阱表、AI 代码气味、交付清单 |
| [examples.md](examples.md) | 版本探测、语法降级、步骤分块、MapStruct 示例 |

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
