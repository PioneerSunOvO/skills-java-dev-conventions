# 注释润色对照示例

配合 [comments.md](comments.md) 与 [comments-examples.md](comments-examples.md) 使用。这里专放“前后改写”示例，方便在审代码时快速比对。

## 润色 1：空洞类注释

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
 * @author: 张三
 * @since: 2026-06-09 10:00
 */
```

## 润色 2：机翻方法注释

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

## 润色 3：复述代码的行注释

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

## 润色 4：配置字段用英文字段名

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

## 润色 5：Controller 不必 Javadoc 与 Swagger 双写

**前（冗余）：**

```java
/**
 * 按关键词检索订单补全建议。
 * @param param 入参
 * @return 建议列表
 */
@ApiOperation(value = "订单补全", notes = "按关键词返回建议列表")
@GetMapping("/suggest")
public R<List<BizVo>> suggest(@Valid SuggestParam param) {
    return R.ok(bizService.suggest(param));
}
```

**后（二选一，Swagger 已够用时保留 Swagger）：**

```java
@ApiOperation(value = "订单补全", notes = "按关键词返回建议列表；无匹配时 data 为空列表")
@GetMapping("/suggest")
public R<List<BizVo>> suggest(@Valid SuggestParam param) {
    return R.ok(bizService.suggest(param));
}
```

## 润色 6：复杂业务编排注释

**类注释：**

```java
// 前：该类用于处理统计报表相关的业务逻辑。
// 后：@description: 统计分类报表服务；汇总口径与统计 SQL 一致。
//     查询与导出固定同一层级；数量与业务键的换算均在业务层处理。
```

**方法 Javadoc：**

```java
// 前：该方法用于组装报表节点数据。
// 后：组装报表节点（固定仅当前层级）。
//     多来源数量在业务层按分组键合并；内部 ID 与业务键的换算也在业务层处理。
```

**步骤注释由简到详：**

```java
// 前：// 查数据
// 后：// 加载本级分类的年份统计数据
// 更后（有分支时）：// 确定查询范围：顶层走 ID IN，子层走父级路径前缀匹配
```

## 润色 7：与业务无关的技术动作

**前：**

```java
// 查询数据库获取明细
// 遍历列表处理数据
// 封装返回结果
```

**后：**

```java
// 一次查出明细与期数，避免循环补查
// 只处理待结算记录，已作废数据不参与本次汇总
// 返回分页结果，口径与列表接口一致
```

## 润色 8：占位词改写

**前：**

```java
// 下面多路统计互不依赖，一并并行查询
// 构建基数映射：同一分组键多行共用
// 构建汇总映射：按分组键汇总，供合并列展示
// 补齐聚合行展示字段
// 父级承接合并单元格汇总值
// uuid写入Redis，保存时原样带回
```

**后：**

```java
// 多路统计互不影响，并行查
// 基数：同一分组键多行共用
// 汇总金额：按分组键汇总，合并列展示
// 写入聚合行展示字段
// 父级汇总合并列金额
// uuid写入Redis，保存时凭uuid读取
```
