# 版本与环境探测

动手写代码前**必须先完成环境画像**。项目 `CLAUDE.md` / `AGENTS.md` 中的版本锁定、模块禁改等规则优先于本文件。

**与版本无关的规则**（分层、注解从短到长、长行换行、步骤分块、注释五层、禁止 N+1 等）不读本文件也可执行；本文件只解决「在当前仓库里该用哪套 API / 包名 / 语法」。

## 环境探测流程

### 1. 读构建文件

| 构建工具 | 文件 |
|----------|------|
| Maven | 根 `pom.xml`、子模块 `pom.xml` |
| Gradle | `build.gradle` / `build.gradle.kts`、`gradle.properties` |

### 2. 提取关键信号

在构建文件中 grep（属性名因项目而异，不假设唯一）：

| 维度 | 常见属性 / 依赖 |
|------|----------------|
| Java | `java.version`、`maven.compiler.source`、`sourceCompatibility` |
| Spring Boot | `spring-boot.version`、`spring-boot-dependencies` |
| MyBatis-Plus | `mybatis-plus-boot-starter` |
| 原生 MyBatis | `mybatis-spring-boot-starter`（无 MP 时） |
| JPA | `spring-boot-starter-data-jpa` |
| 分页 | `pagehelper-spring-boot-starter` |
| API 文档 | `springfox` / `knife4j` vs `springdoc-openapi` |
| 安全 | `shiro-spring` vs `spring-boot-starter-security` |

### 3. 邻域抽样确认

在同模块各抽 1–2 个文件核对：

- `import javax.*` 还是 `jakarta.*`
- 依赖注入写法（字段 `@Autowired`、Lombok `@Setter`、构造器、`@Resource`）
- 分页封装类名（`PageInfo`、`Paging`、`Page<T>`、`Pageable`）
- 统一响应类名（`ApiResult`、`R`、`Result` 等）
- Entity 时间字段类型（`Date`、`LocalDateTime`、`Instant`）

### 4. 输出环境画像（心中或注释里默念）

```text
Java: 8 | Spring Boot: 2.2.5 | 持久层: MyBatis-Plus 3.3
校验包: javax.validation | 分页: PageInfo + Paging
注入: @Setter(onMethod=@__(@Autowired)) | 文档: Swagger2
```

画像未明确前，**不生成**高版本语法、`jakarta.*` 或项目未使用的持久层 API。

---

## Java 版本语法对照（8–21）

以 `pom`/`gradle` 中 `java.version` 为上限；**禁止**在较低版本仓库使用高版本专属语法。

| 禁用语法 | 最低 Java | 替代写法 |
|----------|-----------|----------|
| `var` | 10 | 显式声明类型 |
| `List.of()` / `Map.of()` / `Set.of()` | 9 | `Arrays.asList()` / `Collections.singletonMap()` / `new HashSet<>(Arrays.asList(...))` |
| `String.isBlank()` / `strip()` | 11 | `StringUtils.isBlank()` / `trim()` |
| 文本块 `"""` | 13 | 字符串拼接或 `+` |
| `switch` 表达式（无 break） | 14 | 传统 `switch` + `break` |
| `record` | 16 | `@Data` class + `final` 字段 |
| `sealed` 类 | 17 | 普通 class + 文档说明继承约束 |
| `Stream.toList()` | 16 | `.collect(Collectors.toList())` |
| `instanceof` 模式匹配 | 16 | `if (x instanceof Foo) { Foo foo = (Foo) x; }` |
| `Optional.isEmpty()` | 11 | `!optional.isPresent()` |

Java 21 特性（虚拟线程、`switch` 模式等）仅在项目明确使用 Java 21 时采用。

---

## JDK 特性使用边界

在 `java.version` 上限内，**鼓励**适度使用项目已广泛采用的写法：

| 可用（按版本） | 用途 |
|----------------|------|
| `Stream` + `collect` | 过滤、映射、分组、合并 |
| `Optional` | 查无则抛、默认值；避免深层 `getA().getB()` |
| Lambda / 方法引用 | 集合处理、MapStruct `INSTANCE::method` |
| MP 条件 Lambda | `.eq(Objects.nonNull(x), Entity::getField, x)` |

**禁止为炫技而写**：

- 超出项目 Java 版本的语法（`var`、`record`、`List.of()` 等，见上表）
- `Optional` 嵌套防御（`Optional.of(Optional.of(x))`）
- 把简单 `if (CollectionUtils.isEmpty(list))` 改成难读的 Stream 链
- 用高版本 API 替代项目已有的 `StringUtils` / `CollectionUtils`

