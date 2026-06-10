# 数据访问与 SQL

持久层选型与分页分支见 [versions.md](versions.md)。以下以 MyBatis 系为主；JPA 项目将「Mapper/XML」对应为 Repository/`@Query`/原生 SQL。

## 分页三分支（先探测再写）

| 信号 | 做法 |
|------|------|
| `pagehelper-spring-boot-starter` | `PageHelper.startPage(pageIndex, pageSize)` + 项目分页出参 |
| MyBatis-Plus | `Page<Entity>` + `mapper.selectPage(page, wrapper)` |
| Spring Data JPA | `Pageable` + `Page<T>` 或项目 `PageParam` 封装 |

禁止全量查出后 `subList` 内存分页。

## 选型原则

| 场景 | 做法 |
|------|------|
| 单表按主键/简单条件 | MyBatis-Plus `getById` / `list(wrapper)` / `update(wrapper)` |
| 动态单表条件 | `Wrappers.lambdaQuery(Entity.class).eq(...).in(...)`（**仅 MP 项目**） |
| 多表 JOIN、聚合、统计 | Mapper XML 自定义 SQL |
| 分页列表含关联名称 | XML 一次查出 VO，避免 N+1 |
| 批量插入/更新 | `saveBatch` / XML `foreach` |

无 MyBatis-Plus 时，单表与动态条件走 Mapper 接口 + XML，不用 `Wrappers` / `BaseMapper`。

## 禁止

```java
// 差：循环查库
for (Long id : ids) {
    Order o = orderMapper.selectById(id);
    ...
}

// 好：批量
List<Order> orders = orderMapper.selectBatchIds(ids);
```

```java
// 差：Service 里拼 SQL 字符串
String sql = "select * from order where id = " + id;

// 好：参数化查询（XML #{} 或 MP wrapper）
```

```java
// 差：全量查出再 subList 分页
List<XxxVo> all = mapper.selectAll();
return all.subList(start, end);

// 好：数据库分页（见上方三分支）
```

## Mapper XML

- 列定义用 `<sql id="Base_Column_List">` 复用
- 动态条件用 `<if>` / `<choose>`，注意 `WHERE 1=1` 或 `<where>` 标签
- 返回 VO 时 `resultType` 或 `resultMap` 直接映射，不在 Java 里二次组装大 Map
- 改 XML 前想索引与执行计划；`LIKE` 前缀模糊慎用

## MyBatis-Plus 习惯

**仅当项目依赖 MyBatis-Plus 时使用**（无 MP 则用 XML 或项目已有方式）。

```java
// 查询
Wrappers.lambdaQuery(Order.class)
    .eq(Order::getUserId, userId)
    .eq(Order::getDeleted, DeletedEnum.NOT_DELETED.getValue())
    .orderByDesc(Order::getId);

// 更新（只更新非空字段时配合项目工具或手写 set）
Wrappers.lambdaUpdate(Order.class)
    .set(Order::getStatus, status)
    .eq(Order::getId, id);
```

`eq` 前确认字段可为 null 时的语义；避免无意把 `null` 拼进条件。

## JPA 补充

- 简单 CRUD：`JpaRepository` 派生方法或 `@Query`（参数绑定，禁止拼接用户输入）
- 复杂列表/报表：优先 `@Query(nativeQuery=true)` 或独立 MyBatis Mapper（若项目混用）
- **懒加载关联不要在 Controller 层触发**；需要的数据在 Service 一次查齐或 DTO 投影

## 事务与一致性

- 多表写、先写主表再写子表：同一 Service 方法内同一事务
- 分布式场景下的「最终一致」在注释里说明，代码里体现幂等键或唯一约束
- 乐观锁、版本号字段若项目有，更新时带上 `version`

## 金额与时间

- 金额：优先整数分（`Long`/`Integer`），注释标明单位；避免 `double`
- 时间类型与 JSON 序列化跟 [versions.md](versions.md)「时间类型与 JSON」及邻 Entity 字段一致
- 同一模块不混用 `Date` 与 `LocalDateTime`
