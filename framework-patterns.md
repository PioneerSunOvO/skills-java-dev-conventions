# 框架级可复用模式（风格建议为主）

跨项目通用的「横切 / 扩展 / 上下文」写法。类名一律占位（`Biz*` / `Xxx*`）；真实类名见各仓 **项目 quickref**。

**规则等级**：除文中标明「强约束」的条目外，本节整体为**风格建议**——项目已有同构封装时**跟邻**，不要为达标重造一套。是否采用 Context / 声明式切面 / CAS 封装属风格建议；**一旦采用**，文中「强约束」条（如下沉门槛、有 Context 时禁裸改 ThreadLocal）必须遵守。

<a id="下沉门槛"></a>

## 下沉门槛（强约束）

| 情况 | 做法 |
|------|------|
| 仅 1 处使用 | 留在本域实现；**禁止**为「看起来专业」新建一套全局框架 |
| **≥ 2** 个模块/子域出现同构横切 | 下沉到共享组件（现有 starter / `common` / framework 跟邻）；禁止继续复制第二套 |
| 已有同构封装 | **跟邻复用**，不另起炉灶 |

横切怎么拆（注解 + Aspect + 策略）见下文；**何时**抽共享看本表。模块落点总览 → [architecture.md#模块落点与依赖方向](architecture.md#模块落点与依赖方向)。多模块工程里 Starter / SPI 落点 → [architecture.md#多模块工程骨架](architecture.md#多模块工程骨架)。

**本节索引：**

| 锚点 | 内容 |
|------|------|
| [#下沉门槛](#下沉门槛) | ≥2 处复用才抽共享 |
| [#上下文作用域](#上下文作用域-ttltl--utils) | TTL/TL Context + Utils.execute |
| [#声明式横切](#声明式横切-注解--aspect--策略) | 注解 + Aspect + Resolver/Rule |
| [#生命周期handler](#生命周期-handler) | default 钩子 + List 注入 |
| [#cas-更新封装](#cas-更新封装) | Mapper 条件更新 + 行数检查 |
| [#能力全景注释](#能力全景注释-package-info) | package-info 写清覆盖面 |
| [#与其它文件关系](#与其它文件关系) | 链到 architecture / data-access / comments |

<a id="上下文作用域-ttltl--utils"></a>

## 上下文作用域（TTL/TL + Utils）

> **强约束（有 Context 时）**：业务代码**禁止**裸 `set`/`remove` ThreadLocal；切换用 `XxxUtils.execute*(...)`，由 Filter/Interceptor 负责请求级 set/clear。

适用：租户、数据权限、动态数据源、操作人覆盖等「请求内有效、可嵌套/可异步传递」的上下文。

| 要点 | 做法 |
|------|------|
| 载体 | 优先可传递实现（如 `TransmittableThreadLocal`），避免 `@Async` / 线程池丢上下文 |
| 生命周期 | Web Filter / MQ Consumer：**set → try → finally clear** |
| 业务切换 | `execute(id, runnable)` / `executeIgnore(runnable)`：备份旧值 → try 执行 → finally 恢复 |
| 注解对齐 | 若有 `@XxxIgnore`，Aspect 语义须与 `Utils.executeIgnore` **文档写明一致** |
| 嵌套 | 可能嵌套调用时用栈（`LinkedList`）而非单值覆盖 |

```java
// 占位：ContextHolder / Utils 名跟项目
public final class BizTenantUtils {
    public static void execute(Long tenantId, Runnable runnable) {
        Long oldId = BizTenantContextHolder.getTenantId();
        Boolean oldIgnore = BizTenantContextHolder.isIgnore();
        try {
            BizTenantContextHolder.setTenantId(tenantId);
            BizTenantContextHolder.setIgnore(false);
            runnable.run();
        } finally {
            BizTenantContextHolder.setTenantId(oldId);
            BizTenantContextHolder.setIgnore(oldIgnore);
        }
    }

    public static void executeIgnore(Runnable runnable) {
        Boolean oldIgnore = BizTenantContextHolder.isIgnore();
        try {
            BizTenantContextHolder.setIgnore(true);
            runnable.run();
        } finally {
            BizTenantContextHolder.setIgnore(oldIgnore);
        }
    }
}
```

Javadoc 写清**副作用边界**（例如：忽略租户时强制改回「不忽略」再执行，结束后仍恢复）。

分层 / 数据权限类扩展 → [architecture.md](architecture.md)；注释「为什么」→ [comments.md#设计决策注释](comments.md#设计决策注释)。

<a id="声明式横切-注解--aspect--策略"></a>

## 声明式横切（注解 + Aspect + 策略）

一个横切能力拆成四层，**一件事一个注解**：

| 层 | 职责 | 示例角色 |
|----|------|----------|
| **注解** | 超时、提示文案、策略 Class、SpEL 参数 | `@BizIdempotent` / `@BizRateLimit` |
| **Aspect** | 固定流程：解析 Key → 占坑/计数 → proceed → 异常收尾 | 不写业务规则 |
| **策略接口** | `resolve(joinPoint, annotation)` | KeyResolver / Rule |
| **状态存储** | Redis SETNX、令牌桶、本地缓存等 | DAO / Template |

```java
// 占位：幂等注解上的设计决策注释（问答式，见 comments.md）
public @interface BizIdempotent {
    int timeout() default 1;
    TimeUnit timeUnit() default TimeUnit.SECONDS;
    Class<? extends BizIdempotentKeyResolver> keyResolver() default DefaultBizIdempotentKeyResolver.class;
    String keyArg() default "";

    /**
     * 发生异常时是否删除 Key。
     *
     * 问题：为什么异常要删 Key？
     * 回答：业务失败应允许重试；保留 Key 会误伤下一次合法请求。
     *
     * 问题：为什么不提供「成功也删 Key」？
     * 回答：那是互斥锁语义，应使用项目分布式锁注解，勿与幂等混用。
     */
    boolean deleteKeyWhenException() default true;
}
```

扩展方式：Spring 注入 `List<XxxKeyResolver>` → 构造成 `Map<Class, Resolver>`；注解上指定 `Class<? extends Xxx>`。

**强约束**：幂等 ≠ 分布式锁 → [data-access.md#幂等与分布式锁](data-access.md#幂等与分布式锁)。

<a id="生命周期-handler"></a>

## 生命周期 Handler

主流程会挂多种插件（券、库存、积分、分销…）时，用钩子接口代替巨型 `if-else`。

| 要点 | 做法 |
|------|------|
| 接口 | 各阶段 `default` 空方法，实现类只覆写需要的钩子 |
| 注入 | `List<BizOrderHandler>`（需要稳定顺序时用 `@Order` 或文档约定 bean 序） |
| 主流程 | `handlers.forEach(h -> h.afterPay(order))` |
| 注释 | 非显而易见的阶段选择写「为什么不在前置做」 |

```java
public interface BizOrderHandler {
    default void beforeCreate(BizOrder order, List<BizOrderItem> items) {}
    default void afterCreate(BizOrder order, List<BizOrderItem> items) {}
    default void afterPay(BizOrder order, List<BizOrderItem> items) {}
    default void afterCancel(BizOrder order, List<BizOrderItem> items) {}
}

// Service 编排（步骤编号见 method-structure.md）
// 2. 创建前扩展点
bizOrderHandlers.forEach(h -> h.beforeCreate(order, items));
// 3. 落库
// 4. 创建后扩展点
bizOrderHandlers.forEach(h -> h.afterCreate(order, items));
```

**何时不抽**：仅 1～2 处简单分支 → 留在 Service 步骤块。权威分工 → [architecture.md#扩展点与有序链](architecture.md#扩展点与有序链)、[method-structure.md#扩展点抽取](method-structure.md#扩展点抽取)。

<a id="cas-更新封装"></a>

## CAS 更新封装

状态机写库优先「条件更新 + 检查行数」，可在 Mapper `default` 方法消噪：

```java
// 占位：跟项目 Mapper / Wrapper
default int updateByIdAndStatus(Long id, Integer expectStatus, BizOrder update) {
    return update(update, new LambdaUpdateWrapper<BizOrder>()
            .eq(BizOrder::getId, id)
            .eq(BizOrder::getStatus, expectStatus));
}

// Service
int rows = orderMapper.updateByIdAndStatus(id, order.getStatus(), update);
if (rows == 0) {
    throw new BizException(BizErrorCodes.ORDER_STATUS_NOT_ALLOW);
}
```

回调场景还须：**终态 + 同一外部单号 → 直接 return（幂等）**；与金额/流水校验组合 → [data-access.md#外部回调与幂等](data-access.md#外部回调与幂等)。

<a id="能力全景注释-package-info"></a>

## 能力全景注释（package-info）

横切组件在 `package-info.java`（或模块 README）用编号列出**覆盖面与改造点**，避免后人只改 DB 漏掉 Redis/MQ/Async：

```java
/**
 * 多租户（示意），通常覆盖：
 * 1. DB：拦截器拼租户条件
 * 2. Redis：Key 拼接租户编号
 * 3. Web：解析 Header 写入上下文
 * 4. Security：防越权访问他租户
 * 5. Job / MQ / Async：上下文传递（TTL）与按租户扇出
 *
 * 业务切换请使用 BizTenantUtils，禁止裸改 ContextHolder。
 */
package com.example.framework.tenant;
```

类头「场景 / 不做的事 / 与某某语义一致」→ [comments.md](comments.md)。

<a id="与其它文件关系"></a>

## 与其它文件关系

| 主题 | 权威文件 |
|------|----------|
| 模块落点、**多模块工程骨架 / Starter / SPI**、包级/Maven 契约、错误码、响应贯通 | [architecture.md](architecture.md) |
| 下沉门槛（≥2 处） | 本文 [#下沉门槛](#下沉门槛) |
| 回调幂等、CAS、金额 | [data-access.md](data-access.md) |
| 富枚举、`@InEnum`、多供应商模板 | [coding-patterns.md](coding-patterns.md) |
| 步骤编号、Handler 抽取时机 | [method-structure.md](method-structure.md) |
| 问答式 /「为什么」注释 | [comments.md#设计决策注释](comments.md#设计决策注释) |
| 真实类名与禁改模块 | **项目 quickref**，勿写入本文件 |
