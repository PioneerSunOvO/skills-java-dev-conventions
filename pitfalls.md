# 常见陷阱与交付清单

版本误判全表见 [versions.md](versions.md)。判空/Stream 细则见 [coding-patterns.md](coding-patterns.md)；注释反面见 [comments-examples.md](comments-examples.md)。

## 1. 分层与对象

| 陷阱 | 对策 |
|------|------|
| Entity 作 API 入/出参 | DTO+校验；出参 VO |
| Controller 调 Mapper；Convert 写业务 if | 经 Service；业务回 Service |
| Service 循环调 Mapper | 批量 / XML 一次返 VO（N+1 **硬门禁**） |

## 2. 事务 / 异常 / 日志

| 陷阱 | 对策 |
|------|------|
| 未 `rollbackFor`；同类自调用事务 | 显式 `rollbackFor=Exception.class`；拆类或代理 |
| 回调无幂等 | 唯一键+状态机；注释写清 |
| 空 catch / `return null`；只打 `getMessage()` | log + 抛业务异常；`log.error("…", e)` |
| 日志打敏感信息 | 脱敏；密钥永不落日志 |

## 3. 判空 / Stream（摘要）

完整对照 → [coding-patterns.md#判空与工具类](coding-patterns.md#判空与工具类)、[#stream-流式处理](coding-patterns.md#stream-流式处理)。

| 陷阱 | 对策 |
|------|------|
| 新写 `== null` / `list==null\|\|isEmpty` | `Objects.*` / `CollectionUtils` / `StringUtils.isBlank` |
| 为达标整文件改判空 | **禁止**；只改本次片段 |
| forEach/map 内查库；炫技长链 / peek / parallel | 批量查；链≤4；禁 peek 业务；禁默认并行 |

## 4. SQL / 性能

| 陷阱 | 对策 |
|------|------|
| `SELECT *` 再转 VO；内存分页 | XML 只查需要列；DB 分页见 [versions.md#分页三分支](versions.md#分页三分支) |
| 拼接用户输入；无索引 `%kw%` | `#{}`；评估索引或 ES |

## 5. 注释 / 方法结构

| 陷阱 | 对策 |
|------|------|
| 复述代码、占位词、AI 套话、字段名复述 | [comments-examples.md#占位词禁用法](comments-examples.md#占位词禁用法) |
| 触发分块却无步骤块；80+ 行无分块 | [comments.md#步骤分块](comments.md#步骤分块)；新代码不合格，存量见 [incremental-change.md](incremental-change.md) |
| 过度 `processXxx` 空壳 | 只抽重复或难读块 |

## 6. 格式 / 版本 / 存量

| 陷阱 | 对策 |
|------|------|
| 注解长短交错；长行首参对齐 | [coding-patterns.md#注解排列](coding-patterns.md#注解排列)（风格建议） |
| SB3 仍 javax；Java8 用 record；无 MP 却 Wrappers | [versions.md](versions.md) |
| 擅自改 pom 版本 | 禁止，除非用户明确要求 |
| 巨型类顺手全文件重构 | [incremental-change.md](incremental-change.md) |

## 7. AI 气味（一眼扫）

与邻文件比「多出来的抽象」无收益就删。常见：可省略项硬写多行 Javadoc；`首先/其次/最后`；占位词；手写 `== null`；Stream 内查库；处处 Optional；引入项目未用的 JSON API。

<a id="完整交付清单"></a>

## 8. 交付清单

### 环境

- [ ] 画像明确（项目 quickref / CLAUDE / versions 捷径）；语法与 `java.version` 一致；无 quickref 时是否已提示用户创建
- [ ] javax/jakarta、持久层、分页、注入跟邻代码

### 结构与数据

- [ ] 无 Entity 穿透；Controller 不直调 Mapper；无 N+1；写操作有事务
- [ ] 金额/时间/单位跟项目；复杂查询 XML 返 VO（如适用）

### 质量

- [ ] 新写判空/Stream 符合 coding-patterns；异常与日志跟邻模块
- [ ] 注释：类三要素、必须项、步骤分块；无 AI/占位/复述（见 comments 自检）
- [ ] 公开方法约 15–40 行或已分块；校验集中 `validate*`（如适用）

### 范围

- [ ] 未改用户未要求的模块/配置/依赖；未提交密钥
- [ ] 行为变更有验证证据（见 [tests-policy.md](tests-policy.md)）
