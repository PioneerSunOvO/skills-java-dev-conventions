# 分层与对象边界

环境版本先按 [versions.md](versions.md) 完成探测（含 Spring Cloud 与否）。

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
├── convert/
└── enums/
```

微服务常见额外结构：

```
*-api/          ← Feign 接口、共享 DTO（按项目）
*-gateway/      ← 网关路由与过滤器
scheduled/      ← 定时任务（或各业务模块内 job 包）
```

包名、子域划分以仓库为准；新增类放进与邻类相同的包。

## 新增类注释

见 [comments.md#类注释硬性](comments.md#类注释硬性)。本文件不重复模板。

## Controller

```java
// 好：薄控制器
@PostMapping("/add")
public ApiResult<Boolean> add(@Validated(Add.class) @RequestBody OrderDto dto) {
    return ApiResult.result(orderService.save(dto));
}
```

要点：

- 入参 Param / DTO；出参 VO + 项目统一响应包装（`ApiResult`、`R` 等以邻代码为准）
- **禁止 Entity 作 `@RequestBody` 或直接返回 Entity**
- 分页路径、HTTP 方法跟同模块邻 Controller
- 不在 Controller `try-catch` 吞异常
- OpenAPI 注解跟项目探测：Swagger2 或 springdoc，不混用

## Service

- 接口 `XxxService`，实现 `XxxServiceImpl`
- 跨模块经 Service 调用；**不注入其他模块 Mapper**（除非项目允许）
- 业务分支、状态机、金额、幂等放 Service
- 写方法 `@Transactional(rollbackFor = Exception.class)`
- **Feign 调用**：在 Service 编排，Feign 接口不写业务 if/else

## 对象类型选用

| 类型 | 用途 | 注意 |
|------|------|------|
| Entity | ORM 表映射 | 不进出 API |
| DTO | 创建、更新、非分页提交 | 校验防 Mass Assignment |
| Param | 分页、排序、筛选 | 继承项目分页基类 |
| VO | 列表、详情、导出 | 可含展示用关联名称 |
| BO | 领域内中间对象 | 不穿透 Controller（除非项目约定） |

多表展示：**XML 一次查 VO**；边界转换用 MapStruct。见 [coding-patterns.md](coding-patterns.md)。

## 命名与 URL

| 后缀 | 含义 |
|------|------|
| `Controller` / `Service` / `ServiceImpl` / `Mapper` | 常规分层 |
| `Dto` / `Vo` / `PageParam` / `Convert` | 入参、出参、分页、转换 |

常见路径（以邻 Controller 为准）：`POST /add`、`POST /update`、`POST /delete/{id}`、`GET /info/{id}`、`POST /getPageList`。

## 审计字段

- 有 **MetaObjectHandler** → 不手写重复赋值
- 无自动填充 → Service 新增/更新时按 Entity 字段赋值（跟邻 Service）

## 定时任务

- 类放 `scheduled` 模块或项目约定的 `job`/`task` 包
- **业务逻辑委托 Service**，Job 类只做调度与异常日志
- 写操作加事务；长任务注释说明幂等与断点续跑
- 分布式锁、分片按项目已有组件（XXL-JOB、ShedLock 等），不擅自引入新调度框架

## 导出与文件

- Excel/CSV 导出逻辑放 Service；Controller 只负责触发与写响应头
- 动态列、口径复杂时在 Service 或导出方法注释说明列含义与数据来源
- 大导出考虑分页流式写，避免一次性加载全表到内存

## 安全边界

- 鉴权注解、角色判断跟项目安全框架（Shiro、Spring Security 等）
- 敏感接口不在日志打印完整入参；密钥不落日志
- 回调接口（支付、第三方通知）在 Service 注释写清幂等键与状态机
- 数据权限过滤放 Service 或 Mapper 层，不单靠前端传参

## 测试

| 类型 | 做法 |
|------|------|
| 单元测试 | Mock Mapper/外部 Service；不启完整容器（除非项目惯例 `@SpringBootTest`） |
| 接口测试 | `MockMvc` / `WebTestClient` 跟 `src/test` 邻类基类 |
| 集成测试 | 使用测试库或 Testcontainers；不连生产库 |
| Feign / Cloud | 用 WireMock 或项目既有契约测试方式 mock 下游 |

测试类命名、框架版本以仓库为准；不写死真实密钥。

## Code Review 速查（人审）

与 [pitfalls.md](pitfalls.md) 交付清单一致，人审时重点看：

- 分层是否破了（Controller 查库、Entity 穿透）
- 复杂查询是否 N+1
- 注释是否自然、有业务口径
- 事务与幂等是否写清
- 是否擅自升级依赖或改无关模块
