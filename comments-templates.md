# 注释模板与标杆对照

与 [comments.md](comments.md) 配套。**规则以 comments.md 为准**；本文只放按需复制的形状示例，小改不必每次打开。

## 方法 Javadoc 标杆

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
 * parentId=0 使用 categoryId IN 本级列表；parentId≠0 使用父级 path 作前缀匹配。
 *
 * @param parentId         请求参数中的父级 ID
 * @param levelCatalogList 本级分类列表
 * @param parentCatalog    parentId≠0 时的父级，parentId=0 传 null
 * @return 统计 SQL 用的区域条件
 */
private QueryScope resolveQueryScope(...) { ... }
```

更多编排范本 → [examples-method-structure.md](examples-method-structure.md)；润色前后对照 → [comments-rewrites.md](comments-rewrites.md)。

## `@ExceptionHandler`

```java
/**
 * 处理业务异常，保留自定义 message 与返回码。
 *
 * @param e 业务异常
 * @return 失败响应，data 为 null
 */
@ExceptionHandler(BusinessException.class)
public R<Void> handleBusinessException(BusinessException e) { ... }
```

## `@Bean` / 配置类

```java
/**
 * 创建搜索客户端，基于 Jackson 序列化。
 */
@Bean
public SearchClient searchClient(RestClient restClient) { ... }
```

## MapStruct

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

生成类 `*Impl` **不**手写注释。规则见 [comments.md](comments.md) 分级表。

## 静态工具类 / 接口 default

- 静态工具类：类注释说明无状态、不可实例化；每个 **public** 方法按公开业务方法标准写 Javadoc
- 接口 `default`：与抽象方法同标准；可一句说明与重载的区别（如「默认不覆盖已有文档」）

## 字段注释位置

`/** ... */` 写在字段**上方**，位于 `@TableField`、`@ExcelProperty`、校验注解等**之前**：

```java
/** 单位名称 */
@ExcelProperty("单位名称（必填）")
private String comName;
```

```java
/** 订单金额，单位：元，保留两位小数 */
/** 外部流水号，与第三方支付流水一致 */
/** 列表末尾合计行名称；parentId≠0 时展示为「父级名称+合计」 */
```
