# 注释反面教材、形状模板与润色对照

与 [comments.md](comments.md) 配套。**权威规则**在 comments.md；本文按需阅读，小改不必打开。

| 节 | 何时读 |
|----|--------|
| 占位词 / 套话 / 见到就删 | 审注释、去 AI 味 |
| 形状模板 | 需要可复制的 Javadoc / 字段位置示例 |
| 润色前后对照 | 改写空洞/机翻注释时对照 |

<a id="占位词禁用法"></a>

## 占位词禁用法

见到下列写法，**删或改**为具体业务动作：

| 禁用 | 问题 | 改法示例 |
|------|------|----------|
| 下面… / 上述… / 以下… | 指代空泛 | 直接写对象：`// 五类统计并行查` |
| 补齐 / 回填 / 承接 | 不说清动作 | `写入`、`关联`、`汇总`、`外部编码转内部主键` |
| 构建…索引（指 Map） | 伪技术词 | `// 名称、单价映射` |
| 维度转换 / 组装视图对象 | 抽象隐喻 | `外部编码转内部主键`、`写入明细字段` |
| 供…使用 / 用于…展示 | 空泛 | 写规则：`合并列展示`、`分组小计` |
| 逐组 / 逐行 / 首先…其次…最后… | 机械 / AI 三段式 | `按组`、`按行`，或改步骤块 |
| 拉取后合并计算 / 一并并行查询 | 收尾空泛 | 每路写口径，合并步单独一句 |
| 遍历列表 / 查询数据库 / 设置字段 / 封装返回值 | 复述技术动作 | 写过滤口径、状态前置、保留哪些语义 |
| 提高可读性 / 提高性能 / 方便后续扩展 | 空泛结论 | 写批量查、防重复、固定排序等机制 |

<a id="推荐用词"></a>

## 推荐用词

| 场景 | 推荐 | 避免 |
|------|------|------|
| 内存 Map | 映射、按 X 汇总 | 构建索引 |
| 补全展示字段 | 写入、关联 | 补齐 |
| 父级汇总 | 汇总、合并列 | 承接 |
| 落库关联 ID | 写入主表 ID | 回填 |
| 并行多 SQL | 互不影响，并行查 + 每路冒号说明 | 下面 N 类…一并拉取 |
| 预览缓存 | 写入 Redis，凭 uuid 读取 | 缓存到 Redis，按 uuid 取回 |

## 应避免的模式

| 模式 | 改法 |
|------|------|
| 已触发分块却无步骤块 / 块间无空行 | 补步骤注释与空行 |
| 步骤注释只有名词；Javadoc 与首步骤块重复 | 步骤写阶段，Javadoc 写口径 |
| 机翻被动、复述代码、与业务无关的技术动作 | 改主动短句或删除 |
| `----` 装饰线；Swagger 与 Javadoc 双写同义 | 删线；Controller 二选一 |
| 字段注释复述英文字段名；`@param param 查询参数` | 写业务语义与字段约定 |

<a id="ai-套话--见到就删或改"></a>

## AI 套话 — 见到就删或改

| 删掉或改写 | 替换思路 |
|------------|----------|
| 该方法用于… / 本方法的主要作用是 | 直接写职责或规则 |
| 值得注意的是 / 综上所述 / 旨在 / 有效地 | 删除或只留动词 |
| 确保…的正确性 / 对数据进行处理 | 写具体规则或删除 |
| 这里调用 xxx 方法完成... | 写该调用带来的业务结果 |

<a id="与业务无关的描述--见到就删"></a>

## 与业务无关的描述 — 见到就删

| 差 | 好 |
|----|----|
| `// 遍历订单列表` | `// 只处理待结算订单，已作废记录跳过` |
| `// 查询数据库获取明细` | `// 一次查出明细与期数，避免循环补查` |
| `// 将 DTO 转成 Entity` / `// 封装返回结果` | `// 仅保留可落库字段` / `// 返回分页，口径与列表一致` |
| `// 设置默认值` / `// 调用支付接口` | `// 缺失税率按 0` / `// 仅处理中订单才补单查询` |
| `提高性能` / `便于后续维护` | 写批量查、防并发、固定字段口径 |

## 步骤注释：好 vs 差

| 好 | 差 |
|----|-----|
| `// 确定区域条件：parentId=0 用 categoryId IN，parentId≠0 用父级 path 前缀匹配` | `// 解析` |
| `// 无记录或无明细时清空衍生数据` | `// 判断订单` |
| `// 先删后插，保证与源数据一致` | `// 保存数据` |
| `// 用待支付状态作更新条件，防并发重复` | `// 更新状态` |
| `// 已终态直接返回，避免重复记账` | `// 遍历通知记录` |