判空工具分工见 [coding-patterns.md](coding-patterns.md)。

---

## Spring Boot 2.x vs 3.x

| 项 | Spring Boot 2.x | Spring Boot 3.x |
|----|-----------------|-----------------|
| 常见最低 Java | 8（部分 2.7+ 支持 17） | **17** |
| Servlet / JPA / Validation | `javax.*` | `jakarta.*` |
| 校验注解包 | `javax.validation.constraints.*` | `jakarta.validation.constraints.*` |
| API 文档（常见） | Swagger2：`@Api`、`@ApiOperation` | springdoc：`@Tag`、`@Operation` |
| 配置迁移 | — | 少数属性重命名（如 `spring.redis.*` 结构调整），以官方迁移指南为准，不全文搬运 |

**SB3 项目**：新增代码全面使用 `jakarta.*`，禁止混用 `javax.servlet` 等同文件。

**SB2 项目**：保持 `javax.*`，不要提前「迁移」到 jakarta。

---

## 持久层分支决策

```
依赖检测
├── 有 mybatis-plus-boot-starter
│   └── MyBatis-Plus：BaseMapper、IService、Wrappers、Page<T>、saveBatch
├── 仅有 mybatis-spring-boot-starter（无 MP）
│   └── 原生 MyBatis：Mapper 接口 + XML，不用 Wrappers / BaseMapper
├── 有 spring-boot-starter-data-jpa
│   └── JPA：JpaRepository、@Query、Specification、投影 DTO
└── 混用（MP + JPA 等）
    └── 读邻模块已有写法，不擅自引入第二种风格
```

### 分页三分支

| 信号 | 做法 |
|------|------|
| `pagehelper-spring-boot-starter` | `PageHelper.startPage()` + 项目分页出参封装 |
| MyBatis-Plus | `Page<Entity>` + `mapper.selectPage(page, wrapper)` |
| Spring Data JPA | `Pageable` + `Page<T>` 或项目自定义 `PageParam` |

禁止全量 `list()` 后 `subList` 内存分页。

### JPA 补充

- 简单 CRUD：`JpaRepository` 派生方法
- 复杂报表：`@Query(nativeQuery=true)` 或项目混用的 MyBatis Mapper
- **懒加载关联不在 Controller 触发**；展示数据在 Service 一次查齐或用 DTO 投影

---

## 依赖注入风格

探测邻类后**只沿用已有风格**，不引入项目没有的新模式：

| 风格 | 识别特征 | 注意 |
|------|----------|------|
| 字段 `@Autowired` | 最常见 | 部分规范禁止，以项目为准 |
| Lombok `@Setter(onMethod = @__(@Autowired))` | 部分国内项目偏好 | 禁止与字段 `@Autowired` 混用 |
| 构造器注入 | SB 官方推荐，SB3 常见 | 单构造器可省略 `@Autowired` |
| `@Resource` | 按名称注入 | 与 `@Autowired` 不混用除非项目已混用 |

---

## 时间类型与 JSON

| 项目代际 | Entity 常见类型 | JSON 配置 |
|----------|----------------|-----------|
| Java 8 + SB2 | `java.util.Date` | `spring.jackson.date-format`、`time-zone` |
| Java 11+ / SB2.7+ / SB3 | `LocalDateTime`、`Instant` | 同上或 `@JsonFormat` |

**规则**：跟邻 Entity 字段类型一致，同一模块不混用 `Date` 与 `LocalDateTime`。

金额优先整数分（`Long`/`Integer`），注释标明单位；避免 `double` 做账务计算。

---

## 校验与 API 文档包名速查

| Spring Boot | 校验 import | 文档注解（示例） |
|-------------|-------------|------------------|
| 2.x | `javax.validation.*` | `io.swagger.annotations.ApiOperation` |
| 3.x | `jakarta.validation.*` | `io.swagger.v3.oas.annotations.Operation` |

探测到 `springfox` 或 `knife4j` → Swagger2 体系；探测到 `springdoc-openapi` → OpenAPI 3 体系。**不混用**两套注解。

---

## 常见误判

| 误判 | 正确做法 |
|------|----------|
| 看到 MyBatis 就写 `Wrappers` | 确认是否有 `mybatis-plus-boot-starter` |
| SB2 项目写 `jakarta.validation` | 保持 `javax.validation` |
| Java 8 项目用 `record` 省样板 | 改 `@Data` class |
| 擅自升级 `pom`/`gradle` 版本 | 禁止，除非用户明确要求 |

更多陷阱见 [pitfalls.md](pitfalls.md)；对照示例见 [examples.md](examples.md)。
