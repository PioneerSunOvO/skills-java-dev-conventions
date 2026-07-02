# 自然中文注释

与 [SKILL.md](SKILL.md) 配套。**注释权威文件**：五层结构、分级表、字段规则、特殊类型模板均以此为准。项目 `CLAUDE.md` / rules 中的类注释模板优先于本文件。

## 30 秒自检（审查口令）

```
1. 新类有三要素吗？@description 是职责还是 CRUD 列表？
2. 公开方法：第一句能说清「做什么 + 何时不做」吗？
3. 方法 ≥15 行或有业务分支：有步骤块和块间空行吗？
4. 字段注释是业务语义，还是 Java 字段名复述？
5. 改逻辑的地方，注释同步改了吗？
6. 读起来像同事交代规则，还是机翻 / AI 套话？
```

## 注释分级表（必须 / 建议 / 可省略）

| 对象 | **必须** | **建议** | **可省略** |
|------|----------|----------|------------|
| **类** | `@description` + `@author` + `@since`（或项目模板） | 事务、幂等、切面顺序等边界 | — |
| **公开业务方法**（Service、Client、工具类 public API） | 多行 Javadoc：职责 + 边界；有参写 `@param`；有返回值写 `@return` | 步骤分块（见下文触发条件） | — |
| **Controller 接口** | Javadoc **或** 等价的 Swagger / OpenAPI 注解（二选一，见下节） | 有校验分支、权限、空结果语义时再补 Javadoc | 一行委托不写步骤块 |
| **`@Override` 实现** | 实现有**超出接口**的边界时必须补 | 接口 Javadoc 已完整时可不重复 | 纯转发且无语义增量 |
| **`@ExceptionHandler` / `@Bean`** | 多行 Javadoc：处理什么、返回什么 | — | — |
| **MapStruct 接口方法** | `toXxx`、含业务拼装的 `default` / `@AfterMapping` | 纯字段映射可依赖类注释 | 生成类 `*Impl` 不手写 |
| **私有方法** | — | 口径不直观、≥15 行、抽出的 `validate*` / `build*`；承载业务口径/分支规则/数据转换时写 Javadoc | `trim`、`append` 等机械辅助 |
| **字段** | 见下文「字段注释范围」 | Entity / DTO 与邻类、同域 VO 对齐 | 含义已由类注释完全覆盖的简单 VO |
| **枚举常量** | 业务枚举每个常量 | 技术枚举（如 ES 字段类型）写用途 | 纯占位枚举 |

**与 pitfalls 一致**：不要给每个方法都加千篇一律的 Javadoc；按上表分级，缺「必须」项才不合格。

## 硬性原则

- **新增/修改业务代码**：按分级表补注释
- **可省略**：getter、简单 setter、一眼能看懂的赋值、≤3 行无分支委托
- **改逻辑必改注释**；注释与实现不一致视为缺陷
- **朗读自检**：像同事口头交代规则 → 合格；像翻译腔 → 重写

## 注释五层结构

| 层级 | 形式 | 写什么 |
|------|------|--------|
| **类** | `@description` + `@author` + `@since` | 职责 + 关键边界（事务、幂等、口径、跨层处理位置） |
| **方法** | 标准多行 `/** ... */` | 做什么、口径/约束；`@param` / `@return` 写业务含义 |
| **步骤块** | `// 动词+对象` | 本步目的；有分支口径时用冒号展开；**块间空一行** |
| **块内行** | `//` 单行 | 口径、单位、分支原因、并发/幂等（仅复杂处补） |
| **字段** | `/** ... */` | 业务含义、单位、条件展示规则 |

**禁止**：

```java
// 1. 获取订单
// 2. 判断状态
// 查询数据库
// 遍历列表
```

## 方法 Javadoc

### 必须包含

- 第一句：动词短语 + 可选（范围/约束），不只重复方法名
- 第二句（如有）：口径、归并规则、跨接口一致性、幂等/跳过条件
- `@param`：参数的业务含义与取值约定，不必复述类型
- `@return`：结构、口径或转换结果；空 / 0 代表什么；可用 `→`（`void` 不写 `@return`，副作用写在正文）

### 写法骨架

