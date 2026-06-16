# 方法结构与私有方法抽取

与 [SKILL.md](SKILL.md) 配套。专注**方法体如何组织**；注释写什么见 [comments.md](comments.md)。

## 步骤分块标杆

- 方法 Javadoc：职责 + `@param` / `@return`（格式见 comments.md）
- 体内：每步一行 `//` 总注释 → 本步代码 → **空一行** → 下一步
- 总注释：**动词 + 对象 + 边界/跳过条件**
- 块内复杂分支再补单行 `//`

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

## 按块分层

```
准备/解析入参 → 校验与前置过滤（不通过早退）→ 查数或计算 → 落库/副作用 → 返回
```

| 要求 | 说明 |
|------|------|
| 每步一块 | 块首一行 `//` 总注释 |
| 块间空行 | 每步结束后空一行 |
| 早退集中 | `return`/`continue` 前注释写清跳过条件 |
| 不用 `----` | 靠空行 + 步骤注释分层 |

## 公开方法：编排优先

| 指标 | 建议 |
|------|------|
| 目标行数 | 约 15–50 行（含空行与注释） |
| 超过 50 行 | 必须步骤分块或抽 `validate*` / `build*` |
| 超过 80 行且无分块 | 不合格 |

## 何时抽私有方法

**建议抽**：逻辑重复 2 次以上；单步超过 15 行；需单独 Javadoc 的口径块。

**不建议抽**：只调用一次的 2–3 行胶水；无信息增量的 `processXxx` 空壳。

| 前缀 | 用途 |
|------|------|
| `validate*` / `check*` | 校验 |
| `build*` | 组装 VO/列表 |
| `resolve*` | 解析 ID、表达式、配置 |
| `group*` / `match*` | 分组、规则匹配 |

## Service 方法模板

```java
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
@Override
@Transactional(rollbackFor = Exception.class)
public boolean save(OrderDto dto) {
    // 入参与业务边界校验
    validateOrder(dto, null);

    // 转换并补审计字段（按项目：MetaObjectHandler 或手动赋值）
    Order entity = OrderConvert.INSTANCE.dtoToEntity(dto);
    fillAuditOnCreate(entity);

    return save(entity);
}
```

陷阱见 [pitfalls.md](pitfalls.md#六方法结构)。
