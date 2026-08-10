# 编码习惯速查

跨项目通用习惯。包名/基类/工具类以**邻代码**为准；与项目 rules 冲突时服从项目。

## 类注释

三要素见 [comments.md#类注释硬性](comments.md#类注释硬性)。切面类须说明与事务、幂等边界。

<a id="判空与工具类"></a>

## 判空与工具类

> **强约束**：新代码禁 `== null` / `!= null` 判对象；禁对可能 null 的集合直接 `.isEmpty()`。import **跟邻文件**。
>
> **Why**：语义一致、避免 NPE；**只约束本次新写片段**，禁止为统一风格批量改旧判空。

| 场景 | 必须用 | 禁止 |
|------|--------|------|
| 对象 null | `Objects.isNull` / `nonNull` | `== null` / `!= null` |
| 字符串空白 | lang3 `StringUtils.isBlank` | `str == null \|\| str.isEmpty()`；新代码不用持久层 `StringUtils` |
| 集合/Map 空 | `CollectionUtils.isEmpty`（包名跟邻） | `list == null \|\| list.isEmpty()` |
| 等值 | `Objects.equals` | `a != null && a.equals(b)` |
| 查无则抛 / 默认 | `Optional.ofNullable(...).orElseThrow` / `orElse` | 多层 `if null`；层层 Optional 防御 |

老代码 `== null` / Hutool / MyBatis `StringUtils`：**改邻段时对齐邻段**；新写仍用上表。

<a id="多工具库并存"></a>

### 多工具库并存

```
1. 看本文件 import 主流用哪套
2. 新写跟同文件邻段，不引入第三套
3. 新模块无邻例 → JDK + commons-lang3
4. 禁止同方法混用 Objects.isNull 与 ObjectUtil.isNull
5. 禁止仅为规范整文件改判空
```

`CollectionUtils` / `StringUtils` 多包同名 → **以邻文件为准**。

`Optional`：适合「一次取值 + 抛/默认」；不适合无 null 风险的层层 `.map`。

<a id="stream-流式处理"></a>

## Stream 流式处理

> 可读优先：能**明显**简化转换/过滤/聚合再用；不为炫技。

| 适合 Stream | 保留 for |
|-------------|----------|
| `filter/map/collect`、分组/`toMap`、数值聚合 | 2～3 行简单遍历；多分支/`break`；要下标 |
| Entity→VO 批量映射 | 循环内写库/副作用；逐步早返回的校验 |

**硬门禁**：禁 `stream`/`forEach` 内单条查库/RPC/MQ（N+1）；链 ≤ 4 步；禁 `peek` 写业务；禁嵌套 Stream；禁默认 `parallelStream`。Java 8 用 `.collect(Collectors.toList())`。

自检：去掉 Stream 后 for 更短？→ 用 for。lambda 超 3 行业务？→ 抽方法。

<a id="MapStruct"></a>

## MapStruct

```java
@Mapper
public interface BizConvert {
    BizConvert INSTANCE = Mappers.getMapper(BizConvert.class);
    BizEntity dtoToEntity(BizDto dto);
}
```

- 后缀统一 `*Convert`
- 多目标：`@Named` + `@BeanMapping(qualifiedByName=...)`
- 枚举名：`expression` + `@Mapper(imports=...)`
- 多表 VO：XML 一次查，不在 Service 循环拼

示例见 [examples.md](examples.md)；分层见 [architecture.md](architecture.md)。

## Service / 异常 / AOP（摘要）

| 项 | 做法 |
|----|------|
| 写入口 | `validateXxx`；复杂列表 VO 用 XML 直出 |
| 审计 | MetaObjectHandler 或 Service 手动，跟邻 Entity |
| 自调用事务 | 注入自身代理 |
| catch | 跟邻模块；日志末参带 `Throwable`；禁吞异常 / 只打 `getMessage()` |
| 副作用 | AOP 或 Service 显式调用；不写在 Controller |

API 入参 Param/DTO，出参 VO+统一包装，Entity 不进 Controller。

## 读邻代码时

新代码跟同模块：`CollectionUtils` 包名、注入风格、GET/POST、分页封装类名。不擅自全库统一，除非用户要求重构。

方法结构 → [comments.md](comments.md)、[method-structure.md](method-structure.md)。

<a id="注解排列"></a>

## 注解排列（风格建议）

> **Why**：短上长下更好扫读；与框架代际无关。

同一声明处多个注解，按**整行文本从短到长**排列（多行注解按首行计）。长度相同跟邻方法即可。改邻代码不顺手大面积重排。

```java
@Slf4j
@RestController
@RequestMapping("/orders")
public class OrderController { }

@Override
@GetMapping("/{id}")
@Transactional(rollbackFor = Exception.class)
public BizVo get(@PathVariable Long id) { }
```

## 长行换行（风格建议）

续行用**固定缩进**（通常 +4），不在 `(` 后立刻断行堆参，不用「与首参数对齐」。能在约 120 字符内读完则可不换行。链式调用 `.` 起新行，缩进一致。
