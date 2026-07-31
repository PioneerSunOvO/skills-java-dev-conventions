# Web / Controller 示例

与 [architecture.md](architecture.md)、[comments.md](comments.md) 配套。示例中的 **`R<T>`** 表示项目统一响应包装，占位用。

## 薄控制器

```java
@PostMapping("/add")
public R<Boolean> add(@Validated(CreateGroup.class) @RequestBody BizDto dto) {
    return R.result(bizService.save(dto));
}
```

## 不推荐：Controller 里写业务

```java
@PostMapping("/add")
public R<Boolean> add(@RequestBody BizEntity entity) {
    BizEntity db = bizMapper.selectById(entity.getId());
    if (db.getAmount().compareTo(threshold) > 0) { ... }
    bizMapper.insert(entity);
    return R.ok(true);
}
```

## Swagger / OpenAPI 二选一

```java
@Operation(summary = "补全建议", description = "按关键词返回建议列表；无匹配时 data 为空列表")
@GetMapping("/suggest")
public R<List<BizVo>> suggest(@Valid SuggestParam param) {
    return R.ok(bizService.suggest(param));
}
```

## 统一响应包装

```java
// 只表达“跟项目统一包装”，不发明新类
return R.ok(data);
return R.result(flag);
```