## 正反例（摘要）

**好**：按分级表补全；Javadoc=口径、步骤块=阶段；块间空行；边界写进步骤注释。  
**差**：每个 getter 多行 Javadoc；长方法无分块；`// 查询未删除的` / `首先…其次…` / `构建索引供展示`；字段名复述。

体内无分块 → [examples-method-structure.md](examples-method-structure.md)、[method-structure.md](method-structure.md)。

---

<a id="形状模板"></a>
<a id="注释模板"></a>

## 形状模板（按需复制）

### 方法 Javadoc 标杆

```java
/**
 * 查询层级报表。
 * 固定返回指定父级下一级节点，末尾追加合计行；无权限或无数时对应字段为 0。
 *
 * @param param 含 parentId、year、筛选类型列表
 * @return 本级节点列表（含 reportData、hasNext）
 */
public List<ReportTreeVo> queryReport(ReportQueryParam param) { ... }

/**
 * 解析区域过滤条件，口径与列表查询接口一致。
 * parentId=0 使用 categoryId IN；parentId≠0 使用父级 path 前缀匹配。
 *
 * @param parentId      请求中的父级 ID
 * @param parentCatalog parentId≠0 时的父级，否则 null
 * @return 统计 SQL 用的区域条件
 */
private QueryScope resolveQueryScope(...) { ... }
```

更多编排范本 → [examples-method-structure.md](examples-method-structure.md)。

### `@ExceptionHandler` / `@Bean`

```java
/**
 * 处理业务异常，保留自定义 message 与返回码。
 *
 * @param e 业务异常
 * @return 失败响应，data 为 null
 */
@ExceptionHandler(BusinessException.class)
public R<Void> handleBusinessException(BusinessException e) { ... }

/** 创建搜索客户端，基于 Jackson 序列化。 */
@Bean
public SearchClient searchClient(RestClient restClient) { ... }
```

### MapStruct

```java
/**
 * 将源表实体转为检索文档；检索拼接字段在 @AfterMapping 中填充。
 *
 * @param source 源表记录
 * @return 检索文档
 */
SuggestDocument toDocument(SourceEntity source);

/** 映射完成后拼装检索字段并写入拼音检索字段 */
@AfterMapping
default void afterToDocument(...) { ... }
```

生成类 `*Impl` **不**手写注释。

### 字段注释位置

`/** ... */` 在字段**上方**，位于 `@TableField` / `@ExcelProperty` / 校验注解**之前**：

```java
/** 单位名称 */
@ExcelProperty("单位名称（必填）")
private String comName;

/** 订单金额，单位：元，保留两位小数 */
/** 外部流水号，与第三方支付流水一致 */
```

静态工具类：类注释说明无状态；每个 **public** 方法按公开业务方法写 Javadoc。接口 `default` 与抽象方法同标准。

---

<a id="润色对照"></a>
<a id="注释润色"></a>

## 润色前后对照

### 1. 空洞类注释

```java
// 前：该类用于处理订单相关的业务逻辑。主要功能包括：查询、更新、删除。
// 后：
/**
 * @description: 订单衍生数据服务实现；汇总口径与统计 SQL 一致
 * @author: 张三
 * @since: 2026-06-09 10:00
 */
```

### 2. 机翻方法注释

```java
// 前：值得注意的是，本方法主要用于处理支付回调。此外会校验流水，确保数据一致性。
// 后：支付回调处理；同一 outTradeNo 重复通知时幂等返回，不重复更新订单状态。
```

### 3. 复述代码 → 业务口径

```java
// 前：// 查询未删除的
// 后：// 列表只展示未删除记录，与前台「有效数据」口径一致
```

### 4. 字段名复述 → 业务语义

```java
// 前：/** companyName 短语前缀匹配权重 */
// 后：/** 公司名称短语前缀匹配权重，数值越大排序越靠前 */
```

### 5. Controller：Swagger 与 Javadoc 二选一

```java
// 前：多行 Javadoc + @ApiOperation 同义双写
// 后：仅保留 @ApiOperation（或仅 Javadoc），notes 写清空结果语义
@ApiOperation(value = "订单补全", notes = "按关键词返回建议列表；无匹配时 data 为空列表")
@GetMapping("/suggest")
public R<List<BizVo>> suggest(@Valid SuggestParam param) {
    return R.ok(bizService.suggest(param));
}
```

### 6. 编排类注释

```java
// 前：该类用于处理统计报表相关的业务逻辑。
// 后：@description: 统计分类报表服务；汇总口径与统计 SQL 一致。
//     查询与导出固定同一层级；数量与业务键换算均在业务层处理。
```
