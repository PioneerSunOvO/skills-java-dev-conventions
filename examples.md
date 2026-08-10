# 对照示例

配合 [versions.md](versions.md) 使用。示例中的类名、响应包装均为占位，实际以项目为准。

注释润色前后对照 → [comments-rewrites.md](comments-rewrites.md)  
步骤分块完整范本 → [examples-method-structure.md](examples-method-structure.md)

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

<a id="分页"></a>
<a id="分页写法对照"></a>

## 分页写法对照

### PageHelper

```java
PageHelper.startPage(param.getPageIndex(), param.getPageSize());
List<OrderVo> list = orderMapper.selectOrderVoList(param);
// com.github.pagehelper.PageInfo；再包一层时用项目统一分页出参
PageInfo<OrderVo> pageInfo = new PageInfo<>(list);
return R.ok(pageInfo);
```

### MyBatis-Plus

```java
Page<Order> page = new Page<>(param.getPageIndex(), param.getPageSize());
IPage<Order> result = orderMapper.selectPage(page, wrapper);
// 出参封装类名跟邻代码（示意 PageResult）
return R.ok(new PageResult<>(result));
```

### Spring Data JPA

```java
Pageable pageable = PageRequest.of(param.getPageIndex() - 1, param.getPageSize());
Page<Order> result = orderRepository.findAll(spec, pageable);
return R.ok(result);
```

`R` / `PageResult` 均为占位，以项目为准。分页选型权威见 [versions.md#分页三分支](versions.md#分页三分支)。

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

规则与更多模式见 [coding-patterns.md#MapStruct](coding-patterns.md#MapStruct)。
