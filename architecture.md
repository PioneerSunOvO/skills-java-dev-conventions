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

## 新增类注释（硬性门禁）

每新增 Java 类必须含创建人、创建时间、类职责。详见 [comments.md](comments.md)、[SKILL.md](SKILL.md)「新增文件硬性门禁」。

```java
/**
 * @description: 订单服务实现
 * @author: 开发者
 * @since: 2026-05-28 16:00
 */
```

`@description` 为空或缺 `@author` / `@since` → 不合格。

## Controller

```java
// 好：薄控制器
@PostMapping("/add")
public ApiResult<Boolean> add(@Validated(Add.class) @RequestBody OrderDto dto) {
    return ApiResult.result(orderService.save(dto));
}

// 差：Controller 里查库、算业务、拼 VO
@PostMapping("/add")
public ApiResult<Boolean> add(@RequestBody Order entity) {
    Order db = orderMapper.selectById(entity.getId());
    if (db.getAmount() > 10000) { ... }
    orderMapper.insert(entity);
    return ApiResult.ok(true);
}
```

要点：

- **门禁**：入参用 Param / DTO，**禁止 Entity 作 `@RequestBody`**；出参用 VO，**禁止直接返回 Entity**
- 出参用项目统一包装（如 `ApiResult<Vo>`、`R<Vo>`）+ VO
- 分页：`getPageList(@Validated @RequestBody XxxPageParam param)`（路径以邻 Controller 为准）
- 不在 Controller 里 `try-catch` 吞异常（交给全局异常处理）
- OpenAPI 注解跟项目探测结果：有 `springfox`/`knife4j` 用 Swagger2（`@ApiOperation`）；有 `springdoc-openapi` 用 `@Operation`；**不混用**

## Service

- 接口放 `XxxService`，实现放 `XxxServiceImpl`
- 跨模块通过 Service 调用，**不注入其他模块的 Mapper / Repository**（除非项目明确允许）
- 业务分支、状态机、金额计算、幂等判断放 Service
- 需要回滚的写方法加 `@Transactional(rollbackFor = Exception.class)`

## 对象类型选用

| 类型 | 用途 | 注意 |
|------|------|------|
| Entity | ORM 与表一一对应 | 不进出 API |
| DTO | 创建、更新、非分页提交 | 字段用校验注解防 Mass Assignment |
| Param | 分页、排序、筛选 | 继承项目分页基类 |
| VO | 列表、详情、导出列 | 可含关联名称等展示字段 |
| BO | 领域内中间对象 | 不穿透到 Controller（除非项目约定） |

转换：

- 简单边界：`XxxConvert.INSTANCE.dtoToEntity(dto)`（MapStruct）
- 列表/树形映射：`@Named` + `@BeanMapping(qualifiedByName = ...)` 区分转换目标
- 枚举展示名：`@Mapping(expression = "java(...)")` + `@Mapper(imports = {...})`
- 多表展示：**XML 查 VO**，不在 Service 里 `for` 循环补名称

MapStruct 进阶示例见 [coding-patterns.md](coding-patterns.md)。

## 命名（与项目对齐）

| 后缀 | 含义 |
|------|------|
| `Controller` | REST 入口 |
| `Service` / `ServiceImpl` | 业务 |
| `Mapper` | 数据访问 |
| `Dto` | 入参对象 |
| `Vo` | 出参对象 |
| `PageParam` | 分页入参 |
| `Convert` | 对象转换 |

类名用业务单数名词；方法名用动词或动宾短语，与现有模块保持一致。

## 接口路径（常见约定）

| 操作 | 路径示例 |
|------|----------|
| 新增 | `POST /add` |
| 更新 | `POST /update` |
| 删除 | `POST /delete/{id}` |
| 详情 | `GET /info/{id}` |
| 分页 | `POST /getPageList` |
| 列表 | `GET /list` 或 `POST /list` |

以项目已有 Controller 为准，不混用两套风格。

## 审计字段

若 `BaseEntity` 不含 `opUserid` / `opDate` 等：

- 项目有 **MetaObjectHandler** 自动填充 → 不手写重复赋值
- 无自动填充 → 在 Service 里按 Entity 字段手动赋值：

```java
entity.setOpUserid(LoginUtil.getUserId()).setOpDate(new Date());
```

## 测试（边界）

- 单元测试：Mock Service 依赖，不启完整容器（除非项目惯例即 `@SpringBootTest`）
- 接口测试：`MockMvc` / `WebTestClient` 跟项目现有测试基类
- 测试数据不依赖生产库；不写死真实密钥

不在此 skill 规定测试框架版本；有测试任务时读 `src/test` 邻类为准。
