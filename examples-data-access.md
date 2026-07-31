# 数据访问示例

与 [data-access.md](data-access.md)、[versions.md](versions.md) 配套。示例类名均为占位。

## MyBatis-Plus 动态条件

```java
Wrappers.lambdaQuery(BizEntity.class)
    .eq(Objects.nonNull(param.getStatus()), BizEntity::getStatus, param.getStatus())
    .in(CollectionUtils.isNotEmpty(param.getIds()), BizEntity::getId, param.getIds())
    .like(StringUtils.isNotBlank(param.getKeyword()), BizEntity::getName, param.getKeyword())
    .orderByDesc(BizEntity::getId);
```

## 原生 MyBatis

```java
List<BizEntity> selectByUserId(@Param("userId") Long userId);
```

```xml
<select id="selectByUserId" resultType="com.example.entity.BizEntity">
    SELECT id, user_id, status
    FROM t_biz
    WHERE user_id = #{userId}
    ORDER BY id DESC
</select>
```

## Spring Data JPA

```java
List<BizEntity> list = bizRepository.findByUserIdAndDeletedOrderByIdDesc(
    userId, deletedFlag
);
```

## 分页

```java
// PageHelper
PageHelper.startPage(param.getPageIndex(), param.getPageSize());
List<BizVo> list = bizMapper.selectBizVoList(param);
```

```java
// MyBatis-Plus
Page<BizEntity> page = new Page<>(param.getPageIndex(), param.getPageSize());
IPage<BizEntity> result = bizMapper.selectPage(page, wrapper);
```

```java
// JPA
Pageable pageable = PageRequest.of(param.getPageIndex() - 1, param.getPageSize());
Page<BizEntity> result = bizRepository.findAll(spec, pageable);
```