| 行 | 写什么 | 示例 |
|----|--------|------|
| 第一句 | 动词短语 + 可选（范围/约束） | `组装报表节点（固定仅 pid 下一级）。` |
| 第二句（如有） | 口径、归并规则、跨接口一致性、处理层级 | `自办、邮局数量在业务层按 catId+ISBN 合并；itmId 与 ISBN 的对应关系也在业务层处理。` |
| `@param` | 业务含义与取值约定 | `含 pid、year、moTypes` / `pid≠0 时的父级分类，pid=0 传 null` |
| `@return` | 结构、口径或转换结果 | `带 reportData 的分类节点及末尾合计行` / `catId#isbn → 数量` |

### 标杆对照

```java
/**
 * 查询统计分类报表。
 * 固定返回 pid 下一级分类节点，末尾追加合计行；无权限或无数时对应字段为 0。
 *
 * @param param 含 pid、year、moTypes
 * @return 本级分类节点列表（含 reportData、hasNext）
 */
public List<StatCatalogTreeVo> viewReport(MagOrderViewReportParam param) { ... }

/**
 * 组装报表节点（固定仅 pid 下一级）。
 * 自办、邮局数量在业务层按 catId+ISBN 合并；itmId 与 ISBN 的对应关系也在业务层处理。
 *
 * @param param 查询参数
 * @return 带 reportData 的分类节点及末尾合计行
 */
private List<StatCatalogTreeVo> buildViewReportNodes(MagOrderViewReportParam param) { ... }

/**
 * 解析自办订单区域过滤条件，口径与 getPageList 一致。
 * pid=0 使用 catId IN 本级列表；pid≠0 使用父级 fullid 作 catidArray 前缀。
 *
 * @param parentCatId      请求参数 pid
 * @param levelCatalogList 本级分类列表
 * @param parentCatalog    pid≠0 时的父级分类，pid=0 传 null
 * @return 自办统计 SQL 用的区域条件
 */
private OrderQueryScope resolveOrderQueryScope(...) { ... }

/**
 * 业务成功后解析上下文，按配置执行同步或清理。
 *
 * @param joinPoint 切点
 * @param marker    业务标记注解
 */
public void afterSuccess(JoinPoint joinPoint, BusinessMarker marker) { ... }

/**
 * 重算并落库衍生数据。
 * 主记录不存在或明细为空时清空；有数据则先删后插。
 *
 * @param id 业务主键
 */
public void rebuildById(Long id) { ... }
```

### `@Override` 实现类

- 接口方法 Javadoc 已含职责、边界、`@param`、`@return` → 实现类**可不重复**
- 实现类有额外边界（如增量不同步策略、特殊早退）→ **必须**在实现方法上补充

### 单行 `/** ... */` 的适用范围

| 场景 | 是否允许单行 |
|------|----------------|
| 公开业务方法 | **不允许**，用多行 + `@param`/`@return` |
| 私有辅助方法（口径一句话能说清） | **允许** |
| 字段注释 | **允许**（推荐一行业务说明） |
| 枚举常量 | **允许** |

## Controller 与 Swagger / OpenAPI

- **已用 `@ApiOperation` / `@Operation` 写清职责、参数、返回值** → Javadoc 可省略，审查以 API 文档注解为准
- **Swagger 仅一句话、未写边界** → 仍须 Javadoc 补充（空结果、权限、幂等等）
- **禁止** Javadoc 与 `@ApiOperation.notes` / `@Operation.description` **矛盾**
- 一行 `return ApiResult.ok(service.xxx(param))` → **不需要**步骤分块

## 步骤分块（触发条件与豁免）

**满足任一即必须步骤分块：**

1. 方法体（含空行与注释）**≥ 15 行**
2. 含 **2 个以上**独立业务阶段（校验 → 查询 → 写入 → 汇总）
3. 含非显而易见的分支（幂等、跳过、早退、批量容错）

**不必分块：**

- ≤ 3 行且无语义分支的委托 / 转发
- 纯 getter / setter / `@Bean` 注册（逻辑一眼可读时）
- Controller 一行调 Service

**分块写法：**

1. 每步块首一行 `//` 总注释
2. **块与块之间空一行**
3. 默认 **动词 + 对象**；本步含分支口径时用冒号展开
4. 不用 `----` 装饰线
5. 块内机械循环/判空不必逐步注释；口径、幂等、并发处有信息增量时再补单行 `//`

### 步骤注释两级

| 级别 | 何时用 | 格式 | 示例 |
|------|--------|------|------|
| **简** | 步骤意图一目了然 | `// 动词+对象` | `// 加载本级分类的年份统计数据` |
| **详** | 本步含分支口径或跨层约定 | `// 动词+对象：条件A 用…，条件B 用…` | `// 确定自办订单区域条件：pid=0 用 catId IN，pid≠0 用父级 fullid 匹配 catidArray` |

