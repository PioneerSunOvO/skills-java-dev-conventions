# 编码习惯速查

跨项目通用习惯。包名/基类/工具类以**邻代码**为准；与项目 rules 冲突时服从项目。

**示例约定**：类名/包名/异常均为占位（`Biz*`、`Xxx*`）；不写某一仓库真实业务类。项目专属封装见 **项目 quickref** 或 `CLAUDE.md`。

**本节索引**（权威细则在此；其它文件只链入）：

| 锚点 | 内容 |
|------|------|
| [#lombok](#lombok) | 数据类样板 |
| [#判空与工具类](#判空与工具类) | 判空、比较、字符串/数值/日期/集合 |
| [#optional](#optional) | 空值表达 |
| [#stream-流式处理](#stream-流式处理) | Stream |
| [#MapStruct](#MapStruct) | Bean 拷贝（禁 BeanUtils） |
| [#富枚举与入参校验](#富枚举与入参校验) | 状态语义、枚举取值校验 |
| [#多供应商模板方法](#多供应商模板方法) | 渠道/SDK 适配 |
| [#耗时日志-stopwatch](#耗时日志-stopwatch) | StopWatch |

类注释三要素 → [comments.md#类注释硬性](comments.md#类注释硬性)。

<a id="lombok"></a>

## Lombok（数据类样板）

> **强约束**：Entity / DTO / Param / VO / PO 等**纯数据载体**用 Lombok 生成 accessor/`equals`/`hashCode`/`toString`；**禁止**手写。只约束新写/触及片段。

| 场景 | 推荐 |
|------|------|
| 普通数据类 | `@Data` |
| 继承基类 | `@Data` + `@EqualsAndHashCode(callSuper = true)` |
| 链式 set | `@Accessors(chain = true)`（import 跟邻） |
| 仅读/仅写 | `@Getter` / `@Setter`（邻文件已分离时可跟邻） |
| 构造 | `@NoArgsConstructor` / `@AllArgsConstructor` / `@Builder`（跟邻） |

**禁止**：手写 getter/setter；`@Data` 用于 Service/Controller；Lombok 已覆盖能力再写重复方法；未触及存量整文件删 accessor。

<a id="判空与工具类"></a>
<a id="工具类优先"></a>

## 判空 / 比较 / 工具类优先

> **强约束**：判空、比较、集合/字符串/数值/日期等样板逻辑**尽可能**用工具类；禁 `== null`、裸 `.equals`、冗长 for。只约束新写/触及片段。
>
> **工具来源**：JDK 优先；**项目已有** commons-lang3、commons-collections4、Spring util、Guava、Hutool 等可用（跟邻 import）。**禁止**仅为规范新引依赖；无邻例 → JDK + commons-lang3。

### 场景速查（统一表）

| 类别 | 场景 | 优先 | 禁止 |
|------|------|------|------|
| 判空 | 对象 null | `Objects.isNull` / `nonNull` | `== null` |
| 判空 | 字符串空白 | lang3 `StringUtils.isBlank` / `isNotBlank` | `str == null \|\| isEmpty()` |
| 判空 | 集合/Map/数组空 | `CollectionUtils.isEmpty`；数组 `ArrayUtils.isEmpty` | `list == null \|\| isEmpty()` |
| 判空 | 集合/Map 非空 | `CollectionUtils.isNotEmpty` | 双重判空后 `!isEmpty()` |
| 判空 | 对象默认值 | `ObjectUtils.defaultIfNull` 或 `Optional` | 多层 `if null` |
| 比较 | 对象等值 | `Objects.equals(a, b)` | 裸 `a.equals(b)` |
| 比较 | 字符串等值 | `StringUtils.equals` / `equalsIgnoreCase` | 不防 null 的 equals |
| 比较 | 前缀/后缀/包含 | `StringUtils.startsWith` / `contains` | 先手写判空再调 |
| 比较 | 集合/数组包含 | `CollectionUtils.contains` / `ArrayUtils.contains` | null 集合直接 contains |
| 比较 | 包装类型 | `Objects.equals` / `NumberUtils.compare` | `Integer` 用 `==` |
| 比较 | 金额 `BigDecimal` | `compareTo == 0` / `> 0` / `< 0` | `equals`（scale 陷阱） |
| 比较 | 布尔包装 | `Boolean.TRUE.equals(flag)` / `BooleanUtils.isTrue` | `flag == true` |
| 字符串 | 默认/trim | `defaultIfBlank` / `trimToEmpty` | 手写三元 trim |
| 字符串 | 拼接/拆分 | `StringUtils.join` / `split`；`String.join` | 循环 `+` 拼长串 |
| 数值 | 解析/判数字 | `NumberUtils.toInt` / `isCreatable` | 无兜底 `parseInt` |
| 数值 | 金额运算 | `BigDecimal` + 显式 `scale`/`RoundingMode` | `double` 算钱 |
| 日期 | 解析/格式化/加减 | 项目工具；`DateUtil`/`DateUtils`/`LocalDate`（跟邻） | 每次 `new SimpleDateFormat` |
| 集合 | 空兜底/添加 | `emptyIfNull` / `addAll`（先判空） | null 直接 `addAll` |
| 集合 | 过滤/合并/Map | `filter(Objects::nonNull)`、Guava/commons 集合工具、`getOrDefault` | 双层 for 去重合并 |
| 集合 | 单元素/空常量 | `singletonList` / `emptyList()` | `new ArrayList<>()` 当常量 |
| 杂项 | 三态布尔/首非空 | `BooleanUtils.*` / `ObjectUtils.firstNonNull` | 多层三元 |
| 杂项 | 要求非空 | `Objects.requireNonNull(x, "msg")` | 无信息 NPE |
| 杂项 | Bean 拷贝 | **MapStruct** → [#MapStruct](#MapStruct) | BeanUtils / BeanUtil |

```java
// 好
if (CollectionUtils.isEmpty(ids)) return Collections.emptyList();
if (!Objects.equals(oldVal, newVal)) { ... }
if (paid.compareTo(expected) != 0) throw new BizException("金额不一致");

// 差
if (list != null && !list.isEmpty()) { ... }
if (order.getStatus().equals(target)) { ... }
```

<a id="多工具库并存"></a>

### 多工具库并存

1. 看本模块 import 主流用哪套 → 新写跟邻段，不引入第三套  
2. 无邻例 → JDK + commons-lang3  
3. 禁止同方法混用 `Objects.isNull` 与 `ObjectUtil.isNull`、两套 `CollectionUtils`/`StringUtils`  
4. 禁止仅为规范整文件改写法或换库  

<a id="optional"></a>

## Optional（空值表达）

> **强约束**：**一次**取值 + 抛/默认/映射时用 `Optional`；禁 DTO 字段、禁层层嵌套、禁替代所有判空。

| 适合 | 保留 if/工具类 |
|------|----------------|
| 查无则抛：`.orElseThrow(() -> new BizException("…"))` | 多条件早返回、集合判空 |
| 默认值：`.orElse` / `.orElseGet` | `CollectionUtils.isEmpty(list)` |
| 映射：`.map(X::getY).orElse("")` | 两字段同时非空 |
| 副作用：`.ifPresent(v -> …)` | |
| `findFirst` 终端：直接 `orElseThrow` | |

**禁止**：`.get()` 无守卫；字段类型用 `Optional`；Stream 里 `Optional.ofNullable` 套娃；整文件把 `if null` 改 Optional。

<a id="stream-流式处理"></a>

## Stream 流式处理

> **硬门禁**：禁流内查库/RPC/MQ；链 ≤ 4 步；禁 `peek` 写业务；禁嵌套 Stream；禁 `parallelStream`。  
> **可读**：过滤/映射/分组/聚合/去重/收集能看懂则优先 Stream；去掉后 for 更短则用 for。

| 适合 Stream | 保留 for |
|-------------|----------|
| `filter`/`map`/`collect`、`groupingBy`、`toMap`、聚合 | 多分支/break/下标/副作用/循环写库 |
| 内存 Entity→VO（无 I/O） | lambda > 3 行业务 |

```java
List<Long> ids = rows.stream().filter(Objects::nonNull).map(BizEntity::getId).collect(Collectors.toList());
Map<Integer, List<BizVo>> byStatus = list.stream().collect(Collectors.groupingBy(BizVo::getStatus));
```

- Java 8：`.collect(Collectors.toList())`，不用 `Stream.toList()`  
- 链内去 null：`.filter(Objects::nonNull)`，不用 Optional 套娃  
- 需副作用 → for；需查库 → 先 `collect` id 再批量查  

<a id="MapStruct"></a>

## MapStruct（Bean 拷贝）

> **强约束**：DTO/Param/Entity/VO 映射**必须** MapStruct（`*Convert`）；**禁止**新写 BeanUtils/BeanUtil 反射拷贝。  
> **Why**：编译期显式 get/set，IDE 可查找引用；反射按属性名匹配，**字段引用无法静态排查**。

```java
@Mapper
public interface BizConvert {
    BizConvert INSTANCE = Mappers.getMapper(BizConvert.class);
    BizEntity dtoToEntity(BizDto dto);
}
```

| 项 | 约定 |
|----|------|
| 调用 | `BizConvert.INSTANCE.dtoToEntity(dto)`（或邻代码注入式 `@Mapper`） |
| 边界 | 字段映射与**无 I/O 的组装**可放 Convert；状态机、金额规则、查库放 Service → [architecture.md](architecture.md) |
| 进阶 | `@Named`、`expression`；多表 VO → XML 一次查；多源挂载用 `default` + 内存分组 |
| 存量 | 未触及不改；**新写/改到的映射**一律 Convert |

示例 → [examples.md](examples.md)；分层 → [architecture.md](architecture.md)。

<a id="富枚举与入参校验"></a>

## 富枚举与入参校验（强约束）

> 业务状态/类型枚举承载语义；HTTP 入参用校验注解约束取值。注解类名跟邻（`@InEnum` / `@EnumValue` / 自定义均可）。

| 要点 | 做法 |
|------|------|
| **常量注释** | 每个业务枚举常量写含义（见 [comments.md](comments.md)） |
| **取值暴露** | 需要给校验器用时，提供全部合法值数组（或项目等价 API） |
| **当前态 vs 经历态** | `isXxx(status)` = 正处于；`haveXxx(status)` = 生命周期是否经历过；集中在枚举旁，禁业务散落魔法数 |
| **可检索** | 谓词方法集中在枚举，便于 IDE 查「谁在依赖该状态判断」（可在枚举旁用问/答注释说明动机 → [comments.md#设计决策注释](comments.md#设计决策注释)） |
| **入参** | VO/DTO 上枚举字段加项目枚举校验注解，与合法值同源，避免两处维护 |
| **比较** | 包装类型用 `Objects.equals` / 工具类，禁裸 `status.equals(...)` |

```java
// 占位：校验注解名跟项目
public enum OrderStatusEnum {
    UNPAID(0, "待支付"),
    PAID(10, "已支付"),
    COMPLETED(30, "已完成"),
    CANCELED(40, "已取消");
    // getter / 合法值数组跟邻枚举写法

    // ========== 问：为什么提供 isXxx / haveXxx？ ==========
    // ========== 答：判断集中在此，便于查引用，避免业务散落魔法数 ==========

    /** 是否正处于待支付 */
    public static boolean isUnpaid(Integer status) {
        return Objects.equals(status, UNPAID.getStatus());
    }

    /** 是否已支付过（含后续态，便于 IDE 查引用） */
    public static boolean havePaid(Integer status) {
        return Objects.equals(status, PAID.getStatus())
                || Objects.equals(status, COMPLETED.getStatus());
    }
}

// VO 字段
@InEnum(OrderStatusEnum.class) // 无此注解则用项目等价校验
private Integer status;
```

<a id="多供应商模板方法"></a>

## 多供应商模板方法（风格建议）

```java
// 占位：接口/异常类型跟项目
public abstract class AbstractXxxClient {
    public final XxxResp unifiedCall(XxxReq req) {
        validate(req);
        try {
            return doCall(req);
        } catch (BizException ex) {
            throw ex;
        } catch (Throwable ex) {
            log.error("[unifiedCall][client={}] request failed", getId(), ex);
            throw translateToBizException(ex);
        }
    }
    protected abstract XxxResp doCall(XxxReq req) throws Throwable;
}
```

| 要点 | 说明 |
|------|------|
| 外层 | 校验、日志、异常翻译；对外方法尽量 `final` |
| 子类 | 只碰具体 SDK |
| 工厂 | 按渠道 id/配置创建并缓存客户端 |
| 禁止 | 业务 Service 直接调用多家 SDK 静态方法 |

分层上下文 → [architecture.md#多供应商适配](architecture.md#多供应商适配)。

<a id="耗时日志-stopwatch"></a>

## 耗时日志（StopWatch）

> **强约束**：打印耗时/cost 必须 `org.springframework.util.StopWatch`；禁 `currentTimeMillis` 手算。项目有统一格式化工具 → **quickref 优先**。

```java
StopWatch watch = new StopWatch("task-id");
watch.start("step");
try {
    // … 业务 …
} finally {
    if (watch.isRunning()) watch.stop();
    log.info("done, cost={}s", String.format("%.3f", watch.getTotalTimeSeconds()));
}
```

- 多阶段：同一 watch 多次 `start`/`stop`；阶段用 `getLastTaskTimeMillis()`，汇总用 `getTotalTimeSeconds()`  
- **禁止**：手算 `(end-start)/1000`；只打毫秒整数  

## Service / 注解（摘要）

| 项 | 做法 |
|----|------|
| Service | `validateXxx`；复杂 VO → XML；写操作 `@Transactional(rollbackFor=Exception.class)` |
| 异常 | 业务异常跟邻；log 末参带 `Throwable`；禁吞异常 |
| 注解排序 | 同一声明处从短到长（风格建议） |
| 长行换行 | 固定缩进续行，约 120 字符内可读则不换 |

方法结构 → [comments.md](comments.md)、[method-structure.md](method-structure.md)。
