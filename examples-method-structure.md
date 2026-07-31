# 步骤分块范本

配合 [method-structure.md](method-structure.md) 与 [comments.md](comments.md#步骤分块) 使用。这里专放完整范本，主文只保留规则与入口。

## 范本 A：编排型 build

适用：报表节点组装、PDF/导出数据组装等。Javadoc 写口径与归并规则；体内按阶段分块，块间空一行。

```java
/**
 * 组装报表节点（固定仅 parentId 下一级）。
 * A、B 两类数量在业务层按 catId+业务键合并；内部 ID 与业务键在业务层换算。
 *
 * @param param 含 parentId、year、筛选类型列表
 * @return 带 reportData 的节点列表及末尾合计行
 */
private List<ReportTreeVo> buildReportNodes(ReportQueryParam param) {
    // 确定本次统计涉及的分类范围
    List<CatalogVo> levelList = resolveTargetCatalogs(param);
    if (CollectionUtils.isEmpty(levelList)) {
        return Collections.emptyList();
    }

    // 提取本级 ID，后续统计只在当前层级内归并
    List<Long> levelIds = extractIds(levelList);
    // 确定区域条件：parentId=0 用 catId IN，parentId≠0 用父级 fullId 前缀匹配
    QueryScope queryScope = resolveQueryScope(param.getParentId(), levelList, ...);

    // 加载本级年份统计数据
    YearStats yearStats = loadYearStats(...);
    // 按本级 ID 汇总统计数据
    Map<Long, ReportDataVo> reportById = aggregateByCatalog(levelIds, yearStats);

    // 组装报表节点，数量与金额都按当前口径写入
    List<ReportTreeVo> nodes = buildNodes(...);
    // 添加合计行
    nodes.add(buildTotalRow(...));

    return nodes;
}
```

## 范本 B：预览生成型 generate

适用：结算/订单等“生成预览、uuid 缓存、保存时读快照”流程。Javadoc 写并行口径与缓存边界；多路 SQL 每路一行说明，不用“下面…一并拉取”。

```java
/**
 * 生成业务明细预览。
 * 元数据单独查一次再传入各统计 SQL；多路统计并行查后合并计算。
 * 预览结果写入 Redis，保存时凭 uuid 读取。
 *
 * @param param 业务年份与范围列表
 * @return 父子结构明细、汇总金额和预览 uuid
 */
public PreviewVo generateDetail(GenerateParam param) {
    // 校验年份与范围，缺任一关键条件不生成预览
    validateGenerateParam(param);

    // 查元数据映射，各统计 SQL 按 ID 过滤，不再连维表
    List<ItemMetaVo> itemMetas = mapper.selectItemMetas(param.getYear());
    ...

    // 多路统计互不影响，并行查
    // 路线一：主业务聚合（说明统计口径）
    CompletableFuture<List<RowVo>> mainFuture = CompletableFuture.supplyAsync(..., executor);
    // 路线二：辅助数量（说明 quantityType 或来源表）
    CompletableFuture<List<RowVo>> auxFuture = CompletableFuture.supplyAsync(..., executor);
    // 路线三：历史已结金额（说明抵扣规则）
    CompletableFuture<List<RowVo>> historyFuture = ...
    CompletableFuture.allOf(mainFuture, auxFuture, historyFuture).join();

    // 关联展示字段（名称、单价等在业务层补齐）
    fillRowMeta(mainFuture.join(), ...);

    // 合并多源行并按规则算明细
    List<DetailVo> details = calculateDetails(...);

    // 生成预览头与汇总金额
    PreviewVo preview = buildPreviewHeader(details, param);
    // uuid 写入 Redis，保存时凭 uuid 读取
    cachePreview(preview);

    return preview;
}
```

## 范本 C：切面方法

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

    // 解析本次需要同步的业务主键
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

## 范本 D：Service 方法

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
    BizEntity entity = bizService.getById(id);
    List<BizDetail> details = detailService.listByBizId(id);

    // 无记录或无明细时清空衍生数据
    if (Objects.isNull(entity) || CollectionUtils.isEmpty(details)) {
        deleteById(id);
        return;
    }

    // 按业务规则计算衍生行
    List<SummaryRow> rows = buildSummaryRows(entity, details);

    // 先删后插，保证与源数据一致
    deleteById(id);
    if (CollectionUtils.isNotEmpty(rows)) {
        this.saveBatch(rows);
    }
}
```

## 范本 E：多数据源查询

```java
/**
 * 查询复合列表，含当期数据、历史关联及统计数量。
 *
 * @param param 含 year、分类筛选条件
 * @return 按分类组装后的列表
 */
public List<CatalogVo> getList(QueryParam param) {
    // 解析查询范围
    Integer year = param.getYear();
    List<Long> categoryIds = resolveCategoryIds(param);

    // 加载当期主数据，作为当前列表主体
    List<ItemVo> currentItems = baseMapper.selectItemsByYear(year, categoryIds);

    // 加载跨期关联数据，供历史对比与回填
    List<ItemVo> allItems = baseMapper.selectItemsByYears(year, year - 1);

    // 历史记录映射：按当期与主数据的交集键关联
    Map<Long, ItemVo> historyItemMap = buildHistoryMap(allItems, currentItems, year);

    // 查询历史统计数量
    Map<Long, Map<Long, Integer>> historyQtyMap =
            loadHistoryQuantities(year - 1, categoryIds, historyItemMap.keySet());

    // 按分类组装返回列表
    for (CatalogVo catalog : catalogList) {
        assembleCatalog(catalog, currentItems, historyItemMap, historyQtyMap);
    }
    return catalogList;
}
```
