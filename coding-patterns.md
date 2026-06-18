# 编码习惯速查

跨项目通用的 Java 后端习惯。具体包名、基类、工具类以**当前仓库邻代码**为准；与项目 `CLAUDE.md` / rules 冲突时服从项目。

## 类注释

标准三要素与变体见 [comments.md#类注释硬性](comments.md)。切面类须在类注释说明与事务、幂等相关的边界。

## 判空与工具类

动手前看邻文件 import；新代码优先：

| 场景 | 推荐 | 老代码常见（改时对齐邻段） |
|------|------|---------------------------|
| 对象 null | `Objects.isNull()` / `Objects.nonNull()` | `== null`、Hutool `ObjectUtil` |
| 字符串空白 | `commons-lang3` 的 `StringUtils` | 持久层框架自带 `StringUtils` |
| 集合空 | `collections4` 的 `CollectionUtils` | `collections` 老包 |
| 查无则抛 | `Optional.ofNullable(x).orElseThrow(...)` | 多层 if |
| 等值比较 | `Objects.equals(a, b)` | 手写 null 判断 |

```java
if (CollectionUtils.isEmpty(ids)) {
    return Collections.emptyList();
}

// 动态条件：值为 null 时不拼进 WHERE（MyBatis-Plus 示例）
.eq(Objects.nonNull(param.getStatus()), Entity::getStatus, param.getStatus())
```

## Java 8 惯用法（适度）

在 `java.version` 上限内使用，不为炫技：

```java
list.stream().filter(Objects::nonNull).map(Entity::getId).collect(Collectors.toList());
map.stream().collect(Collectors.groupingBy(Entity::getGroupId));
.collect(Collectors.toMap(Entity::getId, Function.identity(), (o, n) -> o));
items.stream().mapToInt(ItemVo::getQuantity).sum();
rows.stream().map(XxxConvert.INSTANCE::entityToVo).collect(Collectors.toList());
```

禁用超出项目 Java 版本的语法，见 [versions.md](versions.md)。`Optional` 不嵌套防御。

## MapStruct

```java
@Mapper
public interface OrderConvert {
    OrderConvert INSTANCE = Mappers.getMapper(OrderConvert.class);
    Order dtoToEntity(OrderDto dto);
}
```

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