```java
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
        ...
    }
}
```

编排型方法（如 `build*`、`export*`）示例：

```java
private List<StatCatalogTreeVo> buildViewReportNodes(MagOrderViewReportParam param) {
    // 确定本次统计涉及的分类范围
    List<StatCatalog> levelCatalogList = resolveTargetCatalogs(param);
    if (CollectionUtils.isEmpty(levelCatalogList)) {
        return Collections.emptyList();
    }

    // 提取本级分类 ID
    List<Long> levelCatIds = extractCatIds(levelCatalogList);
    // 确定自办订单区域条件：pid=0 用 catId IN，pid≠0 用父级 fullid 匹配 catidArray
    OrderQueryScope orderScope = resolveOrderQueryScope(parentCatId, levelCatalogList, parentCatalog);

    // 加载本级分类的年份统计数据
    ViewReportYearStats yearStats = loadYearStats(...);
    // 汇总本级分类的年份统计数据
    Map<Long, MagOrderViewReportDataVo> reportByCatId = aggregateReportByCat(levelCatIds, yearStats);

    // 组装报表节点
    List<StatCatalogTreeVo> nodes = buildStatCatalogNodes(...);
    // 添加合计行
    nodes.add(buildTotalRow(...));

    return nodes;
}
```

### 步骤注释：好 vs 差

| 好 | 差 |
|----|-----|
| `// 确定自办订单区域条件：pid=0 用 catId IN，pid≠0 用父级 fullid 匹配 catidArray` | `// 解析 condition` |
| `// 解析条件表达式，不通过则不执行` | `// 解析` |
| `// 无记录或无明细时清空衍生数据` | `// 判断订单` |
| `// 先删后插，保证与源数据一致` | `// 保存数据` |
| `// 用待支付状态作更新条件，防并发重复` | `// 更新状态` |

块内行注释示例：

```java
// 按分组键归并，数量取组内最小值（与统计口径一致）
// 期数至少为 1，避免除零
// 单条失败不挡整批，方便断点续跑
```

## 类注释（硬性）

```java
/**
 * @description: 统计分类报表服务；自办读征订汇总、邮局读发行计划，区域过滤与订单列表 getPageList 一致。
 *              查询与导出均固定 pid 下一级；数量在业务层按 catId+ISBN 归并，itmId 与 ISBN 在业务层换算。
 * @author: Codex
 * @since: 2026-06-30 18:30
 */
```

复杂类可在 `@description` 或首行用分号衔接多句边界，例如「切面 Order 设在事务拦截器内侧」。

`@description` 为空视为不合格。禁止写「查询、更新、删除」功能列表。

## 字段注释

### 写法

```java
/** 金额，单位：分 */
/** 外部流水号，与第三方支付 out_trade_no 一致 */
```

### 常量与静态字段

业务常量、枚举旁写**含义 + 条件展示**，一行即可：

```java
/** 列表末尾合计行名称；pid≠0 时展示为「父级名称+合计」 */
private static final String TOTAL_ROW_NAME = "合计";
```

### 必须写字段注释的类型

- `@ConfigurationProperties` 及嵌套配置类
- ES Document、索引 mapping 对应字段
- 对外 API 的 **Param / VO** 出参入参
- 业务含义不直观的枚举常量

### 禁止

- **用 Java 字段名代替业务说明**

```java
// 差
/** companyName 短语前缀匹配权重 */
/** searchText 拼接字段 */

// 好
/** 公司名称短语前缀匹配权重，数值越大排序越靠前 */
/** 拼接检索字段匹配权重，公司名称未命中时作为备选路径 */
```

### 注释位置

`/** ... */` 写在字段**上方**，位于 `@TableField`、`@ExcelProperty`、校验注解等**之前**：

```java
/** 单位名称 */
@ExcelProperty("单位名称（必填）")
private String comName;
```

### 同域字段对齐

同一业务字段在 Document、VO、DTO 间注释口径应一致（如 `highlightName`：展示用高亮名称，含 `<em>` 标签）。

## 特殊类型模板

### `@ExceptionHandler`

```java
/**
 * 处理业务异常，保留自定义 message 与返回码。
 *
 * @param e 业务异常
 * @return 失败响应，data 为 null
 */
@ExceptionHandler(BusinessException.class)
public ApiResult<Void> handleBusinessException(BusinessException e) { ... }
```

