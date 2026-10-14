# 常见陷阱与交付清单

版本误判全表见 [versions.md](versions.md)。判空/Stream 细则见 [coding-patterns.md](coding-patterns.md)；注释反面见 [comments-examples.md](comments-examples.md)。

## 1. 分层与对象

| 陷阱 | 对策 |
|------|------|
| Entity 作 API 入/出参 | DTO+校验；出参 VO |
| 接口 Param/Vo 字段无文档注解（Knife4j/Swagger 说明为空） | 按项目文档栈补字段注解：OpenAPI3 `@Schema`；Swagger2 `@ApiModelProperty` → [architecture.md#swagger--openapi-文档强约束](architecture.md#swagger--openapi-文档强约束) |
| 同一字段重复 `@Schema` / `@ApiModelProperty` | 只保留一条完整注解 |
| `List` 返回缺 `responseContainer = "List"` | `@ApiOperation(response=T.class, responseContainer="List")` → 同上 |
| Swagger `response` 写 Entity 或 Vo 无 `@ApiModel`/`@Schema` | 改 Vo + 类/字段文档注解；核对 `/v2/api-docs` 或 `/v3/api-docs` → 同上 |
| 数据类手写 get/set/equals/toString | Lombok `@Data` 等 → [coding-patterns.md#lombok](coding-patterns.md#lombok) |
| 新写 `BeanUtils`/`BeanUtil.copyProperties` | MapStruct `*Convert`（可静态排查字段引用）→ [coding-patterns.md#MapStruct](coding-patterns.md#MapStruct) |
| Controller 调 Mapper；Convert 写状态机/查库 | 经 Service；Convert 仅映射与无 I/O 组装 |
| 跨业务直接用对方 Mapper / 返回对方表实体；或形态 B 依赖对方实现模块 | 只调对方对外接口，传 DTO；Maven 只依赖契约模块 → [architecture.md#跨模块契约](architecture.md#跨模块契约) |
| 落点乱：启动壳写业务 / 默认就新建业务模块 | 先定落点；启动壳只组装；新建底座 starter 见骨架决策树 → [architecture.md#模块落点与依赖方向](architecture.md#模块落点与依赖方向) |
| 按 admin/app 复制整套 Service | 多端只分 Controller，共用领域 Service → 同上 |
| 一处横切就抽「框架」或两处复制不抽 | ≥2 处复用再下沉 → [framework-patterns.md#下沉门槛](framework-patterns.md#下沉门槛) |
| 底座依赖业务 `*-api` / 业务枚举 | 技术枚举与采集 DTO 放 `*-common` 或本 starter → [architecture.md#多模块工程骨架](architecture.md#多模块工程骨架) |
| 空壳 starter 未填实就当已实现引用 | 先填 AutoConfiguration / 能力，再让业务依赖 → 同上 |
| 启动壳无脑全开全部底座 starter | 业务模块按需声明依赖 → 同上 |
| 未授权就大改底座 / 安全组件 | 须用户明确要求或 quickref 允许 → 同上「改底座门槛」 |
| 业务散落魔法错误码字符串 | 号段 + 集中 ErrorCode → [architecture.md#业务错误码](architecture.md#业务错误码) |
| **错误码与失败原因不一致（混用）** | 一原因一码；缺则新增；禁「差不多就行」复用 → 同上「场景一一对应」 |
| Service 循环调 Mapper | 批量 / XML 一次返 VO（N+1 **硬门禁**） |

## 2. 事务 / 异常 / 日志

| 陷阱 | 对策 |
|------|------|
| 未 `rollbackFor`；同类自调用事务 | 显式 `rollbackFor=Exception.class`；拆类或代理 |
| 回调无幂等；CAS 不检查影响行数 | 唯一键+状态机；`rows==0` 明确处理 → [data-access.md#外部回调与幂等](data-access.md#外部回调与幂等) |
| 用「成功删幂等 Key」当锁 | 幂等 ≠ 锁 → [data-access.md#幂等与分布式锁](data-access.md#幂等与分布式锁) |
| 跨模块吞掉失败包装当成功 | `checkError` / 等价方法 → [architecture.md#统一响应与异常](architecture.md#统一响应与异常) |
| 空 catch / `return null`；只打 `getMessage()` | log + 抛业务异常；`log.error("…", e)` |
| 抛错文案与真实原因不符（排障误导） | 错误码场景一一对应 → [architecture.md#业务错误码](architecture.md#业务错误码) |
| `currentTimeMillis` 手算耗时打日志 | `StopWatch` + 项目格式化辅助 → [coding-patterns.md#耗时日志-stopwatch](coding-patterns.md#耗时日志-stopwatch) |
| 日志打敏感信息 | 脱敏；密钥永不落日志 |

## 3. 判空 / 比较 / 工具类 / Optional / Stream / MapStruct（摘要）

→ [coding-patterns.md](coding-patterns.md) 权威；交付前对照下表：

| 陷阱 | 对策 |
|------|------|
| `== null` / 裸 `.equals` / 双重 isEmpty | 工具类速查表 → [#判空与工具类](coding-patterns.md#判空与工具类) |
| BeanUtils / BeanUtil | MapStruct → [#MapStruct](coding-patterns.md#MapStruct) |
| 状态魔法数散落业务 | 富枚举 `isXxx`/`haveXxx` + 入参校验 → [#富枚举与入参校验](coding-patterns.md#富枚举与入参校验) |
| Optional 字段 / 层层 map / 流内查库 | → [#optional](coding-patterns.md#optional)、[#stream](coding-patterns.md#stream-流式处理) |
| 为达标整文件改写法 | **禁止**；只改本次片段 |

## 4. SQL / 性能

| 陷阱 | 对策 |
|------|------|
| `SELECT *` 再转 VO；内存分页 | XML 只查需要列；DB 分页见 [versions.md#分页三分支](versions.md#分页三分支) |
| 拼接用户输入；无索引 `%kw%` | `#{}`；评估索引或 ES |
| 仅为消噪自造 `xxxIfPresent` | 用 MP 原生条件参数；**若项目已有 Wrapper 扩展则跟邻** → [data-access.md](data-access.md) |
| CAS 不检查行数 / 回调无终态短路 | → [data-access.md#外部回调与幂等](data-access.md#外部回调与幂等)、[framework-patterns.md#cas-更新封装](framework-patterns.md#cas-更新封装) |

## 5. 注释 / 方法结构

| 陷阱 | 对策 |
|------|------|
| 复述代码、占位词、AI 套话、字段名复述 | [comments-examples.md#占位词禁用法](comments-examples.md#占位词禁用法) |
| 只写「调用了什么」，不写为什么/边界 | [comments.md](comments.md)「为什么优先」 |
| 触发分块却无步骤块；80+ 行无分块 | [comments.md#步骤分块](comments.md#步骤分块)；新代码不合格，存量见 [incremental-change.md](incremental-change.md) |
| 只有总结性步骤、难懂参数/分支无解释性单行 | 补解释性单行 → [comments.md#步骤分块](comments.md#步骤分块)；标杆 [范本 G](examples-method-structure.md#步骤分块范本-g检索客户端型) |
| 机械套「固定三步」或块间无空行 | 按实际阶段拆分；块间空一行 |
| 编排方法步骤风格混乱（有无编号混用） | 统一 `// 1.` / `// 1.1` → [method-structure.md#编号步骤](method-structure.md#编号步骤) |
| 易误用 API 不写决策注释 | 问题/回答或「为什么」→ [comments.md#设计决策注释](comments.md#设计决策注释) |
| 裸改租户/权限 ThreadLocal | `XxxUtils.execute*` → [framework-patterns.md#上下文作用域](framework-patterns.md#上下文作用域-ttltl--utils) |
| 过早抽 Handler/Calculator | 无第二扩展预期则留步骤块 → [method-structure.md#扩展点抽取](method-structure.md#扩展点抽取) |
| 过度 `processXxx` 空壳 | 只抽重复或难读块 |

## 6. 格式 / 版本 / 存量

| 陷阱 | 对策 |
|------|------|
| 注解长短交错；长行首参对齐 | [coding-patterns.md#注解排列](coding-patterns.md#注解排列)（风格建议） |
| SB3 仍 javax；Java8 用 record；无 MP 却 Wrappers | [versions.md](versions.md) |
| 擅自改 pom 版本 | 禁止，除非用户明确要求 |
| 巨型类顺手全文件重构 | [incremental-change.md](incremental-change.md) |

## 7. AI 气味（一眼扫）

与邻文件比「多出来的抽象」无收益就删。常见：硬写 Javadoc、占位词、手写 `== null`、BeanUtils、Stream 内查库、处处 Optional。

<a id="完整交付清单"></a>

## 8. 交付清单

### 环境

- [ ] 画像明确（项目 quickref / CLAUDE / versions 捷径）；语法与 `java.version` 一致；无 quickref 时是否已提示用户创建
- [ ] javax/jakarta、持久层、分页、注入跟邻代码

### 结构与数据

- [ ] 无 Entity 穿透；Controller 不直调 Mapper；无 N+1；写操作有事务
- [ ] 跨业务只走对方对外接口/契约；未直用对方 Mapper/表实体；形态 B 未依赖对方实现模块；落点正确；错误码落本模块号段且与失败场景一一对应（如适用）
- [ ] 若动到底座 starter：已授权；未依赖业务 `*-api`；空壳已填实；启动壳未无脑全开 → [architecture.md#多模块工程骨架](architecture.md#多模块工程骨架)
- [ ] 金额/时间/单位跟项目；复杂查询 XML 返 VO（如适用）
- [ ] 回调/状态流转：幂等键、CAS 行数、终态短路说清；未把幂本当锁

### 质量

- [ ] 新写判空/比较/集合工具类/Optional/Lombok/MapStruct/Stream/耗时日志符合 coding-patterns；异常与日志跟邻模块
- [ ] 枚举状态无魔法数；入参枚举校验跟邻（如适用）
- [ ] 注释：类三要素、必须项、步骤分块（编排优先 `// 1.`）、**为什么/设计问答**；无 AI/占位/复述/口语白话（见 comments 自检）
- [ ] 公开方法约 15–40 行或已分块；校验集中 `validate*`（如适用）
- [ ] 若动到上下文/幂等注解/Handler：跟邻封装，未平行造轮子（见 [framework-patterns.md](framework-patterns.md)）

### 范围

- [ ] 未改用户未要求的模块/配置/依赖；未提交密钥
- [ ] 行为变更有验证证据（见 [tests-policy.md](tests-policy.md)）
