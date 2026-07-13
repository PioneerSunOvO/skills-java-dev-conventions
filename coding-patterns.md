# 编码习惯速查

跨项目通用的 Java 后端习惯。具体包名、基类、工具类以**当前仓库邻代码**为准；与项目 `CLAUDE.md` / rules 冲突时服从项目。

## 类注释

标准三要素与变体见 [comments.md#类注释硬性](comments.md)。切面类须在类注释说明与事务、幂等相关的边界。

## 判空与工具类

> **硬性**：新代码禁止 `== null` / `!= null` 判对象；禁止对可能 null 的集合直接 `.isEmpty()`。动手前看邻文件 import，包名跟邻类。

### 场景对照

| 场景 | 必须用 | 禁止 |
|------|--------|------|
| 对象 null | `Objects.isNull()` / `Objects.nonNull()` | `== null`、`!= null` |
| 字符串空白 | `commons-lang3` 的 `StringUtils.isBlank()` / `isNotBlank()` | `str == null \|\| str.isEmpty()`、持久层自带 `StringUtils`（新代码不用） |
| 集合/Map 空 | `CollectionUtils.isEmpty()` / `isNotEmpty()` | `list == null \|\| list.isEmpty()`、`map.size() == 0` |
| 等值比较 | `Objects.equals(a, b)` | `a != null && a.equals(b)` |
| 查无则抛 | `Optional.ofNullable(x).orElseThrow(...)` | `if (x == null) throw …` 多层嵌套 |
| 查无给默认 | `Optional.ofNullable(x).orElse(def)` | 冗长三元（简单字面量三元可保留） |

老代码常见 `== null`、Hutool `ObjectUtil`、MyBatis `StringUtils`——**改邻段时对齐邻段**；**新写片段**仍用上表。

### 示例

```java
// 集合早返回
if (CollectionUtils.isEmpty(ids)) {
    return Collections.emptyList();
}

// 对象判空 + 业务异常
if (Objects.isNull(order)) {
    throw new BusinessException("订单不存在");
}

// Optional：查库无则抛（Lambda 内抛项目业务异常）
Order order = Optional.ofNullable(getById(id))
        .orElseThrow(() -> new BusinessException("订单不存在"));

// Optional：空则给默认（替代 getXxx() != null ? getXxx() : ""）
String remark = Optional.ofNullable(dto.getRemark()).orElse("");

// 动态条件：值为 null 时不拼进 WHERE（MyBatis-Plus 示例）
.eq(Objects.nonNull(param.getStatus()), Entity::getStatus, param.getStatus())
.in(CollectionUtils.isNotEmpty(param.getIds()), Entity::getId, param.getIds())
```

### Optional 边界

| 适合 | 不适合 |
|------|--------|
| `getById` / `getOne` 查无则抛 | 普通 if 分支里已有明确 null 处理 |
| 链式取值给默认值（`orElse` / `orElseGet`） | 层层 `Optional.ofNullable(a).map(...).map(...)` 防御 |
| `filter(Objects::nonNull)` 配合 Stream | 把每个局部变量都包一层 Optional |

`Optional` 不嵌套防御；无 null 风险的变量不必包 Optional。

## Stream 流式处理

> **原则**：可读性第一。能**明显**简化集合转换、过滤、聚合时用 Stream；不过度函数式、不为炫技。

### 适合用 Stream

| 场景 | 示例思路 |
|------|----------|
| 列表 → 另一列表 | `filter` + `map` + `collect` |
| 分组 / 转 Map | `Collectors.groupingBy` / `toMap` |
| 数值聚合 | `mapToInt(...).sum()`、`reduce` |
| Entity → VO 批量映射 | `map(XxxConvert.INSTANCE::entityToVo).collect(...)` |
| 去 null 后转换 | `filter(Objects::nonNull).map(...)` |

```java
List<Long> ids = orders.stream()
        .filter(Objects::nonNull)
        .map(Order::getId)
        .collect(Collectors.toList());

Map<Long, List<OrderDetail>> grouped = details.stream()
        .collect(Collectors.groupingBy(OrderDetail::getOrderId));

int totalQty = items.stream().mapToInt(ItemVo::getQuantity).sum();
```

### 保留 for / 增强 for

| 场景 | 原因 |
|------|------|
| 2～3 行简单遍历 | for 更直观，不必强行 Stream |
| 循环内多分支、`continue`/`break` | Stream 难读 |
| 需要下标 | 传统 `for` 或 `IntStream.range` |
| 循环体有副作用（写库、累加外部状态、发消息） | Stream 易藏 N+1，语义也不清晰 |
| 业务校验需逐步早返回 | 抽私有方法 + for，或卫语句 |

```java
// 简单遍历：for 足够清晰时不必改 Stream
for (OrderDetail detail : details) {
    validateDetail(detail);
}

// 反例：forEach 里查库 → N+1
ids.forEach(id -> orderMapper.selectById(id));  // 禁止
```

### 硬性边界

- **禁止** `stream` / `forEach` 内单条查库、RPC、发 MQ（与 for 循环同理，见 data-access.md）
- 链式操作 **≤ 4 步**；更长则拆中间变量或抽 `buildXxxList(...)` 私有方法
- **禁止** `peek` 承载业务逻辑；**禁止**多层嵌套 Stream（`flatMap` 套 `flatMap` 套 `map`）
- **禁止**默认 `parallelStream()`（除非有明确性能依据且邻模块已有先例）
- 收集结果跟 JDK 版本：Java 8 用 `.collect(Collectors.toList())`；高版本见 [versions.md](versions.md)

### 可读性自检（30 秒）

1. 去掉 Stream 后，等价 for 是否**更短或一样短**？→ 优先 for  
2. 链上能否一眼看出「输入 → 过滤什么 → 输出什么」？→ 否则拆分  
3. 有没有在 lambda 里写超过 3 行业务？→ 抽方法引用或私有方法  

## MapStruct

```java
@Mapper
public interface OrderConvert {
    OrderConvert INSTANCE = Mappers.getMapper(OrderConvert.class);
    Order dtoToEntity(OrderDto dto);
}
```

- 转换类命名统一使用 `*Convert` 后缀（如 `OrderConvert`），不使用 `*Converter`、`*MapperStruct` 等变体
- 列表不同映射目标：`@Named` + `@BeanMapping(qualifiedByName = ...)`
- 枚举展示名：`@Mapping(expression = "java(...)")` + `@Mapper(imports = {...})`
- 多表 VO：**Mapper XML 一次查出**，不在 Service 循环拼

详见 [examples.md](examples.md)、[architecture.md](architecture.md)。

## Service 模式

| 模式 | 做法 |
|------|------|
| 写操作校验 | `save`/`update` 入口调 `validateXxx` |
| 复杂分页 VO | Mapper XML / 自定义 SQL 直出 VO |
| 审计字段 | 按项目：MetaObjectHandler 或 Service 手动赋值 |
| 批处理容错 | 单条 catch + log.error，注释说明续跑策略 |
| 同类自调用 | 注入自身代理，保证 AOP / 事务生效 |

## 异常捕获与堆栈

- `try-catch` 统一捕获 `Exception`，不捕获具体子异常（除非项目或用户明确指定）
- 捕获后使用异常工具类打印完整堆栈，例如 `ExceptionUtils.printRootCauseStackTrace(e)`
- 禁止只记录 `e.getMessage()` 或吞掉堆栈，避免排障信息缺失

## AOP 与副作用

- 副作用（同步缓存、写衍生表）优先 AOP 或 Service 内显式调用，与邻模块一致
- 切面须在类注释说明相对事务的顺序
- 不在 Controller 里散落副作用逻辑

## 对象边界

| 方向 | 类型 |
|------|------|
| API 入参 | `Param`（分页）、`DTO`（非分页） |
| API 出参 | `VO` + 统一响应包装 |
| ORM | `Entity`，不进 Controller |
| 边界转换 | MapStruct `XxxConvert.INSTANCE` |

## 读邻代码时留意

不同仓库可能并存多种风格，**新代码跟同模块邻类**：

- 多种 `CollectionUtils` / `StringUtils` 包名
- 字段 `@Autowired` vs Lombok `@Setter` 注入
- GET vs POST 读接口
- 分页出参封装类名

不擅自统一全库风格，除非用户明确要求重构。

## 方法结构

见 [comments.md](comments.md)（分级表、Javadoc、步骤分块）与 [method-structure.md](method-structure.md)（编排与私有方法抽取）。

## 注解排列

> **版本无关**：与 JDK、Spring Boot / Cloud 代际、API 文档体系（Swagger2 / springdoc / 无文档注解）无关。适用于任意 Java 注解——Spring Web、JPA、Bean Validation、Lombok、安全、项目自定义等，**只排顺序，不规定必须用哪些注解**。

同一声明处（类、方法、字段、参数等）挂多个注解时，**按注解整行文本从短到长**自上而下排列。

**计量方式**：以源码中 `@` 起至该行注解结束的字符数为准（含属性、括号、引号）；多行注解按首行计。

**推荐**（类级；下列注解仅示意顺序，以当前项目实际使用的为准）：

```java
@Slf4j
@RestController
@RequestMapping("/orders")
public class OrderController {
```

**推荐**（方法级）：

```java
@Override
@GetMapping("/{id}")
@Transactional(rollbackFor = Exception.class)
public OrderVo get(@PathVariable Long id) {
```

**推荐**（参数级）：

```java
public void update(@PathVariable Long id, @Valid @RequestBody OrderDto dto) {
```

**不推荐**（长短交错）：

```java
@Transactional(rollbackFor = Exception.class)
@GetMapping("/{id}")
```

补充：

- 长度相同时，与邻方法一致即可，不必强行微调
- 项目特有注解（操作日志、模块标记、权限等）与框架注解混排时，同样按长度排序
- 改邻代码时不必为统一风格大面积重排既有注解，除非用户明确要求格式化

## 长行换行

一行过长需换行时，**续行用固定缩进**（相对语句起始再缩进一级，通常 +4 空格），保持左缘整齐；避免 IDE「与首参数/括号对齐」造成大片左侧空白，也避免在 `(` 后立刻断行、下一行堆满参数导致首行只剩半截、视觉上割裂。

**推荐**（每参数一行，续行缩进一致）：

```java
MethodBasedEvaluationContext context = new MethodBasedEvaluationContext(
        joinPoint.getTarget(),
        method,
        joinPoint.getArgs(),
        parameterNameDiscoverer);
```

**不推荐**（与首参数对齐，左侧空洞）：

```java
MethodBasedEvaluationContext context = new MethodBasedEvaluationContext(joinPoint.getTarget(),
                                                                            method,
                                                                            joinPoint.getArgs(),
                                                                            parameterNameDiscoverer);
```

**不推荐**（`(` 后立刻断行，首行语义不完整）：

```java
MethodBasedEvaluationContext context = new MethodBasedEvaluationContext(
        joinPoint.getTarget(), method, joinPoint.getArgs(), parameterNameDiscoverer);
```

补充习惯：

- 在逗号、运算符、`=` 后等**语义完整处**断行，不在类型名与 `(` 之间硬拆
- 链式调用：`.` 起新行，或按调用链分段，续行缩进一致
- 能在一行内读完（约 120 字符内）则不必强行换行
