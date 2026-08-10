# 版本与环境探测

动手写代码前**须明确环境画像**（不必每次全量探测）。项目 `CLAUDE.md` / `AGENTS.md` 中的版本锁定、模块禁改等规则优先于本文件。

**版本无关规则**（分层、注释、N+1 等）见 [SKILL.md](SKILL.md)；本文件只解决「当前仓库该用哪套 API / 包名 / 语法」。

## 捷径（可跳过全量探测）

与 [SKILL.md](SKILL.md)「versions 捷径」一致：

| 条件 | 做法 |
|------|------|
| `CLAUDE.md`、项目 skill（如 `<project>-quickref`）或 `pom`/`gradle` **已写明** JDK、Spring Boot 代际、持久层、分页 | **直接采用**；只核对任务相关模块邻文件 `import`（`javax`/`jakarta`、文档注解） |
| 首次进入陌生仓库、依赖/父 POM 刚变更、画像与邻代码 import 矛盾 | 走下文**全量探测** |

画像未明确前，**不生成**高版本语法、`jakarta.*` 或项目未使用的组件 API。

### 探测失败时的保守策略

若构建文件不全、父 POM 看不到、或同仓多模块风格不一致，按以下顺序保守落地：

1. **先看邻模块 import 与测试**，不要凭经验猜框架
2. **无法确认是否 MP** → 先按原生 MyBatis / JPA 邻代码写，不主动生成 `Wrappers`
3. **无法确认文档体系** → 沿用邻 Controller 的 Swagger2 / springdoc / 无注解风格
4. **无法确认是否 Cloud** → 不主动生成 `@FeignClient`、Gateway 过滤器、配置中心代码
5. **仍不确定且改动会引入新契约** → 先 `brainstorming`，向用户确认

## 环境探测流程

### 读构建文件

| 构建工具 | 文件 |
|----------|------|
| Maven | 根 `pom.xml`、子模块 `pom.xml` |
| Gradle | `build.gradle` / `build.gradle.kts`、`gradle.properties` |

### 提取关键信号

| 维度 | 常见属性 / 依赖 |
|------|----------------|
| Java | `java.version`、`maven.compiler.source`、`sourceCompatibility` |
| Spring Boot | `spring-boot.version`、`spring-boot-dependencies` |
| **Spring Cloud** | `spring-cloud-dependencies`、`spring-cloud.version` |
| MyBatis-Plus | `mybatis-plus-boot-starter` |
| 原生 MyBatis | `mybatis-spring-boot-starter`（无 MP） |
| JPA | `spring-boot-starter-data-jpa` |
| 分页 | `pagehelper-spring-boot-starter` |
| API 文档 | `springfox` / `knife4j` vs `springdoc-openapi` |
| 安全 | `shiro-spring` vs `spring-boot-starter-security` |
| 注册/配置中心 | `nacos-discovery`、`eureka-client`、`consul-discovery`、`spring-cloud-config` |
| 网关 / 调用 | `spring-cloud-starter-gateway`、`spring-cloud-starter-openfeign` |

### 邻域抽样确认

同模块抽 1–2 个文件核对：

- `javax.*` 还是 `jakarta.*`
- 依赖注入写法
- 分页封装类名
- 统一响应类名
- Entity 时间字段类型
- **微服务项目**：Feign 接口位置、`@FeignClient` 命名、是否独立 `*-api` 模块

### 输出环境画像（示例）

```text
Java: 8 | Spring Boot: 2.2.5 | Spring Cloud: 无
持久层: MyBatis-Plus 3.3 | 校验: javax.validation | 分页: 项目自定义封装
注入: @Setter(onMethod=@__(@Autowired)) | 文档: Swagger2
```

```text
Java: 17 | Spring Boot: 3.2 | Spring Cloud: 2023.0.x
持久层: MyBatis-Plus | 注册: Nacos | 调用: OpenFeign | 网关: Gateway
校验: jakarta.validation | 文档: springdoc
```

---

<a id="spring-cloud-分支"></a>

## Spring Cloud 分支

检测到 `spring-cloud-dependencies` 或子模块含 `*-gateway`、`*-api`、Feign 接口时，进入 Cloud 画像：

| 组件 | 识别 | 编码注意 |
|------|------|----------|
| **OpenFeign** | `@FeignClient`、`spring-cloud-starter-openfeign` | 接口放 api 模块或约定包；不在 Service 里手写 URL 拼参 |
| **Gateway** | `spring-cloud-starter-gateway` | 路由、过滤器跟邻配置；业务逻辑不放 Gateway |
| **Nacos / Eureka** | 对应 starter | 服务名与 `spring.application.name` 一致；配置从配置中心读时不硬编码环境地址 |
| **Config** | `spring-cloud-config` | 敏感项外置；本地 `bootstrap`/`application` 分工跟项目 |
| **负载 / 熔断** | `spring-cloud-loadbalancer`、`resilience4j`、`sentinel` | 用项目已有组件，不擅自引入第二种 |

**原则**：

- 单体模块与 Cloud 模块**分层相同**（Controller → Service → Mapper），Feign 相当于跨服务 Service 调用
- 跨服务 DTO 用 api 模块共享类，不传 Entity
- 分布式事务：以项目方案为准（Seata、最终一致、 Saga）；在注释中写清一致性边界

