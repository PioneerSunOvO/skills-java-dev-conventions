# 对照示例

配合 [versions.md](versions.md) 使用。示例中的类名、响应包装均为占位，实际以项目为准。

## 版本探测示例

**给定 pom 片段：**

```xml
<properties>
    <java.version>1.8</java.version>
    <spring-boot.version>2.2.5.RELEASE</spring-boot.version>
    <mybatis-plus-boot-starter.version>3.3.1</mybatis-plus-boot-starter.version>
</properties>
```

**输出环境画像：**

```text
Java: 8
Spring Boot: 2.2.5（2.x 代际）
持久层: MyBatis-Plus 3.3
校验包: javax.validation（待邻文件 import 确认）
禁用: var, record, List.of(), String.isBlank(), jakarta.*
分页/API/注入: 需抽样邻 Controller、Service 确认
```

**给定 pom 片段（SB3）：**

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>
<properties>
    <java.version>17</java.version>
</properties>
```

**输出环境画像：**

```text
Java: 17
Spring Boot: 3.2（3.x 代际）
校验包: jakarta.validation
可用: record, switch 表达式, var（若团队允许）
API 文档: 探测 springdoc-openapi，不用 Swagger2 注解
```

---

## 语法降级示例

**Java 17+ 写法（不可用于 Java 8 项目）：**

```java
public record OrderSummary(Long orderId, Long amountFen) {}

var list = orders.stream().filter(o -> o.getPaid()).toList();

if (obj instanceof OrderDto dto) {
    return dto.getId();
}
```

**Java 8 等价：**

```java
@Data
public class OrderSummary {
    private final Long orderId;
    private final Long amountFen;
}

List<Order> list = orders.stream()
    .filter(o -> o.getPaid())
    .collect(Collectors.toList());

if (obj instanceof OrderDto) {
    OrderDto dto = (OrderDto) obj;
    return dto.getId();
}
```

---

## 持久层分支对照

同一需求：按用户 ID 查未删除订单列表。

### MyBatis-Plus

```java
List<Order> orders = orderMapper.selectList(
    Wrappers.lambdaQuery(Order.class)
        .eq(Order::getUserId, userId)
        .eq(Order::getDeleted, DeletedEnum.NOT_DELETED.getValue())
        .orderByDesc(Order::getId)
);
```

### 原生 MyBatis（无 MP）

```java
// Mapper 接口
List<Order> selectByUserId(@Param("userId") Long userId);
```

```xml
<select id="selectByUserId" resultType="com.example.entity.Order">
    SELECT id, user_id, status, deleted
    FROM t_order
    WHERE user_id = #{userId} AND deleted = 0
    ORDER BY id DESC
</select>
```

### Spring Data JPA

```java
List<Order> orders = orderRepository.findByUserIdAndDeletedOrderByIdDesc(
    userId, DeletedEnum.NOT_DELETED.getValue()
);
```

---

## javax vs jakarta 校验示例

**Spring Boot 2.x：**

```java
import javax.validation.constraints.NotBlank;
import javax.validation.constraints.NotNull;
```

**Spring Boot 3.x：**

```java
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
```

---

## 分页写法对照

### PageHelper

```java
PageHelper.startPage(param.getPageIndex(), param.getPageSize());
List<OrderVo> list = orderMapper.selectOrderVoList(param);
return new Paging<>(new PageInfo<>(list));
```

### MyBatis-Plus

```java
Page<Order> page = new Page<>(param.getPageIndex(), param.getPageSize());
IPage<Order> result = orderMapper.selectPage(page, wrapper);
return new Paging<>(result);
```

### Spring Data JPA

```java
Pageable pageable = PageRequest.of(param.getPageIndex() - 1, param.getPageSize());
Page<Order> result = orderRepository.findAll(spec, pageable);
```

实际出参封装类名以项目为准（`Paging`、`PageResult` 等）。

---

## 注释润色（AI → 自然中文）

### 润色 1：空洞类注释

**前：**

```java
/**
 * 该类用于处理订单相关的业务逻辑。
 * 主要功能包括：订单查询、订单更新、订单删除。
 */
```

**后：**

```java
/**
 * @description: 订单衍生数据服务实现；汇总口径与统计 SQL 一致
 * @author: 开发者
 * @since: 2026-06-09 10:00
 */
```

### 润色 2：机翻方法注释

**前：**

```java
/**
 * 值得注意的是，本方法主要用于处理支付回调。
 * 此外会对支付流水进行校验，确保数据一致性。
 */
```

**后：**

```java
/**
 * 支付回调处理；同一 outTradeNo 重复通知时幂等返回，不重复更新订单状态。
 */
```

### 润色 3：复述代码的行注释

**前：**

```java
// 查询未删除的
List<Order> list = mapper.selectList(wrapper);
```

**后：**

```java
// 列表只展示未删除记录，与前台「有效数据」口径一致
List<Order> list = mapper.selectList(wrapper);
```

---

## 步骤分块示例（切面方法）

```java
/**
 * 业务成功后解析上下文，按配置执行同步或清理。
 *
 * @param joinPoint 切点
 * @param marker    业务标记注解
 */
