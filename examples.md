# 对照示例

精简示例集；完整模板见各专题文件。

## 环境画像示例

```text
Java: 8 | Spring Boot: 2.2.5 | Spring Cloud: 无
持久层: MyBatis-Plus | 校验: javax.validation | 分页: PageInfo + Paging（非 PageHelper）
```

```text
Java: 17 | Spring Boot: 3.2 | Spring Cloud: 2023.0.x | Nacos + OpenFeign
持久层: MyBatis-Plus | 校验: jakarta.validation
```

## 语法降级（Java 8 仓库）

```java
// 禁用 List.of() → Arrays.asList()
// 禁用 var → 显式类型
// 禁用 String.isBlank() → StringUtils.isBlank()
```

## 持久层分支

**MyBatis-Plus：**

```java
List<Order> list = orderMapper.selectList(
    Wrappers.lambdaQuery(Order.class).eq(Order::getUserId, userId));
```

**原生 MyBatis：**

```xml
<!-- 按用户查有效订单 -->
<select id="selectByUserId" resultType="com.example.entity.Order">
    SELECT id, user_id, status FROM t_order
    WHERE user_id = #{userId} AND deleted = 0
</select>
```

## 分页

<a id="分页"></a>

分页写法以环境画像为准（见 [versions.md](versions.md#分页三分支)）。**禁止**全量查出后内存截取。

### MP + 项目自定义分页封装

常见于 `PageInfo` 包装入参、`Paging` 包装出参、`IPage` 承接 Mapper 分页结果。检测到项目有此类封装时优先用此分支，**以邻代码为准**。

```java
// 入参 Param 继承 BasePageOrderParam；复杂列表走 Mapper XML 分页
Page<Order> page = new PageInfo<>(orderPageParam);
IPage<OrderVo> iPage = orderMapper.selectOrderPageList(page, orderPageParam);
return new Paging<>(iPage);
```

### PageHelper（其他仓库）

检测到 `pagehelper-spring-boot-starter` 时用此分支，勿与上式混用：

```java
PageHelper.startPage(param.getPageIndex(), param.getPageSize());
List<OrderVo> list = orderMapper.selectOrderVoList(param);
return new Paging<>(new PageInfo<>(list));
```

MP 原生 `Page` + `selectPage`、JPA `Pageable` 见 [versions.md](versions.md#分页三分支)。

## YAML 注释示例

```yaml
############################## 服务端口 start #############################
server:
  port: 8080
############################## 服务端口 end ###############################

############################## 业务自定义配置 start #############################
app:
  feature:
    # 单张限额（元）；-1 无限制
    max-amount: -1
############################## 业务自定义配置 end #############################
```

## 注释润色

**类注释：**

```java
// 前：该类用于处理订单相关的业务逻辑，主要功能包括查询、更新、删除。
// 后：@description: 订单保存与状态流转；写操作在同一事务内更新主表与明细
```

**步骤注释：**

```java
// 前：// 解析
// 后：// 解析业务主键列表，表达式为空则跳过
```

## 步骤分块（Service）

```java
public void rebuildById(Long id) {
    // 主键为空直接返回
    if (Objects.isNull(id)) {
        return;
    }
    // 无明细时清空衍生数据
    ...
}
```

完整模板见 [method-structure.md](method-structure.md)。

<a id="git-提交说明"></a>

## Git 提交说明

规范见 [SKILL.md#git-提交说明](SKILL.md#git-提交说明)。正文用 **`-` 列表**。

**正例：**

```
feat(order): 新增订阅汇总表并在订单变更时自动同步

- 新增汇总表，按明细预计算订阅结果，便于列表与统计查询
- 订单保存、修改、删除后自动重算汇总行
- 管理端提供全量重建接口，用于历史数据补算
```

```
refactor(order): 重复订单查重改为一次关联查询

- 取消先查明细 id 再 in 订单号的两步查询，减少往返
- 主表关联明细，按收件人要素与订期一次匹配
```

**反例：**

```
feat(order): 新增 OrderSummary Entity 及 SyncAspect AOP

- 新增 Entity 和 Mapper
- 新增 Aspect
```

项目专属提交风格（如中文模块名、业务字段）见各仓库 `CLAUDE.md` 或项目 skill。
