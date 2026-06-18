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

完整分级表与自检口令见 [comments.md](comments.md)。

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

### 润色 4：配置字段用英文字段名

**前：**

```java
/** companyName 短语前缀匹配权重 */
private float companyNamePhrasePrefix = 10.0f;
```

**后：**

```java
/** 公司名称短语前缀匹配权重，数值越大排序越靠前 */
private float companyNamePhrasePrefix = 10.0f;
```

### 润色 5：Controller 不必 Javadoc 与 Swagger 双写

**前（冗余）：**

```java
/**
 * 按关键词检索订单补全建议。
 * @param param 入参
 * @return 建议列表
 */
@ApiOperation(value = "订单补全", notes = "按关键词返回建议列表")
@GetMapping("/suggest")
public ApiResult<List<OrderVo>> suggest(@Valid OrderSuggestParam param) {
    return ApiResult.ok(orderSuggestService.suggest(param));
}
```

**后（二选一，Swagger 已够用时保留 Swagger）：**

```java
@ApiOperation(value = "订单补全", notes = "按关键词返回建议列表；无匹配时 data 为空列表")
@GetMapping("/suggest")
public ApiResult<List<OrderVo>> suggest(@Valid OrderSuggestParam param) {
    return ApiResult.ok(orderSuggestService.suggest(param));
}
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
