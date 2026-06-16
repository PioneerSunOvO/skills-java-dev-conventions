# 版本与环境探测

动手写代码前**必须先完成环境画像**。项目 `CLAUDE.md` / `AGENTS.md` 中的版本锁定、模块禁改等规则优先于本文件。

**版本无关规则**（分层、注释、N+1 等）见 [SKILL.md](SKILL.md)；本文件只解决「当前仓库该用哪套 API / 包名 / 语法」。

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
持久层: MyBatis-Plus 3.3 | 校验: javax.validation | 分页: PageInfo + Paging
注入: @Setter(onMethod=@__(@Autowired)) | 文档: Swagger2
```

```text
Java: 17 | Spring Boot: 3.2 | Spring Cloud: 2023.0.x
持久层: MyBatis-Plus | 注册: Nacos | 调用: OpenFeign | 网关: Gateway
校验: jakarta.validation | 文档: springdoc
```

画像未明确前，**不生成**高版本语法、`jakarta.*` 或项目未使用的组件 API。

---

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
| MP + 项目 `PageInfo` / `Paging` | `new PageInfo<>(param)` + Mapper 分页方法 + `new Paging<>(iPage)`；部分脚手架/多模块项目常见 |
| PageHelper | `PageHelper.startPage()` + 项目分页出参 |
| MyBatis-Plus 原生 | `Page<Entity>` + `selectPage` |
| Spring Data JPA | `Pageable` + `Page<T>` |

禁止全量 `list()` 后 `subList` 内存分页。对照示例见 [examples.md#分页](examples.md#分页)。

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

跟邻 Entity 一致；同一模块不混用 `Date` 与 `LocalDateTime`。金额优先整数分，注释标明单位。

---

## 常见误判

| 误判 | 正确做法 |
|------|----------|
| 看到 MyBatis 就写 `Wrappers` | 确认是否有 MP |
| 单体项目生成 `@FeignClient` | 确认是否有 Spring Cloud |
| SB2 写 `jakarta.validation` | 保持 `javax` |
| Java 8 用 `record` | 改 `@Data` class |
| 擅自升级依赖版本 | 禁止，除非用户明确要求 |

更多见 [pitfalls.md](pitfalls.md)、[examples.md](examples.md)。
