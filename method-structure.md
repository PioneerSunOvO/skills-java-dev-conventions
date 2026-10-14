# 方法结构与私有方法抽取

与 [SKILL.md](SKILL.md) 配套。**步骤分块写法、分级表、Javadoc 标准**以 [comments.md](comments.md) 为权威；本文只讲方法体组织与私有方法抽取。

## 步骤分块标杆

- 方法 Javadoc、两层写法（总结性 + 解释性）、占位词表 → [comments.md](comments.md) + [comments-examples.md](comments-examples.md)
- 体内：**总结性** `//` 块首 → 本步代码（难懂处加**解释性**单行）→ **空一行** → 下一总结块
- **何时必须分块、何时豁免** → [comments.md#步骤分块](comments.md#步骤分块)
- 完整范本（切面 / 编排 / 多路查询 / **检索客户端型**）→ [examples-method-structure.md](examples-method-structure.md)

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
| 每步一块 | 块首一行**总结性** `//`，说明本阶段业务目的、口径与跳过条件 |
| 块间空行 | 每总结块结束后空一行再写下步 |
| 早退集中 | 不满足条件则 `return`/`continue`，注释写在块首或紧贴早退行 |
| 解释性单行 | 难懂参数/分支/兜底/排序次键等，紧贴代码行上方；不与总结性同义重复 |
| 步数按实际 | 2～N 步均可，禁止机械套「固定三步」 |
| 不用 `----` | 靠空行 + 步骤注释分层，不用装饰线 |

<a id="编号步骤"></a>

### 编号步骤（推荐）

公开编排方法（创建单、支付回调、复杂审核等）**优先**：

```text
// 1. 校验与幂等短路
// 1.1 …
// 1.2 …
// 2. 外部单据校验
// 3. CAS 更新状态
// 4. 扩展点 / 副作用
// 5. 记日志
```

| 要求 | 说明 |
|------|------|
| 同文件统一 | 不要混用「有编号 / 无编号 / 有无空格」三种风格 |
| 子步骤 | 用 `1.1` `2.1`，避免过深（一般两级够用） |
| 与问答注释分工 | 步骤块写阶段意图；设计取舍用 [设计决策注释](comments.md#设计决策注释) |

**不合格**：已触发分块条件，却整段无空行、无总结性步骤、只有方法 Javadoc；或仅有总结性而难懂处无解释性单行。

<a id="公开方法编排优先"></a>

## 公开方法：编排优先

行数口径分工（避免与 pitfalls / incremental 打架）：

| 口径 | 含义 |
|------|------|
| 目标约 **15–50 行**（含空行与分块注释） | **新代码舒适区**：公开编排方法尽量落在此区间 |
| 交付自检「约 **15–40 行**或已分块」 | pitfalls 可勾选近似；已分块即可，不与上条冲突 |
| **>80 行且无分块** | **新代码不合格**；存量大方法见 [incremental-change.md](incremental-change.md) 作重构信号 |

| 指标 | 建议 |
|------|------|
| ≥ 15 行或有多个业务阶段 | 必须步骤分块（见 comments.md） |
| 业务复杂 | 用步骤分块保持可读，不要机械拆成多个一行 wrapper |

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

<a id="扩展点抽取"></a>

## 扩展点抽取（按需）

当公开方法里出现「可插拔阶段」或「固定先后的多段计算」时，再抽扩展点；否则步骤分块即可。

| 信号 | 抽法 | 注意 |
|------|------|------|
| 同一生命周期多处要挂逻辑（券、库存、积分…） | `XxxHandler` + `default` 空方法；主流程 `handlers.forEach(...)` | 需要稳定顺序时加 `@Order` 或文档约定 bean 序 |
| 计价/优惠有行业固定顺序 | `XxxCalculator` + 优先级常量 | 常量旁注释写清为何此序 |
| 仅一两处 if | **不抽**，留在 Service 步骤块 |

上下文 → [architecture.md#扩展点与有序链](architecture.md#扩展点与有序链)。

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

完整范本（含步骤分块与边界注释）见 [examples-method-structure.md 范本 D](examples-method-structure.md#步骤分块范本-d-service方法)。此处不重复贴代码，避免双份维护漂移。

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
    // 审计字段：MetaObjectHandler 或手动赋值；setter 名跟邻 Entity（见 architecture.md）

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
