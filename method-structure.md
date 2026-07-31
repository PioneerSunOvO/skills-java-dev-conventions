# 方法结构与私有方法抽取

与 [SKILL.md](SKILL.md) 配套。**步骤分块写法、分级表、Javadoc 标准**以 [comments.md](comments.md) 为权威；本文只讲方法体组织与私有方法抽取。

## 步骤分块标杆

- 方法 Javadoc、步骤块写法、占位词表 → [comments.md](comments.md) + [comments-examples.md](comments-examples.md)
- 体内：每步一行 `//` 总注释 → 本步代码 → **空一行** → 下一步
- **何时必须分块、何时豁免** → [comments.md#步骤分块](comments.md#步骤分块)
- 完整范本（切面 / 编排方法 / 多路查询）→ [examples-method-structure.md](examples-method-structure.md)

## 不必步骤分块的典型场景

```java
// Controller 一行委托；R<T> = 项目统一响应包装，跟邻类
@GetMapping("/suggest")
public R<List<BizVo>> suggest(@Valid SuggestParam param) {
    return R.ok(bizService.suggest(param));
}

// @Bean 注册（框架无关示意；具体返回类型、构造器参数跟项目）
@Bean
public Object frameworkInterceptor() {
    Object interceptor = createFrameworkInterceptor();
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
| 每步一块 | 块首一行 `//` 总注释，说明本步业务目的、口径与跳过条件 |
| 块间空行 | 每步结束后空一行再写下步 |
| 早退集中 | 不满足条件则 `return`/`continue`，注释写在块首 |
| 核心步骤 | 循环、分支、金额计算等块内再补行内 `//`，只写业务信息，不复述技术动作 |
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

    // 查主记录与明细，后续按两者共同口径重算衍生数据
    BizEntity entity = bizService.getById(id);
    List<BizDetail> details = detailService.listByBizId(id);

    // 无记录或无明细时清空衍生数据
    if (Objects.isNull(entity) || CollectionUtils.isEmpty(details)) {
        deleteById(id);
        return;
    }

    // 按业务规则计算衍生行
    List<SummaryRow> rows = buildSummaryRows(entity, details);

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
public boolean save(BizDto dto) {
    // 入参与业务边界校验
    validateBiz(dto, null);

    // 转成可落库对象，并沿用项目审计字段写法
    BizEntity entity = BizConvert.INSTANCE.dtoToEntity(dto);
    // 审计字段：MetaObjectHandler 或手动 setOpUserId/setOpDate，跟 architecture.md

    return save(entity);
}
```

## 与注释的配合

步骤块、Javadoc、占位词、并行查询模板均以 [comments.md](comments.md) 为准。本文只保留结构要点：

- 公开方法：多行 Javadoc + 步骤分块（已触发条件时）
- 块间空一行；不用 `----` 装饰线
- 复杂分支在块内补单行 `//`
- 步骤注释只写业务口径/边界/前置条件；`查询数据库`、`遍历列表`、`封装结果` 这类实现噪音直接删

## 常见陷阱

| 陷阱 | 对策 |
|------|------|
| 只有 Javadoc、体内无分块（已触发条件） | [comments.md#步骤分块](comments.md#步骤分块) |
| 占位词（下面 / 补齐 / 构建索引） | [comments-examples.md#占位词禁用法](comments-examples.md#占位词禁用法) |
| 步骤注释只写技术动作（查库、遍历、set 字段） | 改成业务口径、过滤条件、跳过条件或删除 |
| 给一行 Controller 委托也写步骤块 | 不必，见上文豁免场景 |
| 过度抽私有方法 | 只抽重复或难一眼读懂的块 |

更多见 [pitfalls.md](pitfalls.md)。

## 存量大文件

修改 500+ 行的 `*ServiceImpl` 等存量大类时，见 [incremental-change.md](incremental-change.md)（只动相关方法、不顺手整类重构）。
