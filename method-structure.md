# 方法结构与私有方法抽取

与 [SKILL.md](SKILL.md) 配套。**步骤分块写法、分级表、Javadoc 标准**以 [comments.md](comments.md) 为权威；本文只讲方法体组织与私有方法抽取。

## 步骤分块标杆（通用模板）

- 方法 Javadoc：见 [comments.md#方法-javadoc](comments.md)
- 体内：每步一行 `//` 总注释 → 本步代码 → **空一行** → 下一步
- 总注释：**动词 + 对象 + 边界/跳过条件**
- 块内复杂分支再补单行 `//`

**何时必须分块、何时豁免** → [comments.md#步骤分块触发条件与豁免](comments.md)

```java
/**
 * 业务成功后解析上下文，按配置执行同步或清理。
 *
 * @param joinPoint 切点
 * @param marker    业务标记注解
 */
@AfterReturning("@annotation(marker)")
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
        if (Objects.isNull(id)) {
            continue;
        }
        if (ActionType.DELETE == marker.action()) {
            syncService.deleteById(id);
        } else {
            syncService.rebuildById(id);
        }
    }
}
```

## 不必步骤分块的典型场景

```java
// Controller 一行委托
@GetMapping("/suggest")
public ApiResult<List<OrderVo>> suggest(@Valid OrderSuggestParam param) {
    return ApiResult.ok(orderSuggestService.suggest(param));
}

// @Bean 注册（逻辑一眼可读）
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
    return interceptor;
}
```

## 按块分层（方法体结构）

公开方法（尤其 ServiceImpl）典型分层：

```
准备/解析入参 → 校验与前置过滤（不通过早退）→ 查数或计算 → 落库/副作用 → 返回
```

| 要求 | 说明 |
|------|------|
| 每步一块 | 块首一行 `//` 总注释，说明本步目的与跳过条件 |
| 块间空行 | 每步结束后空一行再写下步 |
| 早退集中 | 不满足条件则 `return`/`continue`，注释写在块首 |
| 核心步骤 | 循环、分支、金额计算等块内再补行内 `//` |
| 不用 `----` | 靠空行 + 步骤注释分层，不用装饰线 |

**不合格**：已触发分块条件，却整段无空行、无步骤注释、只有方法 Javadoc。

## 公开方法：编排优先

| 指标 | 建议 |
|------|------|
| 目标行数 | 约 15–50 行（含空行与分块注释） |
| ≥ 15 行或有多个业务阶段 | 必须步骤分块（见 comments.md） |
| 超过 80 行且无分块 | 不合格 |

业务复杂时用步骤分块保持可读，不要机械拆成多个一行 wrapper。

## 何时抽私有方法（按需）

### 建议抽

- 同一段逻辑出现 **2 次及以上**
- 某一步超过 15 行，公开方法难一眼读完
- 需要单独 Javadoc 说明口径（见 comments.md 分级表）

### 不建议抽

- 只调用一次，抽完要跳转 3+ 次才懂全流程
- 2–3 行胶水代码
- `processXxx` / `handleXxx` 空壳

私有方法抽出后，公开方法仍保留步骤分块；私有方法体内同样按块分层。口径一句话能说清的私有方法可用单行 `/** ... */`。

## 私有方法命名前缀

| 前缀 | 用途 |
|------|------|
| `validate*` / `check*` | 校验 |
| `build*` | 组装行/VO/列表 |
| `group*` / `match*` | 分组、规则匹配 |
| `merge*` / `append*` | 合并、追加 |
| `find*` / `getMatch*` | 查询辅助 |
| `resolve*` | 解析 ID、表达式、配置 |
| `normalize*` | 输入标准化 |

## Service 方法模板

```java
/**
 * 重算并落库衍生数据。
 * 主记录不存在或明细为空时清空；有数据则先删后插。
 *
 * @param id 业务主键
 */
public void rebuildById(Long id) {
    // 主键为空直接返回
    if (Objects.isNull(id)) {
        return;
    }

    // 加载主记录与明细
    Order order = orderService.getById(id);
    List<OrderDetail> details = detailService.listByOrderId(id);

    // 无记录或无明细时清空衍生数据
    if (Objects.isNull(order) || CollectionUtils.isEmpty(details)) {
        deleteById(id);
        return;
    }

    // 按业务规则计算衍生行
    List<SummaryRow> rows = buildSummaryRows(order, details);

    // 先删后插，保证与明细一致
    deleteById(id);
    if (CollectionUtils.isNotEmpty(rows)) {
        this.saveBatch(rows);
    }
}
```

## 写操作入口

```java
/**
 * 新增记录。
 *
 * @param dto 入参
 * @return 保存成功返回 true
 */
@Override
@Transactional(rollbackFor = Exception.class)
public boolean save(OrderDto dto) {
    // 入参与业务边界校验
    validateOrder(dto, null);

    // 转换并补审计字段
    Order entity = OrderConvert.INSTANCE.dtoToEntity(dto);
    entity.setOpUserid(LoginUtil.getUserId()).setOpDate(new Date());

    return save(entity);
}
```

## 与注释的配合

| 结构元素 | 要求（详见 comments.md） |
|----------|--------------------------|
| 类 | `@description` 或首行 + 关键边界 |
| 公开方法 | 多行 Javadoc + `@param` / `@return` |
| 步骤块 | `// 动词+对象+边界`；块间空一行 |
| 块内 | 复杂分支、口径补单行 `//` |

## 常见陷阱

| 陷阱 | 对策 |
|------|------|
| 只有 Javadoc、体内无分块（已触发条件） | 补步骤注释与块间空行 |
| 步骤注释只有名词无边界 | 补「不通过则…」「无数据则…」 |
| 块间无空行 | 每步后空一行 |
| 给一行 Controller 委托也写步骤块 | 不必，见上文豁免场景 |
| 复杂逻辑无块内注释 | 在 if/循环/计算处补口径 |

更多见 [pitfalls.md](pitfalls.md)。
