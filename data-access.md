# 数据访问与 SQL

持久层与分页分支见 [versions.md](versions.md)。JPA 项目将 Mapper/XML 对应为 Repository / `@Query`。

## 分页

选型权威见 [versions.md#分页三分支](versions.md#分页三分支)。写法对照见 [examples.md#分页写法对照](examples.md#分页写法对照)。**禁止**全量查出后 `subList` 内存分页。

## 选型原则

| 场景 | 做法 |
|------|------|
| 单表简单条件 | MP `getById` / `list(wrapper)` 或 XML |
| 多表 JOIN、聚合、统计 | Mapper XML 自定义 SQL |
| 分页含关联名称 | XML 一次返 VO，避免 N+1 |
| 批量写 | `saveBatch` / XML `foreach` |

无 MP 时不用 `Wrappers` / `BaseMapper`。

## 禁止

- 循环内单条查库 → 批量或 XML 一次查
- Service 拼 SQL 字符串 → `#{}` 参数化
- 全量 list 后内存分页 → 见 [versions.md#分页三分支](versions.md#分页三分支)

## Mapper XML 写法

- `<sql id="Base_Column_List">` 复用列
- 动态条件 `<if>` / `<where>`
- 返 VO 用 `resultType` / `resultMap`，不在 Java 二次拼大 Map
- 改 XML 前考虑索引；`LIKE '%x'` 慎用

## Mapper XML 注释

每个 `<select>` / `<insert>` / `<update>` / `<delete>` 及独立 `<sql>`，须在标签**正上方**写 `<!-- -->`。

| 要素 | 说明 |
|------|------|
| 第一句 | 查/写什么、供哪类场景 |
| 第二句（如有） | 过滤口径、状态枚举、关联表 |
| 性能（如有） | 为何走汇总表、子查询先取 id |

复杂 `<if>` / 子查询 / `CASE` 在片段处补块内注释；写口径不写 SQL 关键字复述。

```xml
<!--
    按状态聚合订单金额。
    有效单：status in (1,2)；金额取自明细表避免主表冗余字段偏差。
-->
<select id="selectAmountAggregation" ...>
```

语气见 [comments.md](comments.md)。与 Service 分工：XML 写 SQL 口径，Service 写流程。

## MyBatis-Plus（仅 MP 项目）

### Lambda 固定条件

```java
Wrappers.lambdaQuery(BizEntity.class)
    .eq(BizEntity::getUserId, userId)
    .eq(BizEntity::getDeleted, notDeletedFlag)
    .orderByDesc(BizEntity::getId);
```

### 动态条件（推荐）

入参字段可能为空时，用 **条件参数** 形式（MP `wrapper.eq(condition, column, val)` 等），避免 `if` 拼一串 `wrapper`：

```java
Wrappers.lambdaQuery(Order.class)
    .eq(Objects.nonNull(param.getStatus()), Order::getStatus, param.getStatus())
    .in(CollectionUtils.isNotEmpty(param.getIds()), Order::getId, param.getIds())
    .like(StringUtils.isNotBlank(param.getKeyword()), Order::getName, param.getKeyword())
    .ge(Objects.nonNull(param.getStartTime()), Order::getCreateTime, param.getStartTime())
    .orderByDesc(Order::getId);
```

| 要点 | 说明 |
|------|------|
| 第一个参数 | `false` 时该条件**不拼入** SQL |
| 判空工具 | 跟邻文件 import（`Objects` / `CollectionUtils` / `StringUtils`），见 [coding-patterns.md#多工具库并存](coding-patterns.md#多工具库并存) |
| 禁止 | 在 `stream` / `forEach` 里按 id 循环 `selectById`（N+1） |
| XML 等价 | 复杂动态 SQL 仍优先 Mapper XML `<if>` / `<where>` |
| **勿自造** | MP 已支持条件参数时，**禁止**仅为消噪再封装一套 `xxxIfPresent`；项目**已有** QueryWrapper 扩展（`*IfPresent` 等方法）且邻代码在用 → **跟邻**，不新引入第三套 |
| **跟邻优先于教条** | 画像/quickref 写明「用项目 Wrapper 扩展」时按封装写；无封装时用 MP 原生 `eq(condition, column, val)` |

## 事务与一致性

- 多表写在同一 Service 事务内
- 分布式场景注释说明最终一致、幂等键（见下节）
- 有乐观锁字段时更新带 `version`

<a id="外部回调与幂等"></a>

## 外部回调与幂等

适用于支付渠道异步通知、第三方 Webhook、MQ 消费、定时补偿等——**与具体支付 SDK 无关**，各项目按邻代码对齐。

| 要点 | 做法 |
|------|------|
| **幂等键** | 业务单号 + 渠道流水号 / 消息 ID / 通知次数；库表唯一索引或状态机防重 |
| **状态前置** | 仅允许从「待支付 / 处理中」→「成功」；非法状态记录日志并返回渠道约定响应 |
| **事务边界** | 改单 + 写流水在同一 `@Transactional`；先落库再返回 SUCCESS 给外部 |
| **重复通知** | 已终态则直接返回成功，不重复记账 |
| **金额** | Service 内保持业务单位（元/分跟 Entity）；与渠道换算在适配层，见下节「金额与时间」 |
| **日志** | 打幂等键、业务单号；不打印完整卡号、密钥 |
| **异常** | 可重试错误与不可重试错误区分；跟邻模块 `catch` 与 MQ 重试策略 |

```java
// 回调处理骨架（示意；类名、响应体跟项目）
@Transactional(rollbackFor = Exception.class)
public void handlePaymentNotify(PaymentNotifyDto notify) {
    // 幂等：已处理过则直接返回
    if (notifyLogService.existsByNotifyId(notify.getNotifyId())) {
        return;
    }
    // 加载业务单并校验状态与金额
    Order order = orderService.getById(notify.getOrderId());
    validateNotifyAgainstOrder(order, notify);
    // 更新状态并记流水
    orderService.markPaid(order, notify);
    notifyLogService.saveProcessed(notify.getNotifyId());
}
```

**Spring Cloud**：跨服务入账时，以项目方案为准（本地事务 + 消息、Seata、Saga 等）；在类或方法注释写清**一致性边界**，不擅自引入新中间件。

### 幂等 / 并发最小模板

适用于支付通知、MQ 消费、库存扣减、状态流转等“重复执行会出错”的写路径。

| 模板 | 适用 | 要点 |
|------|------|------|
| **唯一键防重** | 有天然外部流水号 / 消息 ID | 表上唯一索引；重复插入捕获后按已处理返回 |
| **状态条件更新（CAS）** | 单据状态流转 | `update ... where id = ? and status = 期望旧状态`；**必须检查影响行数**：`0` → 并发抢先或状态不符，抛业务错误码或按幂等短路（注释写清选哪种） |
| **先查后改 + 终态短路** | 已终态可直接返回成功 | 先判断是否终态；**同一外部单号**重复回调 → 直接成功返回，不重复记账 |
| **业务单 + 渠道流水双校验** | 支付 / 对账 / 第三方通知 | 既校验本地业务单，也校验外部流水与金额，避免串单 |

```java
// CAS 示意：方法名、Wrapper、异常类型跟邻
int rows = orderMapper.updatePaidByIdAndStatus(id, StatusEnum.UNPAID.getStatus(), updateObj);
if (rows == 0) {
    throw new BizException(BizErrorCodes.ORDER_STATUS_NOT_UNPAID);
}
```

Mapper `default` 消噪封装与回调编排示例 → [framework-patterns.md#cas-更新封装](framework-patterns.md#cas-更新封装)。

**优先级**：先用项目已有模式；无先例时，优先“唯一键/条件更新”这类数据库层保护，而不是只靠内存锁。

<a id="幂等与分布式锁"></a>

## 幂等与分布式锁（强约束）

二者目的不同，**禁止**用「成功后立刻删幂等 Key」假装分布式锁。

| | 幂等（防重复提交 / 防重复消费） | 分布式锁（互斥临界区） |
|--|-------------------------------|------------------------|
| **目标** | 同一业务意图执行一次或可安全重放 | 同一时刻只有一个执行者进入 |
| **Key 生命周期** | 窗口期内保留；异常时可删 Key 以便重试（跟邻注解） | 持锁期间有效，结束释放 |
| **推荐** | 项目 `@Idempotent` / 唯一索引 / CAS | 项目 `@Lock4j` / Redisson / DB 锁跟邻 |
| **注释** | 写清幂等键、窗口、重复时行为 | 写清锁粒度、持锁做什么 |

**交付说明至少回答**：

1. 幂等键是什么？
2. 允许从哪些状态跳到哪些状态？
3. 重复调用时返回什么？
4. 并发失败 / 更新条数为 0 时怎么处理？
5. 若同时用了锁：锁与幂等各自保护什么？

<a id="金额与时间"></a>

## 金额与时间

- **类型**：金额用 `BigDecimal`，禁止 `float`/`double` 做金额运算
- **单位**：跟邻 Entity/VO 与注释（元 `decimal(18,2)`、整数分 `bigint` 等）；**注释必须标明单位**
- **运算**：`compareTo` 比较；除法指定 `scale` 与 `RoundingMode`；聚合 `reduce(BigDecimal.ZERO, BigDecimal::add)`
- **空值**：运算前 `Optional.ofNullable(x).orElse(BigDecimal.ZERO)` 或卫语句，跟邻代码
- 时间类型跟邻 Entity；同模块不混用 `Date` 与 `LocalDateTime`

### 金额规则补充

| 场景 | 规则 |
|------|------|
| 比较是否为 0 / 大小 | 用 `compareTo`，不要用 `equals` 判断数值相等 |
| 除法 / 比例 / 折扣 | 明确 `scale` 与 `RoundingMode`，不依赖默认精度 |
| 元 / 分换算 | 在适配层或转换层统一做；Service 内尽量只保留一种业务单位 |
| 汇总求和 | 从 `BigDecimal.ZERO` 起累加；空值先归零 |
| 展示格式化 | 放 VO 组装或展示层，不把格式化字符串写回 Entity |

```java
BigDecimal paidAmount = Optional.ofNullable(order.getPaidAmount()).orElse(BigDecimal.ZERO);
if (paidAmount.compareTo(expectedAmount) != 0) {
    throw new BizException("支付金额不一致");
}

BigDecimal ratio = amount
        .divide(totalAmount, 4, RoundingMode.HALF_UP);
```

---

## 附录：JPA 项目补充

仅当检测到 `spring-boot-starter-data-jpa` 时适用：

- 简单 CRUD：`JpaRepository` 派生方法或 `@Query`（参数绑定）
- 复杂报表：`@Query(nativeQuery=true)` 或项目混用的 MyBatis
- 懒加载不在 Controller 触发；展示数据 Service 一次查齐或 DTO 投影