@AfterReturning("@annotation(marker)")
public void afterSuccess(JoinPoint joinPoint, BusinessMarker marker) {
    // 解析条件表达式，不通过则不执行
    if (!evaluateCondition(marker.condition(), joinPoint)) {
        return;
    }

    // 解析业务主键列表
    List<Long> ids = resolveIds(marker, joinPoint);
    if (ids.isEmpty()) {
        return;
    }

    // 按动作类型执行同步或删除
    for (Long id : ids) {
        if (Objects.isNull(id)) {
            continue;
        }
        if (ActionType.DELETE == marker.action()) {
            syncService.deleteById(id);
        } else {
            syncService.rebuildById(id);
        }
    }
}
```

要点：Javadoc 概括全流程；体内每步一行总注释；块间空一行；总注释含边界。

---

## 步骤分块示例（Service 方法）

```java
/**
 * 重算并落库衍生数据。
 * 主记录不存在或明细为空时清空；有数据则先删后插。
 *
 * @param id 业务主键
 */
public void rebuildById(Long id) {
    // 主键为空直接返回
    if (Objects.isNull(id)) {
        return;
    }

    // 加载主记录与明细
    Order order = orderService.getById(id);
    List<OrderDetail> details = detailService.listByOrderId(id);

    // 无记录或无明细时清空衍生数据
    if (Objects.isNull(order) || CollectionUtils.isEmpty(details)) {
        deleteById(id);
        return;
    }

    // 按业务规则计算衍生行
    List<SummaryRow> rows = buildSummaryRows(order, details);

    // 先删后插，保证与源数据一致
    deleteById(id);
    if (CollectionUtils.isNotEmpty(rows)) {
        this.saveBatch(rows);
    }
}
```

---

## 步骤分块示例（多数据源查询）

```java
/**
 * 查询复合列表，含当期数据、历史关联及统计数量。
 *
 * @param param 查询条件
 * @return 组装后的列表
 */
public List<CatalogVo> getList(QueryParam param) {
    // 解析查询范围
    Integer year = param.getYear();
    List<Long> categoryIds = resolveCategoryIds(param);

    // 加载当期主数据
    List<ItemVo> currentItems = baseMapper.selectItemsByYear(year, categoryIds);

    // 加载跨期关联数据
    List<ItemVo> allItems = baseMapper.selectItemsByYears(year, year - 1);

    // 构建历史关联索引
    // 取当期与主数据的交集键，再关联历史记录
    Map<Long, ItemVo> historyItemMap = buildHistoryMap(allItems, currentItems, year);

    // 查询历史统计数量
    Map<Long, Map<Long, Integer>> historyQtyMap =
            loadHistoryQuantities(year - 1, categoryIds, historyItemMap.keySet());

    // 按分类组装返回列表
    for (CatalogVo catalog : catalogList) {
        assembleCatalog(catalog, currentItems, historyItemMap, historyQtyMap);
    }
    return catalogList;
}
```

分块规则见 [method-structure.md](method-structure.md)。

---

## MapStruct expression 示例

```java
@Mapper(imports = {StatusEnum.class})
public interface ReportConvert {
    ReportConvert INSTANCE = Mappers.getMapper(ReportConvert.class);

    @Mapping(target = "statusName",
             expression = "java(StatusEnum.getDescByCode(detail.getStatus()))")
    ReportRowVo detailToRow(ReportDetail detail);
}
```

Service 调用：

```java
List<ReportRowVo> rows = details.stream()
        .map(ReportConvert.INSTANCE::detailToRow)
        .collect(Collectors.toList());
```

更多模式见 [coding-patterns.md](coding-patterns.md)。

---

## Git 提交说明示例

均来自本仓库实际风格；撰写新 message 时对齐语气与结构。规范摘要见 [SKILL.md#git-提交说明](SKILL.md#git-提交说明magapi)。

### 示例 1：新功能，多条能力（编号列表）

```
feat(订单): 新增征订汇总表，并在订单变更时自动同步汇总数据

1. 新增 imo_magorder_subscribe_summary 征订汇总表，按订单明细预计算全年征订、破订、图书等结果，便于列表和统计查询
2. 订单保存、修改、删除、作废、调整折扣后，自动按最新明细重算并更新汇总表
3. 新增管理端全量重建接口，用于历史订单一次性补算和数据修复
```

### 示例 2：规则扩展（编号 + 业务字段）

```
feat(订单): 支持全年订期代销订单、导入指定订单类型及代销折扣不改类型

1. 全年订期新增/修改订单时，允许提交代销订单（moType=11）
2. 代销订单修改折扣时保持订单类型不变，不再自动转为折扣订单
3. 动态订单导入新增 moType 入参，支持征订订单（普通）与代销订单导入
```

### 示例 3：查询重构（通俗说清「之前 / 之后」）

```
refactor(订单): 优化重复订单查重，改为主表与明细表一次关联查询

1. 取消「先查明细订单号、再按订单号列表过滤」的两步查询，减少数据库往返和大列表查询开销
2. 新增订单主表关联明细表查询，按收件人五要素与订期同时匹配，一次完成重复订单判断
3. 移动端查重补充当前登录用户限制，仅检索本人创建的订单
```

### 示例 4：跨接口改动（`-` 列表亦可）

```
feat(来款): 退款科目改为读取配置动态查询生成；新增在线退款接口

- confirmRefund 退款科目由硬编码改为按 subjectType=退款 动态匹配
- 新增 /onlineRefund，调用退款后走与 confirmRefund 相同收尾逻辑
- 抽取公共收尾方法，供线下确认与在线退款共用
```

### 反例（不要这样写）

```
feat(订单): 新增 MagOrderSubscribeSummary 及 SyncMagOrderSubscribeSummary AOP

1. 新增 Entity 和 Mapper
2. 新增 Aspect 和 Annotation
3. Controller 加 rebuild 方法
```

### 快速模板

```
<type>(<模块>): <业务结果一句话>

1. <能力/规则 1>
2. <能力/规则 2>
3. <能力/规则 3>（按需增减）
```
