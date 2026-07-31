---
name: java-dev-conventions
description: >-
  Use when adding or modifying Java/Spring Boot backend code without a project-specific
  skill, or for cross-project review: Controller, Service, Mapper, Entity, DTO/Param/VO,
  SQL/XML, YAML. Triggers: Java规范, 分层, 中文注释, 机翻, AI味, 占位词, 套话, deslop,
  Git 提交, MyBatis, MapStruct, Spring Boot 2→3 migration.
  If the repo has a project-specific skill, use it before this one for API names,
  modules, and business vocabulary.
  NOT for: project-specific conventions → that project's skill; Superpowers workflow only
  → brainstorming/writing-plans; concept Q&A without code changes → answer directly.
  Project CLAUDE.md, AGENTS.md, .cursor/rules, and project skills override this skill.
---

# Java 开发约束

国内企业级 Java 后端（toB/toC）的**跨项目通用**编写与审查约束。**各仓库的项目专属 skill**（若有）负责版本锁定、API 类名、业务词汇等画像；本 skill 负责通用分层与质量底线。冲突时服从项目 rules / 项目 skill。

版本差异、持久层分支见 [versions.md](versions.md)；分层细节见 [architecture.md](architecture.md)。

## Superpowers 6.2.0 衔接

本 skill 是**编码约束**，不替代 Superpowers **流程** skill。二者叠加；**项目 rules / 项目 skill 优先于 Superpowers 默认行为**。

| 阶段 | Superpowers skill | 本 skill |
|------|-------------------|----------|
| 新功能 / 改业务行为 | `brainstorming` →（获批后）`writing-plans` | 设计阶段不写实现代码 |
| 有书面计划的多步实施 | `executing-plans` 或 `subagent-driven-development` | 按计划写代码时 invoke 本 skill |
| 写/改 Java 实现 | — | 本 skill + 项目 skill（若有） |
| 改业务行为且需要回归验证 | `test-driven-development` 或项目既有测试流程 | 测试触发条件见 [tests-policy.md](tests-policy.md) |
| 声称完成 / 可提交 | `verification-before-completion` | 交付自检见 [pitfalls.md](pitfalls.md) |
| bug / 测试失败 | `systematic-debugging` | 定位后再 invoke 本 skill 改代码 |

### 局部小改例外（与官方 brainstorming 并存）

官方 `brainstorming` 默认 **MUST** 先设计。下列**全部满足**时可跳过 `brainstorming` / `writing-plans`，直接 invoke 本 skill（用户可说「局部小改，直接实现」）：

| 条件 | 说明 |
|------|------|
| 单文件或局部 | 改动集中在 1 个类，或配套 1 个 Mapper XML / 配置文件 |
| 规模小 | 预计净增删 **&lt; 50 行**（不含纯格式化） |
| 无新契约 | 不新增 REST 路径、表结构、状态机、跨模块对外 API |
| 方案已定 | 用户已指明改法，或 bug 根因已明确 |

**仍须** `brainstorming`：新接口、新表/字段、核心业务规则变更、多方案取舍、跨模块或跨子系统改动。

**有项目专属 skill 时**：`Superpowers（按上表）→ 项目 skill → 本 skill`（顺序以各仓 `CLAUDE.md` / `.cursor/skills/README.md` 为准）。

### 跨模块 / 跨服务变更的升级条件

以下任一命中，都不再视为“局部小改”，应先走 `brainstorming`，通常还需要 `writing-plans`：

| 信号 | 原因 |
|------|------|
| 新增或修改 REST / RPC / Feign / MQ 契约 | 会影响调用方或消费者 |
| 改共享 DTO / VO / `api` 模块 | 会波及多个模块或服务 |
| 改分布式事务、幂等、补偿策略 | 需要先明确一致性边界 |
| 改注册中心、网关、配置中心行为 | 影响运行时路由或环境配置 |
| 需要在多个模块同时改代码 | 已超出单点修复 |

## 适用边界