### `@Bean` / 配置类

```java
/**
 * 创建 ES 高级客户端，基于 Jackson 序列化。
 */
@Bean
public ElasticsearchClient elasticsearchClient(RestClient restClient) { ... }
```

### MapStruct

```java
/**
 * 将源表实体转为 ES 文档；searchText 与拼音字段在 @AfterMapping 中填充。
 *
 * @param invoice 源表记录
 * @return ES 文档
 */
InvoiceCompanySuggestDocument toDocument(CommonInvoice invoice);

/** 映射完成后拼装 searchText 并构建拼音索引字段 */
@AfterMapping
default void afterToDocument(...) { ... }
```

生成类 `*MapperImpl` **不**手写注释。

### 静态工具类

类注释说明无状态、不可实例化；每个 **public** 方法按公开业务方法标准写 Javadoc。

### 接口 `default` 方法

与抽象方法同标准；可在一句话中说明与重载方法的区别（如「默认不覆盖已有文档」）。

## 语气与排版

- 短句、直说；术语不硬翻（DTO、SpEL、幂等保留英文）
- 多句 Javadoc 用分号衔接相关事实，不必强行拆段
- 范围约束放括号：`（固定仅 pid 下一级）`、`（含 reportData、hasNext）`
- 数据转换可用箭头：`catId+ISBN → 数量`、`itmId → ISBN`
- 中英空格：`按 orderId 幂等更新`、`pid≠0 时`
- 全角标点用于中文句子
- 去 AI 味、去机翻味（见下表）

## 应避免的模式

| 模式 | 改法 |
|------|------|
| 只有 Javadoc、体内无步骤分块（且已触发分块条件） | 补步骤注释与空行 |
| 步骤注释只有名词 | 补动词与边界 |
| 机翻被动语态 | 改主动短句 |
| 复述可见代码 | 写业务口径或删除 |
| AI 套话 | 删除 |
| 用 `----` 装饰线 | 删除，靠空行分层 |
| Controller Javadoc 与 Swagger 双写同义内容 | 二选一 |
| 字段注释复述英文字段名 | 改业务中文语义 |

## AI 套话 — 见到就删或改

| 删掉或改写 | 替换思路 |
|------------|----------|
| 该方法用于… / 本方法的主要作用是 | 直接写职责或规则 |
| 值得注意的是 / 综上所述 | 删除 |
| 旨在 / 有效地 / 进行…操作 | 删修饰，保留动词 |
| 确保…的正确性 | 写具体规则或删除 |

## 润色示例

### 体内无分块 → 步骤分块

**前：**

```java
public void rebuildById(Long id) {
    if (Objects.isNull(id)) return;
    Order order = orderService.getById(id);
    if (Objects.isNull(order)) { deleteById(id); return; }
    ...
}
```

**后：** 见 [method-structure.md](method-structure.md) Service 方法模板。

### 配置字段注释

**前：** `/** companyName 短语前缀匹配权重 */`

**后：** `/** 公司名称短语前缀匹配权重，数值越大排序越靠前 */`

### 支付回调幂等

**前：**

```java
/**
 * 值得注意的是，本方法主要用于处理支付回调。
 */
```

**后：**

```java
/**
 * 支付回调处理；同一 outTradeNo 重复通知时幂等返回，不重复更新订单状态。
 */
```

### 步骤注释由简到详

```java
// 前：// 查数据
// 后：// 加载本级分类的年份统计数据
// 更后（有分支时）：// 确定自办订单区域条件：pid=0 用 catId IN，pid≠0 用父级 fullid 匹配 catidArray
```

## 正反例

### 好

- 按分级表补全，不机械全覆盖
- 步骤分块 + 块间空行 + 边界写进步骤注释
- 块内补充口径：`// 单条失败不挡整批，方便断点续跑`
- VO 与 Document 同字段注释口径一致

### 差

- 每个 getter 都加 Javadoc
- 只有 Javadoc，长方法体内无分块、无空行
- `// 查询未删除的`、`// 遍历列表`（复述代码）
- `// 首先…其次…最后…`（AI 三段式）
- `/** searchText 拼接字段 */`（字段名复述）

## 与代码同步

改逻辑必改注释。方法结构（何时抽私有方法）见 [method-structure.md](method-structure.md)；陷阱表见 [pitfalls.md](pitfalls.md)；示例见 [examples.md](examples.md)。
