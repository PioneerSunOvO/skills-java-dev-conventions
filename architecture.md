# 分层与对象边界

环境版本（Java、Spring Boot 代际、校验包名）先按 [versions.md](versions.md) 完成探测。

## 标准分包（按项目裁剪）

```
.../模块名/
├── controller/
├── service/
│   └── impl/
├── mapper/          ← XML 通常在 resources/mapper/
├── entity/
├── vo/              ← 出参
├── dto/             ← 非分页入参
├── param/           ← 分页入参
├── convert/         ← MapStruct 等
└── enums/
```

包名、子域划分以仓库为准；新增类放进与邻类相同的包，不另起一套结构。

<a id="模块落点与依赖方向"></a>

## 模块落点与依赖方向（强约束）

写代码前先定「放哪里」，再写类。顺序固定；局部小改只勾命中项。

| 步骤 | 自检 |
|------|------|
| **1. 定边界** | 领域业务，还是跨域技术横切（鉴权/缓存/锁/租户）？横切见 [framework-patterns.md](framework-patterns.md) |
| **2. 定落点** | 落在哪个**现有**模块/包？跟邻；**默认不新建** Maven 模块或平行分包体系。例外：仓库已具备「底座 + 业务域 + 启动壳」，且 [#多模块工程骨架](#多模块工程骨架) 决策树命中「多域/纯技术且无对应空壳」时，才允许新建 `*-spring-boot-starter-*`（仍须用户明确要求或 quickref 允许改底座） |
| **3. 定契约** | 跨业务边界 → 走 [跨模块契约](#跨模块契约)；禁止直接碰他域 Mapper/Entity |
| **4. 定方向** | 依赖只向下或指向契约；启动/聚合模块（`*-server` / `*-bootstrap` 等启动壳）**只组装依赖与 HTTP 入口，不塞领域规则** |
| **5. 定下沉** | 同构横切何时抽共享 → [framework-patterns.md#下沉门槛](framework-patterns.md#下沉门槛)（≥2 处）；禁止一处就造全局框架 |
| **6. 定端点** | 多端（admin/app/…）可分 Controller；**同一领域 Service/规则尽量一套**，禁止按端复制整套业务实现 |

```
依赖方向（示意，以仓库为准）：
  启动壳（*-server / *-bootstrap …）  →  业务模块  →  技术底座（*-common + *-spring-boot-starter-*）
  业务 A  ──契约──►  业务 B（禁碰实现面；形态 B 禁依赖对方实现模块）
```

<a id="多模块工程骨架"></a>

## 多模块工程骨架（强约束）

适用于**已拆**「技术底座 + 业务域 + 启动壳」的多模块工程。与 [跨模块契约](#跨模块契约) 形态 A/B **正交**：不强制拆 `*-api`/`*-biz`；仓内无底座三层时**省略本节**，只守 [模块落点](#模块落点与依赖方向)。具体 artifact、启用地图 → **项目 quickref**。

**导语：** 技术能力进底座 `*-spring-boot-starter-*`、按需装配；领域规则与表访问留在业务域；跨域只走契约；底座与业务之间若必须耦合，用 SPI，禁止依赖倒置。

**改底座门槛：** 新建 / 填实 / 大改底座 Starter，或改安全组件（过滤器、登录态、Token）→ 须**用户明确要求**，或项目 quickref「禁改」表允许；否则只改业务域。

### 三层职责

```
启动壳（*-server / *-bootstrap …；命名跟邻，勿与底座 `*-spring-boot-starter-*` 混称）
    ↓ 只组装依赖与配置
业务域（实现模块 + 可选契约模块）
    ↓ 依赖
技术底座（*-common + *-spring-boot-starter-*）
```

| 层 | 只做 | 禁止 |
|----|------|------|
| **技术底座** | 自动装配、横切采集面、基础设施客户端 | 依赖业务域（含业务契约模块 `*-api`）；写具体业务表规则 |
| **业务域** | 领域规则、表访问、SPI 实现、HTTP/RPC | 反向被底座依赖；他域实现模块互引（形态 B） |
| **启动壳** | Application、扫描、`application*.yml`、聚合业务模块依赖 | 塞领域规则、当「第二个 biz」；无脑全开全部底座 starter |

共享技术枚举 / 采集 DTO 放 `*-common` 或**本底座 starter**；**禁止**底座为用业务枚举而依赖 `*-api`（例：短信渠道枚举留在 sms starter，订单状态枚举留在订单契约）。

### 底座模块分类（角色名，跟邻命名）

| 角色 | 典型命名示意 | 放什么 | 不放什么 |
|------|--------------|--------|----------|
| 公共内核 | `*-common` | 统一返回、异常、分页基类、纯工具 | 业务 Entity、厂商调用编排 |
| 基础设施 Starter | `*-spring-boot-starter-web` / `security` / `redis` / `mybatis` … | AutoConfiguration、拦截器、ORM/缓存客户端 | 具体业务表读写 |
| 业务横切 Starter | `*-spring-boot-starter-biz-log` / `data-permission` / `sms` … | 注解、切面、DTO、客户端、上下文 | 落库实现、角色范围计算 |
| 空壳 Starter | 同上命名、仅占位 | pom + `package-info`，对齐后续能力 | 半实现业务污染 |

`biz-` 前缀表示偏业务横切，**仍属底座**；真正规则与表仍在业务域。多厂商 SDK 类内拆法另见 [多供应商适配](#多供应商适配)（与「能力落哪个 starter」互补，不互斥）。

### 横切两种形态

| 形态 | 何时用 | 底座负责 | 业务域负责 |
|------|--------|----------|------------|
| **纯技术** | 不碰业务表也能跑通（Web、ORM、缓存） | 全套实现 + AutoConfiguration | 只引依赖、写配置 |
| **SPI 解耦** | 必须写业务表或算业务规则（操作日志、数据权限） | 注解 / 切面 / 采集 DTO / SPI 接口 | 实现 SPI（落库、算范围） |

何时抽共享 → [framework-patterns.md#下沉门槛](framework-patterns.md#下沉门槛)（≥2 处同构横切）。声明式切面拆法 → [framework-patterns.md#声明式横切](framework-patterns.md#声明式横切-注解--aspect--策略)。

### 新增组件决策树

```
要加能力（先确认：改底座是否已获用户授权 / quickref 允许）
 ├─ 仅某一业务域使用？ → 落该域实现模块（必要时扩契约模块）
 ├─ 多域会用 / 纯技术？
 │    ├─ 已有对应 *-spring-boot-starter-*（含空壳）？ → 填实现，禁止再开同名模块
 │    └─ 无 → 新建 *-spring-boot-starter-*（父工程 modules + BOM/dependencyManagement）
 └─ 必须碰业务表/规则？ → 底座只留采集面 + SPI；实现放业务域
```

新建 / 填实底座 Starter 时勾选：

- [ ] 已获用户明确要求，或 quickref 允许改该底座模块
- [ ] 父工程已加 `<module>`（或等价）
- [ ] 根 BOM / `dependencyManagement` 已登记
- [ ] 有 `AutoConfiguration` + Boot 3 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`（Boot 2 跟邻用 `spring.factories`）
- [ ] 底座未依赖业务域 / 业务 `*-api`；技术枚举与采集 DTO 在 common 或本 starter
- [ ] 可选依赖场景用 `@ConditionalOnClass(name = "…")` 等，避免配置类硬引用缺失类导致加载失败
- [ ] 业务实现模块**按需**引入；禁止在启动壳无脑全开全部底座 starter

### 按需组合（新开项目 / 新业务模块）

| 场景 | 建议组合（角色级） |
|------|-------------------|
| 公开查询 / 同步 API | web + mybatis（可不引 security） |
| 后台登录管理 | + security + redis（或项目等价会话存储） |
| 操作审计 / 数据范围 | + 对应 biz-*，并在业务域实现 SPI |
| 短信 / 导出 / 任务 / MQ | + 对应 starter（空壳须先填实）；多厂商 SDK 再按 [多供应商适配](#多供应商适配) 拆模板方法 |

**启动壳聚合业务模块；业务模块声明自己需要的底座 starter。** 本仓启用了哪些、SPI 实现类在哪 → 项目 quickref「Framework 启用地图」。

<a id="跨模块契约"></a>

## 跨模块契约（强约束）

**目标**：跨业务边界只依赖**契约**，不依赖库表对象与数据访问实现。

**不强制**拆 `*-api` / `*-biz` Maven 模块。拆分是加固手段，不是前提。未拆时用**包级契约面**；需要编译期硬隔离再升独立 api 模块或 ArchUnit。

### 什么可跨界 / 什么禁止

| 可跨界（契约面） | 禁止跨界（实现面） |
|------------------|-------------------|
| `XxxApi` / `XxxFacade`（或项目约定的对外接口） | `Mapper` / `Repository` |
| 契约入出参 DTO | Entity / DO / PO |
| 对外枚举、错误码常量 | 他域 `*ServiceImpl` 内部细节、Convert、Mapper XML |
| 项目已约定的只读 Facade | 他域 Controller（除非项目明确是 BFF 互调） |

跨界方法参数与返回值**只能**是契约面类型（或 JDK/公共库类型）。返回他域 Entity、注入他域 Mapper → 不合格。DTO 转换在**提供方**完成，调用方不拿 DO 自己转。

### 两种落地（跟邻二选一，勿混造第三套）

| 形态 | 何时用 | 怎么保证 |
|------|--------|----------|
| **A. 包级契约** | 单体、无独立契约 Maven 模块 | 每子域固定对外包（如 `…api` / `…facade`）；他域**只** import 该包；`service`/`dal`/`entity` 禁止被他域引用 |
| **B. Maven 契约模块** | 已有先例，或拆服务/多应用复用 | 他域 **Maven 只依赖对方契约模块**；**禁止**依赖对方**实现模块**（名称跟邻：`*-api`↔`*-biz` / `*-impl` / `*-service` 等）；实现面仅留在实现模块，编译期隔离 |

```
# 形态 A（同 JAR，靠包边界）
com.example.order/
├── api/              ← 他域唯一允许依赖（可跨界面）
│   ├── OrderApi.java
│   └── dto/
├── service/          ← 他域禁止 import
├── mapper/
└── entity/

# 形态 B（编译期隔离；artifact 名跟邻）
order-api/   → 契约模块：可跨界面（接口 / 契约 DTO / 对外枚举 / 错误码等）；禁 Mapper、Entity、Controller
order-biz/   → 实现模块：实现 + DAL；依赖本域契约模块；他域不得依赖本模块
```

依赖对方**实现模块** = 打穿编译隔离（即使代码里没直接碰 Mapper，也不合格）。

```java
// 契约：他域只依赖接口与契约 DTO（形态 A/B 相同）
public interface XxxOrderApi {
    Long createOrder(@Valid XxxOrderCreateDto req);
    XxxOrderRespDto getOrder(Long id);
}

// 实现：本域内 Entity ↔ DTO，他域看不到表对象
@Service
public class XxxOrderApiImpl implements XxxOrderApi {
    public XxxOrderRespDto getOrder(Long id) {
        XxxOrderDO row = orderMapper.selectById(id);
        return XxxOrderConvert.INSTANCE.toResp(row);
    }
}
```

### 没有 Maven 拆分时如何「保证」

同 classpath 下，Java **默认挡不住**错误 import，须分层保证：

| 强度 | 做法 |
|------|------|
| **必做** | 对外能力放进约定好的契约包；跨域只依赖对方对外接口；对象转换在提供方完成；交付按下表自检 |
| **推荐** | 禁止 `import …他域….entity|mapper|dal`；评审 / Agent 交付前 grep |
| **加固（可选）** | ArchUnit / Spring Modulith 断言域边界；项目已有则跟邻补测，无则不强制新造 |

仅包约定 + 审查 = 强纪律，不是编译器铁壁。频繁踩线或准备拆服务 → 升形态 B 或加 ArchUnit。

### 跨域自检

1. 调用方是否只引用了对方的**对外契约**（约定包，或独立契约 Maven 模块），而没有引用对方的 `entity` / `mapper` / `dal`？  
2. 注入的是否为对方**对外接口**（而不是 Mapper、也不是实现类细节）？  
3. 方法参数/返回值里是否出现了对方的表实体（Entity/DO）？  
4. Entity ↔ DTO 的转换是否在**提供方**做完（调用方不拿库表对象自己转）？  
5. 仓库若未拆契约模块：新接口是否放进既有的对外包（跟邻命名，如 `api` / `facade`），而不是另起一套叫法？  
6. 形态 B：`pom` / Gradle 是否**只**依赖对方契约模块，而没有依赖对方实现模块？  

## 新增类注释（硬门禁）

每新增 Java 类必须含创建人、创建时间、类职责。**模板与字段规则**见 [comments.md](comments.md)。

`@description` 为空或缺 `@author` / `@since` → 不合格。

## Controller

> 示例中 **`R<T>`** 表示项目统一 HTTP 响应包装，**类名与静态方法跟邻 Controller**；不要因为示例新建一套响应类型。

```java
// 好：薄控制器
@PostMapping("/add")
public R<Boolean> add(@Validated(CreateGroup.class) @RequestBody BizDto dto) {
    return R.result(bizService.save(dto));
}

// 差：Controller 里查库、算业务、拼 VO
@PostMapping("/add")
public R<Boolean> add(@RequestBody BizEntity entity) {
    BizEntity db = bizMapper.selectById(entity.getId());
    if (db.getAmount().compareTo(threshold) > 0) { ... }
    bizMapper.insert(entity);
    return R.ok(true);
}
```

要点：

- **门禁**：入参用 Param / DTO，**禁止 Entity 作 `@RequestBody`**；出参用 VO，**禁止直接返回 Entity**
- 出参用项目统一包装 + VO（见上 `R<T>` 说明）
- 多端（admin/app/…）：Controller 可按端分包；**不要**在 Controller 复制业务，共用下层 Service（见 [模块落点](#模块落点与依赖方向)）
- 分页：路径与方法名**以邻 Controller 为准**（示意 `POST /page`）；入参多为 `XxxPageParam`
- 不在 Controller 里 `try-catch` 吞异常（交给全局异常处理）
- OpenAPI 注解跟项目探测结果：有 `springfox`/`knife4j`（Swagger2）用 `@ApiOperation`；有 `springdoc-openapi` 用 `@Operation`；**不混用**
- 接口入出参字段文档、列表响应与 `response` 规则见 [Swagger / OpenAPI 文档](#swagger--openapi-文档强约束)

<a id="swagger--openapi-文档强约束"></a>

## Swagger / OpenAPI 文档（强约束）

先按项目画像选用文档体系（见 [versions.md](versions.md)），**禁止** Swagger2 与 springdoc 注解混用：

| 体系 | 探测线索 | Controller | 模型类 / 字段 |
|------|----------|------------|----------------|
| **Swagger 2** | `springfox` / 旧版 Knife4j、`io.swagger.annotations` | `@Api` / `@ApiOperation` | `@ApiModel` / `@ApiModelProperty` |
| **OpenAPI 3** | `springdoc-openapi`、新版 Knife4j、`io.swagger.v3.oas.annotations` | `@Tag` / `@Operation` | `@Schema`（类与字段） |

项目 quickref / `CLAUDE.md` 已写明文档栈时直接采用；否则跟邻 Controller / Param / Vo 的 `import`。

### 入参 / 出参字段文档（强约束）

HTTP 接口的**入参、出参模型**（常见命名：`*Param` / `*DTO` / `*Req` / `*Vo` / `*Resp`，以及统一响应包装内的业务 `data` 类型）**每个对外字段**必须有文档注解，保证文档 UI（Knife4j / Swagger UI 等）「参数说明」非空。

| 要求 | 说明 |
|------|------|
| **类级** | 有类文档注解（Swagger2 `@ApiModel`；OpenAPI3 `@Schema(description=…)`），跟邻代码 |
| **字段级** | **每个**会出现在请求体/查询参数/响应体中的字段都有对应字段注解；**禁止**只写 Java `/** */` 却无文档注解（Javadoc 不能替代文档注解） |
| **文案** | 中文、写清业务含义；枚举/状态写清取值口径（如 `0未启用 1启用`）；必填字段按栈补 `required` / `requiredMode`（跟邻） |
| **范围** | 仅 API 契约模型；**不要求** Entity/DO、Mapper、内部 Service DTO 写 `@Schema`/`@ApiModelProperty`（除非该类型直接暴露给 OpenAPI） |
| **嵌套** | 嵌套对象、内部静态类、集合元素类型同样按字段补注解 |
| **禁重复** | 同一字段禁止堆多个同名文档注解（`@Schema` / `@ApiModelProperty` 均不可重复） |

```java
// OpenAPI 3（springdoc）示意
@Schema(description = "登录日志")
public class XxxLoginLogVo {
    @Schema(description = "日志主键")
    private Long id;

    @Schema(description = "登录结果（1成功 2账号或密码错误 3用户禁用）")
    private Integer result;
}

// Swagger 2 示意
@ApiModel(value = "XxxLoginLogVo", description = "登录日志")
public class XxxLoginLogVo {
    @ApiModelProperty(value = "日志主键")
    private Long id;
}
```

### Swagger 2：列表响应与 `response` 类型

使用 Swagger 2（`io.swagger.annotations`）+ Springfox / Knife4j 时，**文档注解须与真实返回类型一致**。`@ApiOperation(response = …)` 会覆盖方法签名推断，写错则文档工具中响应参数缺失或结构错误。

| 场景 | 做法 |
|------|------|
| **`R<List<T>>`** | **必须** `response = T.class` **且** `responseContainer = "List"`；缺 `responseContainer` 时文档按**单个对象**生成，数组与 `data` 结构丢失 |
| **`response` 指向类型** | 写实际 API 出参 **Vo/DTO**，**禁止**写 Entity（即使 Vo `extends` Entity） |
| **Vo extends Entity** | Vo 须有独立 `@ApiModel(value="…")` + **全部对外字段**（含继承后扩展字段）`@ApiModelProperty`；否则 Springfox 常只注册父 Entity 模型，Vo 扩展字段在文档中不可见 |
| **单对象 `R<T>`** | `response = T.class` 可跟项目惯例（多数项目文档只展示 `data` 内层类型，不重复包 `R` 外层） |
| **分页 `R<PageOf<T>>`** | `PageOf` 表示项目分页包装（如 `Paging`、`PageResult` 等）；`response` 跟邻 Controller |

```java
// 差：返回 List，文档却声明为单个 Vo → 仅父 Entity 字段、无数组
@ApiOperation(value = "列表", response = XxxVo.class)
public R<List<XxxVo>> list() { return R.ok(service.list()); }

// 好：List 容器 + Vo 类型
@ApiOperation(value = "列表", response = XxxVo.class, responseContainer = "List")
public R<List<XxxVo>> list() { return R.ok(service.list()); }

// 差：实际返回 Vo，response 却指向 Entity
@ApiOperation(value = "列表", response = XxxEntity.class, responseContainer = "List")
public R<List<XxxVo>> list() { ... }

// 好：Vo 独立 @ApiModel，扩展字段才会出现在文档
@ApiModel(value = "XxxVo")
public class XxxVo extends XxxEntity {
    @ApiModelProperty("扩展说明字段")
    private String extDesc;
}
```

交付前可在 OpenAPI JSON（常见 `/v2/api-docs`、`/v3/api-docs` 或项目配置路径）核对：列表 200 响应应为 `type: array` + `items.$ref` 指向 Vo；模型字段均有 `description`。

## Service

- 接口放 `XxxService`，实现放 `XxxServiceImpl`
- 跨业务调用只走对方的**对外接口**（常见命名：`XxxApi` / `XxxFacade`；若仓库已拆出独立契约模块，则只依赖该模块），经接口传 DTO。  
  **禁止**直接注入或使用对方的 Mapper、Repository、表实体（Entity/DO）——库表细节留在对方内部，由对方实现类完成转换后再返回。
- 业务分支、状态机、金额计算、幂等判断放 Service
- 需要回滚的写方法加 `@Transactional(rollbackFor = Exception.class)`
- 多端入口：可按 admin/app 等分包 Controller；**领域规则留在共享 Service**，禁止按端复制整套 Impl

<a id="业务错误码"></a>

## 业务错误码（强约束）

| 要点 | 做法 |
|------|------|
| **分段** | 按系统/模块划号段（十进制或文档表均可）；新增码落在本模块区间，避免冲突 |
| **对象化** | 常量集中声明（`ErrorCode` / 枚举 / 接口常量跟邻）；禁止业务里散落魔法数字 + 裸字符串 |
| **消息** | 用户可读；可变部分用占位符（`{}` / `{0}` 跟邻格式化工具），勿手拼难排查的文案 |
| **命名** | `模块_场景_结果`，可读出失败原因（如 `ORDER_PAY_AMOUNT_NOT_MATCH`） |
| **区间登记** | 仓库有错误码区间说明类/文档 → 新模块先登记再占用；无则在本模块常量文件头注释写清号段 |
| **场景一一对应（禁止混用）** | 抛出的错误码必须与**当前分支的真实失败原因**一致；禁止「图省事」复用邻近码。日志/排障靠码与文案定位，混用会直接误导排查 |

### 错误码与失败场景一一对应（细则）

抛业务异常（类名跟邻，如 `BizException`）时：

1. **先写清本分支失败原因**（入参缺失？资源不存在？状态不允许？下游失败？）  
2. **再选常量**：常量名、Javadoc 适用范围、默认文案三者须同时匹配该原因  
3. **缺码则新增**：同一号段内补常量；禁止用「差不多」的码顶替（例：必填入参为空 ≠ 配置/资源未启用；凭证校验失败 ≠ 账号未绑定；登录态下「绑定对象缺失」≠ 管理端 CRUD 的「记录不存在」）  
4. **分场景校验分抛**：多个独立入参的空值/非法校验，不要合成一个 `if` 后抛同一码  
5. **跨子域慎用**：子域 A 的失败（如 `AUTH_*`）勿直接抛子域 B（如 `USER_*`）的码/文案，若会把排障引向错误模块；应在本子域号段声明语义准确的码

```java
// 错误：入参空却抛「资源未启用」——日志会让人去查配置/上下架
if (!StringUtils.hasText(skuId) || !StringUtils.hasText(channelCode)) {
    throw BizException.of(ErrorCodes.PRODUCT_OFF_SHELF);
}

// 正确：一原因一码（常量名跟项目 ErrorCode 约定）
if (!StringUtils.hasText(skuId)) {
    throw BizException.of(ErrorCodes.PRODUCT_SKU_ID_EMPTY);
}
if (!StringUtils.hasText(channelCode)) {
    throw BizException.of(ErrorCodes.PRODUCT_CHANNEL_CODE_EMPTY);
}
```

```java
// 占位：类名、构造跟项目已有 ErrorCode
/** 订单模块错误码；号段 1-011-xxx 以仓库区间文档为准 */
ErrorCode ORDER_PAY_AMOUNT_NOT_MATCH = new ErrorCode(1_011_000_016, "支付金额与订单不一致");
ErrorCode ORDER_STATUS_NOT_ALLOW = new ErrorCode(1_011_000_033, "订单状态不允许此操作，当前需为【{}】");
```

全局 HTTP/系统码（未登录、无权限等）与业务码分区；业务码不要占用全局段。

<a id="统一响应与异常"></a>

## 统一响应与异常贯通（强约束）

> 响应类名、成功码、静态工厂**一律跟邻**（示意 `R` / `CommonResult`）；禁止为规范新建第二套包装。

| 要点 | 做法 |
|------|------|
| **HTTP 出口** | Controller 只返回统一包装；业务失败抛项目业务异常，由全局异常处理转包装 |
| **跨模块调用** | 若契约返回统一包装：失败时 `checkError()` / `getCheckedData()`（或邻等价方法）转本地业务异常，禁止吞掉 `code` |
| **成功断言** | `error(...)` 工厂禁止传入成功码（跟邻 Assert） |
| **禁止** | Controller `try-catch` 后改写成成功；跨模块拿到失败包装当成功数据用 |

## 对象类型选用

| 类型 | 用途 | 注意 |
|------|------|------|
| Entity | ORM 与表一一对应 | 不进出 API；样板用 Lombok（见 [coding-patterns.md#lombok](coding-patterns.md#lombok)） |
| DTO | 创建、更新、非分页提交 | 字段用校验注解防 Mass Assignment；禁手写 get/set |
| Param | 分页、排序、筛选 | 继承项目分页基类；禁手写 get/set |
| VO | 列表、详情、导出列 | 可含关联名称等展示字段；禁手写 get/set |
| BO | 领域内中间对象 | 不穿透到 Controller（除非项目约定） |

转换：

- 简单边界：用 MapStruct `*Convert`（`INSTANCE` **或**注入式 `@Mapper(componentModel = "spring")`，**跟邻**）；**禁止**新写 `BeanUtils`/`BeanUtil.copyProperties`（反射拷贝无法 IDE 查找字段引用）
- 多表展示：**优先 XML 一次返 VO**；若必须多源组装，放 Convert 的 `default` 方法做**纯映射/分组挂载**（如 `convertMultiMap` 后填 VO），**禁止**在 Convert 里写状态机、金额规则、查库
- Controller 可编排「查主表 → 批量查关联 → Convert 组装」；业务规则仍在 Service
- Convert **禁止**膨胀成上帝接口：按读写/场景拆分（如 `XxxPageConvert` / `XxxDetailConvert`），跟邻模块粒度
- 列表映射、枚举展示名、`@Named` 等进阶写法 → **权威** [coding-patterns.md#MapStruct](coding-patterns.md#MapStruct)

<a id="扩展点与有序链"></a>

## 扩展点与有序链（风格建议）

主流程会随营销/渠道/插件膨胀时，优先开闭扩展，避免巨型 `if-else`。

| 模式 | 适用 | 要点 |
|------|------|------|
| **生命周期钩子** | 创建前/后、支付后、取消后等 | 接口 `default` 空实现；Spring 注入 `List<XxxHandler>`；主流程只 `forEach` 调用 |
| **有序计算器** | 计价、优惠叠加、运费等有明确先后 | 优先级用**命名常量**或 `@Order`；注释写清为何此序（可链外部业务文档） |
| **跟邻** | 项目已有 Handler/Calculator → 新扩展实现同接口，不另起一套 |

**何时不抽**：仅 1～2 处简单分支、无第二扩展预期 → 留在 Service 步骤块内即可。编排细节 → [method-structure.md#扩展点抽取](method-structure.md#扩展点抽取)。

占位代码与 Context/声明式横切/CAS 封装 → [framework-patterns.md](framework-patterns.md)。

<a id="上下文与声明式横切"></a>

## 上下文与声明式横切（风格建议）

是否采用这些形态属风格建议；**一旦项目已有 Context**，业务禁裸改 ThreadLocal 为强约束（见 framework-patterns 对应节）。

| 模式 | 一句话 | 详情 |
|------|--------|------|
| **Context + Utils.execute** | 备份/恢复 ThreadLocal（优先 TTL）；业务禁裸改 | [framework-patterns.md#上下文作用域](framework-patterns.md#上下文作用域-ttltl--utils) |
| **注解 + Aspect + 策略** | 声明式能力一件事一个注解；策略可插拔 | [framework-patterns.md#声明式横切](framework-patterns.md#声明式横切-注解--aspect--策略) |
| **幂等 ≠ 锁** | 成功后立刻删 Key ≠ 分布式锁 | [data-access.md#幂等与分布式锁](data-access.md#幂等与分布式锁) |

项目已有租户/数据权限/幂等封装时**跟邻调用**，不要平行再造。多模块仓数据权限 SPI 落点 → [多模块工程骨架](#多模块工程骨架)。

<a id="多供应商适配"></a>

## 多供应商适配（风格建议）

支付 / 短信 / 物流 / OCR 等 SDK 差异大时：

| 层 | 职责 |
|----|------|
| **对外接口** | 统一入出参（下单、查单、解析回调） |
| **模板外层** | 校验、日志、把 SDK 异常译成项目业务/支付异常；方法宜 `final` |
| **子类** | 只实现 `doXxx` 调具体 SDK |
| **工厂** | 按渠道/配置解析客户端；缓存跟邻 |

禁止在业务 Service 里直接散落各 SDK API。细则示例 → [coding-patterns.md#多供应商模板方法](coding-patterns.md#多供应商模板方法)。

多模块仓先定能力落点（哪个 `*-spring-boot-starter-*`、是否 SPI）→ [多模块工程骨架](#多模块工程骨架)；再按本节拆模板方法 / 工厂。

## 命名（与项目对齐）

| 后缀 | 含义 |
|------|------|
| `Controller` | REST 入口 |
| `Service` / `ServiceImpl` | 业务 |
| `Mapper` | 数据访问 |
| `Dto` | 入参对象 |
| `Vo` | 出参对象 |
| `PageParam` | 分页入参 |
| `Convert` | 对象转换（MapStruct 转换类统一 `*Convert` 后缀） |

类名用业务单数名词；方法名用动词或动宾短语，与现有模块保持一致。

## 接口路径（常见约定）

| 操作 | 路径示例（示意，**以邻 Controller 为准**） |
|------|------|
| 新增 | `POST /add` |
| 更新 | `POST /update` |
| 删除 | `POST /delete/{id}` |
| 详情 | `GET /info/{id}` |
| 分页 | `POST /page` |
| 列表 | `GET /list` 或 `POST /list` |

以项目已有 Controller 为准，不混用两套风格。

## 审计字段

字段名、类型、setter **一律跟邻 Entity**（不同项目可能是 `createBy`/`updateTime`、`opUserId`/`opDate` 等）：

- 项目有 **MetaObjectHandler**（或等价自动填充）→ 不手写重复赋值
- 无自动填充 → 在 Service 里按 Entity 实际字段赋值：

```java
// 登录人工具与字段名均跟邻代码（以下为占位示意）
entity.setCreateBy(CurrentUser.getUserId()).setUpdateTime(new Date());
```

## 测试（边界）

**何时必须补验证、优先补哪一层** → [tests-policy.md](tests-policy.md)。

- 单元测试：Mock Service 依赖，不启完整容器（除非项目惯例即 `@SpringBootTest`）
- 接口测试：`MockMvc` / `WebTestClient` 跟项目现有测试基类
- 测试数据不依赖生产库；不写死真实密钥

不在此 skill 规定测试框架版本；有测试任务时读 `src/test` 邻类为准。