| 项 | 说明 |
|----|------|
| 默认栈 | Java 8–21、Spring Boot 2.x/3.x、可选 Spring Cloud、MyBatis / MyBatis-Plus / JPA |
| 不假设 | 包名、基类全限定名、统一响应类名、依赖版本——以**当前仓库**为准 |
| 项目规则优先 | `CLAUDE.md`、`AGENTS.md`、`.cursor/rules` 冲突时服从项目 |
| 改动范围 | 只动任务相关业务模块；不动框架底层、安全底层、敏感配置，除非用户明确要求 |

### 适配边界

本 skill 面向 **Java + Spring 生态**的企业后端，重点覆盖：

- Spring Boot 2.x / 3.x
- 可选 Spring Cloud（OpenFeign / Gateway / Nacos / Eureka / Config / LoadBalancer 等）
- MyBatis、MyBatis-Plus、Spring Data JPA
- 常见配套组件：Swagger2 / springdoc、Jackson / Fastjson / Gson、Hutool / Guava / Commons

**不直接覆盖**：Dubbo、Vert.x、Quarkus、Micronaut、纯响应式 WebFlux 全链路、非 Spring 容器。遇到这些栈时，以邻代码与项目规则为准，不强套本文示例。

## 规则分层（跨项目 / 跨版本）

约束分两类。**切换项目、JDK、Spring Boot / Cloud 版本时，版本无关规则始终有效**，不因栈变化而废弃或改写。

| 类型 | 说明 | 代表条目 |
|------|------|----------|
| **版本无关（核心）** | 与 JDK、Spring 代际、持久层选型无关；换仓库仍适用 | 分层边界、注解从短到长、长行换行、步骤分块、注释五层、禁止 N+1、禁止 Entity 穿透 API |
| **版本相关（自适应）** | 须先完成 [versions.md](versions.md) 环境画像，再选具体 API / 包名 | Java 语法上限、javax/jakarta、Swagger2/springdoc、MP/MyBatis/JPA、注入风格、分页封装 |

**执行原则**：先套用版本无关规则；涉及具体注解类名、import、持久层写法时，以**当前仓库画像**为准。**不得**把某一项目的专属注解、基类或响应类写进 skill 正文示例。

## 执行顺序

```
1. 环境画像        → 见下「versions 捷径」；未明确时再读 versions.md 全量探测
2. 读邻代码        → 命名、分层、注入、判空、异常日志、注释详略、返回体
3. 定边界          → 入参 Param/DTO、出参 VO、Entity 转换走 MapStruct
4. 大文件局部改    → 目标类 500+ 行时读 incremental-change.md，只动相关方法
5. 写实现          → architecture.md + data-access.md（按画像选 API）
6. 方法结构        → method-structure.md（步骤分块、按块分层、按需抽私有方法）
7. 补注释          → comments.md（五层注释、去 AI/机翻味自检）
8. 改 YAML（如有） → config-yml.md（段分割符、配置项中文注释）
9. 交付自检        → pitfalls.md 清单
```

### versions 捷径

`CLAUDE.md`、项目 skill（如 `<project>-quickref`）或 `pom`/`gradle` **已写明** JDK、Spring Boot 代际、持久层、分页方式时：

- **可直接采用**，不必每次重跑全量探测
- 仅核对邻文件 `import`（`javax`/`jakarta`、文档注解体系）与任务相关模块

**仍须读 versions.md 全量探测**当：首次进入陌生仓库、依赖/父 POM 刚变更、或画像与邻代码 import 矛盾时。

画像未明确前，不生成高版本语法、`jakarta` 包名或项目未使用的持久层 API。

## 新增文件硬性门禁

**每新增一个 Java 类**，类注释不可缺项。项目有模板时以 `CLAUDE.md` 为准；标准三要素：

