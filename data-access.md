# 数据访问与 SQL

持久层与分页分支见 [versions.md](versions.md)。JPA 项目将 Mapper/XML 对应为 Repository / `@Query`。

## 分页

| 信号 | 做法 |
|------|------|
| MP + 项目 `PageInfo` / `Paging` | `new PageInfo<>(param)` + Mapper 分页 SQL + `new Paging<>(iPage)`；见 [examples.md#分页](examples.md#分页) |
| PageHelper | `PageHelper.startPage()` + 项目分页出参 |
| MyBatis-Plus 原生 | `Page<Entity>` + `selectPage` |
| Spring Data JPA | `Pageable` + `Page<T>` |

示例见 [examples.md#分页](examples.md#分页)。禁止全量查出后 `subList` 内存分页。

## 选型原则

| 场景 | 做法 |
|------|------|
| 单表简单条件 | MP `getById` / `list(wrapper)` 或 XML |
| 多表 JOIN、聚合、统计 | Mapper XML 自定义 SQL |
| 分页含关联名称 | XML 一次返 VO，避免 N+1 |
| 批量写 | `saveBatch` / XML `foreach` |

无 MP 时不用 `Wrappers` / `BaseMapper`。

## 禁止

- 循环内单条查库 → 批量或 XML 一次查
- Service 拼 SQL 字符串 → `#{}` 参数化
- 全量 list 后内存分页 → 数据库分页

## Mapper XML 写法

- `<sql id="Base_Column_List">` 复用列
- 动态条件 `<if>` / `<where>`
- 返 VO 用 `resultType` / `resultMap`，不在 Java 二次拼大 Map
- 改 XML 前考虑索引；`LIKE '%x'` 慎用

## Mapper XML 注释

每个 `<select>` / `<insert>` / `<update>` / `<delete>` 及独立 `<sql>`，须在标签**正上方**写 `<!-- -->`。

| 要素 | 说明 |
|------|------|
| 第一句 | 查/写什么、供哪类场景 |
| 第二句（如有） | 过滤口径、状态枚举、关联表 |
| 性能（如有） | 为何走汇总表、子查询先取 id |

复杂 `<if>` / 子查询 / `CASE` 在片段处补块内注释；写口径不写 SQL 关键字复述。

```xml
<!--
    按状态聚合订单金额。
    有效单：status in (1,2)；金额取自明细表避免主表冗余字段偏差。
-->
<select id="selectAmountAggregation" ...>
```

语气见 [comments.md](comments.md)。与 Service 分工：XML 写 SQL 口径，Service 写流程。

## MyBatis-Plus（仅 MP 项目）

```java
Wrappers.lambdaQuery(Order.class)
    .eq(Order::getUserId, userId)
    .eq(Order::getDeleted, DeletedEnum.NOT_DELETED.getValue())
    .orderByDesc(Order::getId);
```

## 事务与一致性

- 多表写在同一 Service 事务内
- 分布式场景注释说明最终一致、幂等键
- 有乐观锁字段时更新带 `version`

## 金额与时间

- 金额优先整数分，注释标明单位
- 时间类型跟邻 Entity；同模块不混用 `Date` 与 `LocalDateTime`

---

## 附录：JPA 项目补充

仅当检测到 `spring-boot-starter-data-jpa` 时适用：

- 简单 CRUD：`JpaRepository` 派生方法或 `@Query`（参数绑定）
- 复杂报表：`@Query(nativeQuery=true)` 或项目混用的 MyBatis
- 懒加载不在 Controller 触发；展示数据 Service 一次查齐或 DTO 投影