无 Spring Cloud 依赖时，**不生成** Feign、Gateway 相关代码。

---

<a id="java-版本语法"></a>

## Java 版本语法对照（8–21）

以 `java.version` 为上限；禁止在较低版本仓库使用高版本专属语法。

| 禁用语法 | 最低 Java | 替代写法 |
|----------|-----------|----------|
| `var` | 10 | 显式声明类型 |
| `List.of()` / `Map.of()` | 9 | `Arrays.asList()` / `Collections.singletonMap()` |
| `String.isBlank()` | 11 | `StringUtils.isBlank()` |
| 文本块 `"""` | 13 | 字符串拼接 |
| `switch` 表达式 | 14 | 传统 `switch` + `break` |
| `record` | 16 | `@Data` class |
| `Stream.toList()` | 16 | `.collect(Collectors.toList())` |
| `instanceof` 模式匹配 | 16 | `instanceof` + 强转 |

---

## Spring Boot 2.x vs 3.x

| 项 | SB 2.x | SB 3.x |
|----|--------|--------|
| 常见最低 Java | 8（部分 2.7+ 支持 17） | **17** |
| Servlet / JPA / Validation | `javax.*` | `jakarta.*` |
| API 文档（常见） | Swagger2 | springdoc OpenAPI 3 |

SB3 全面 `jakarta.*`；SB2 保持 `javax.*`，不提前迁移。

---

## 持久层分支决策

```
依赖检测
├── mybatis-plus-boot-starter → MP：BaseMapper、Wrappers、Page<T>
├── 仅 mybatis-spring-boot-starter → 原生 MyBatis + XML
├── spring-boot-starter-data-jpa → JPA（见 data-access.md 附录）
└── 混用 → 跟邻模块，不擅自引入第二种风格
```

### 分页三分支

<a id="分页三分支"></a>

| 信号 | 做法 |
|------|------|
| MP + 项目自定义分页封装 | 跟邻 Controller/Service 的封装类（常见 `PageResult`、包装 `IPage` 等；**类名以项目为准**） |
| PageHelper | `PageHelper.startPage()` + 出参跟邻代码（常含 `com.github.pagehelper.PageInfo`） |
| MyBatis-Plus 原生 | `Page<Entity>` + `selectPage` |
| Spring Data JPA | `Pageable` + `Page<T>` |

禁止全量 `list()` 后 `subList` 内存分页。对照示例见 [examples.md#分页写法对照](examples.md#分页写法对照)。

---

## 依赖注入风格

探测邻类后**只沿用已有风格**：

| 风格 | 识别 |
|------|------|
| 字段 `@Autowired` | 最常见 |
| Lombok `@Setter(onMethod = @__(@Autowired))` | 部分国内项目 |
| 构造器注入 | SB 官方推荐，SB3 常见 |
| `@Resource` | 按名称注入 |

---

## 时间类型与 JSON

跟邻 Entity 一致；同一模块不混用 `Date` 与 `LocalDateTime`。金额用 `BigDecimal`，单位跟邻字段注释（元/分等）。

---

## 常见误判

| 误判 | 正确做法 |
|------|----------|
| 看到 MyBatis 就写 `Wrappers` | 确认是否有 MP |
| 单体项目生成 `@FeignClient` | 确认是否有 Spring Cloud |
| SB2 写 `jakarta.validation` | 保持 `javax` |
| Java 8 用 `record` | 改 `@Data` class |
| 擅自升级依赖版本 | 禁止，除非用户明确要求 |
| 引入项目未用的 JSON 库新 API | 跟邻代码；见下文「老旧依赖」 |

---

<a id="老旧依赖与第三方组件"></a>

## 老旧依赖与第三方组件

存量项目常锁定较旧版本的 JSON、HTTP、安全、注册中心等组件。**跨项目通用原则**：

| 原则 | 说明 |
|------|------|
| **跟邻代码** | 序列化用项目已有的 Fastjson / Jackson / Gson；不混用两套注解 |
| **不擅自升级** | 不在单次业务改动中升 `pom` / `gradle` 版本，除非用户明确要求 |
| **不引入新库** | 已有 Hutool / Guava / Commons 时，不另加功能重叠依赖 |
| **安全敏感** | 已知有漏洞的旧版 JSON 库：业务改动不扩大使用面；升级走专项任务 |
| **Spring Cloud** | 注册中心、配置中心、网关版本与 BOM 对齐；不单独升某个 starter |

探测时记录画像中的关键组件（示例，非穷举）：

```text
JSON: Jackson 2.x | HTTP: RestTemplate | 注册: Nacos | 熔断: Sentinel
```

写新代码时以该画像为准；无先例则不引入第二种同类组件。

更多见 [pitfalls.md](pitfalls.md)、[examples.md](examples.md)。

---

## 升级 / 迁移入口

遇到 Java、Spring Boot、Spring Cloud、JSON 库、API 文档体系升级任务时，先读 [migration.md](migration.md)。普通业务改动里**不顺手迁移**依赖或注解体系。