```java
/**
 * @description: 订单服务实现
 * @author: 张三
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

## 测试与验证门禁

测试框架与基类跟项目 `src/test` 邻类，不在本 skill 里强行统一；但**何时必须补验证**应一致。详细清单见 [tests-policy.md](tests-policy.md)。

| 场景 | 最低要求 |
|------|----------|
| 纯注释/文案/无行为变化重命名 | 可不新增测试；至少做编译或局部人工验证 |
| 明确单点 bug 修复 | 至少补 1 个能复现旧问题、能证明已修复的验证（单测 / 接口 / 手工步骤三选一，优先自动化） |
| 改业务行为、状态流转、异常码、分页口径、金额计算 | **必须**补或更新自动化测试；若仓库无测试基础，交付说明里列出手工验证步骤与风险 |
| 支付回调、Webhook、MQ 消费、补偿任务、幂等逻辑 | **必须**覆盖重复调用 / 非法状态 / 边界金额 或 等价场景 |

**原则**：

- 测试验证的是**行为**，不是实现细节；避免把当前代码写法当断言对象
- 存量仓无测试时，不因“补测试”擅自大规模改造测试基建；最小化补关键验证
- 只改局部逻辑时，测试范围也局部；不顺手补无关模块覆盖率

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

**新写片段**禁止手写 `== null` / `!= null` 判对象；禁止对可能为 null 的集合/Map 直接 `.isEmpty()` / `.size() == 0`。细则与场景对照表见 [coding-patterns.md#判空与工具类](coding-patterns.md#判空与工具类)。

- 对象：`Objects.isNull()` / `Objects.nonNull()`；集合/Map：`CollectionUtils`；字符串：`StringUtils.isBlank()`
- 多库并存（Hutool + Commons + JDK）决策树见 [coding-patterns.md#多工具库并存](coding-patterns.md#多工具库并存)
- `Optional` 仅用于「可能为空的一次取值 + 抛异常/默认值」；**禁止**层层嵌套防御
- `StringUtils` / `CollectionUtils` 的 **import 包名跟邻文件**
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
- `@Slf4j`；**跟邻文件**：常见为 `log.error("说明, key={}", val, e)` 末参传 `Throwable` 保留堆栈
- 禁止 `System.out` / `printStackTrace`；禁止只打 `e.getMessage()` 而无堆栈
- 邻模块无先例时，不强行引入项目未用的异常工具类（如 `ExceptionUtils`）
- `try-catch` **默认跟同模块邻代码**；支付回调、导入监听等常见 `catch (Exception e)`；无邻文件时再统一 `Exception`

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
- 回调 / MQ / 定时补偿等含幂等场景，先读 [data-access.md#外部回调与幂等](data-access.md#外部回调与幂等)
- 涉及金额字段、换算、比较或舍入时，先读 [data-access.md#金额与时间](data-access.md#金额与时间)

详见 [data-access.md](data-access.md)。

## 安全与鉴权（通用门禁）

本 skill 不绑定某个安全框架；Shiro、Spring Security、自定义 SSO、网关鉴权都适用。详细清单见 [security.md](security.md)。

- 权限校验写在**项目约定的入口层**（Controller、注解、切面或统一鉴权组件），不要在 Mapper / Convert 里散落权限分支
- 涉及“当前登录人只能看/改自己的数据”的规则，Service 层仍要保留数据范围复核，不能只相信前端入参
- 日志、异常、导出、回调报文里禁止泄露密钥、token、证件号、卡号、完整手机号等敏感信息
- 拼 SQL、拼排序字段、拼导出列时，用户输入必须白名单或参数化，不把原始输入直接进入语句片段
- 新增外部接口 / 回调处理时，要同时检查鉴权、签名、重放防护、幂等

## 中文注释（摘要）

注释是**硬性约束**，**权威细则见 [comments.md](comments.md)**（分级表、三层分工、30 秒自检）；占位词/套话表见 [comments-examples.md](comments-examples.md)。

**必须写**：类三要素、分级表中的「必须」项（公开业务方法、配置/Document/对外 VO 字段等）。

**不要写**：复述代码、与业务无关的技术动作说明、AI 套话/占位词（下面/补齐/构建索引）、机械全覆盖 Javadoc、用英文字段名当字段注释。

**步骤分块**：触发条件见 [comments.md#步骤分块](comments.md#步骤分块)；完整范本见 [examples.md](examples.md)。

<a id="git-提交说明"></a>

## Git 提交说明（通用骨架）

`type(scope): 一句话` + 正文 `-` 列表（多项改动时分条）。`type` / `scope` **跟仓库 `git log`**。

| 字段 | 说明 |
|------|------|
| `type` | 常见 `feat` `fix` `refactor` `perf` `docs` `chore` |
| `scope` | 模块或业务域，与仓库一致 |
| 正文 | 每条：做了什么 → 带来什么效果；可保留表名/路径 |

- 不提交密钥、`.env`、无关二进制；未要求不主动 commit
- 项目专属 `scope`、署名、提交用语 → 各仓 `CLAUDE.md` 或项目 skill

<a id="交付前自检"></a>

## 交付前自检（入口）

交付前跑一遍；完整分节清单见 [pitfalls.md#完整交付清单](pitfalls.md#完整交付清单)：

```
[ ] 环境画像已明确（项目 skill / versions 捷径）
[ ] 分层与对象边界正确（无 Entity 穿透、无 N+1）
[ ] 注释与类三要素符合 comments.md；占位词对照 comments-examples.md
[ ] 新写片段判空跟邻代码或 coding-patterns 决策树
[ ] 业务行为变化已补对应验证（见 tests-policy.md）
[ ] 鉴权、数据范围、日志脱敏未被改坏（见 security.md）
[ ] 未越权改模块/依赖/敏感配置
```

## 规则等级

| 等级 | 内容 | 说明 |
|------|------|------|
| **硬门禁** | Entity 不穿透 API、无 N+1、事务边界、幂等、敏感配置/依赖不乱动 | 不满足即视为缺陷 |
| **强约束** | 注释分级、步骤分块、判空与日志风格、对象边界 | 默认遵守；若项目已有稳定风格，先跟邻代码 |
| **风格建议** | 注解从短到长、长行换行、示例命名偏好 | 无明确收益时不对旧代码大面积格式化 |

## 参考文件索引

| 文件 | 何时读 | 内容 |
|------|--------|------|
| [versions.md](versions.md) | 画像不明或跨仓首次进入 | 环境探测、Java 8–21 语法、SB2/SB3、持久层/注入/分页分支 |
| [migration.md](migration.md) | 升级 Java / Spring Boot / Spring Cloud | `javax`→`jakarta`、文档/注入/Cloud 升级要点 |
| [architecture.md](architecture.md) | 定分层、URL、DTO/VO | 分包、Controller/Service、审计字段 |
| [data-access.md](data-access.md) | 写 SQL/XML、分页、回调幂等 | SQL/XML、MP 动态条件、事务、金额时间 |
| [incremental-change.md](incremental-change.md) | 改 500+ 行存量大类 | 局部修改、不顺手重构、Cloud 注意点 |
| [comments.md](comments.md) | **每次**补/审注释 | 分级表、三层分工、30 秒自检 |
| [comments-examples.md](comments-examples.md) | 去 AI 味、审查占位词 | 占位词表、套话表、正反例 |
| [method-structure.md](method-structure.md) | 长方法编排、抽私有方法 | 步骤分块模板、按块分层 |
| [coding-patterns.md](coding-patterns.md) | 判空、Stream、MapStruct | 判空表、Stream、注解排列、Service/AOP |
| [tests-policy.md](tests-policy.md) | 改业务行为、补 bug、回调/MQ/金额逻辑 | 测试触发条件、验证层级、自动化优先级 |
| [security.md](security.md) | 鉴权、数据范围、脱敏、SQL 排序白名单 | 通用安全门禁与交付核对项 |
| [quick-check.md](quick-check.md) | 只想 30 秒按改动类型自检 | 按改动类型的最小检查表 |
| [config-yml.md](config-yml.md) | 改 `application*.yml` | 段分割符、配置项中文注释、YAML 去机翻味 |
| [pitfalls.md](pitfalls.md) | 交付前 | 陷阱表、AI 气味、完整交付清单 |
| [examples-controller.md](examples-controller.md) | 需要中性化 Web 示例 | 响应包装、Controller、Swagger/OpenAPI |
| [examples-data-access.md](examples-data-access.md) | 需要持久层中性示例 | MP / MyBatis / JPA / 分页 / 动态条件 |
| [examples.md](examples.md) | **按需**（范本、润色对照） | 版本探测、步骤分块范本 A/B、注释润色、MapStruct |

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
