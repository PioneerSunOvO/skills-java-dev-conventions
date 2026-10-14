# 默认入口：按改动类型选读

**每次写/改 Java 先读本表。** 选中一行 → **只打开**「先看」列的 1～2 个文件 → 按「确认项」勾选 → 写完再扫文末超短口令。  
**不要**为小改通读 `SKILL.md` 全文，或整包翻 `examples*.md` / `comments-examples.md`（按本表打开 1～2 个细则即可）。

## 首次使用（约 3 分钟）

0. **项目 quickref**：探测 `.cursor/skills/*-quickref/SKILL.md`（或 `.agents/skills/`）。**若无**且画像不足 → 读 [project-quickref.md#Agent：未发现 quickref 时](project-quickref.md#agent-未发现-quickref)，**提示用户创建**后再改业务代码（见该文探测表）  
1. 读 [versions.md](versions.md)（或项目 skill / `CLAUDE.md` 捷径）→ 完成环境画像；语法上限见 [java-版本语法](versions.md#java-版本语法)  
2. 回到本表 → 按「我在改什么」只读 1～2 个细则  
3. 交付前扫文末超短口令；完整清单见 [pitfalls.md](pitfalls.md)

## 按改动类型

| 我在改什么 | 先看 | 确认项 |
|-----------|------|----------|
| 新增 / 改 Java 业务方法 | `architecture.md`；注释用 `comments.md`；抛业务异常时必看 `architecture.md#业务错误码` | [ ] 无 Entity 穿透<br>[ ] 分层未乱<br>[ ] 必要注释已补<br>[ ] 错误码与失败场景一一对应（未混用邻近码） |
| 新增 / 改业务错误码或抛异常分支 | `architecture.md#业务错误码` | [ ] 常量名/Javadoc/文案与分支原因一致<br>[ ] 缺码已新增而非复用「差不多」的码<br>[ ] 入参校验分场景分抛 |
| 落点不明 / 新模块或新子域 / 启动壳改动 | `architecture.md#模块落点与依赖方向`；多模块仓再读 `#多模块工程骨架` | [ ] 定边界（业务 vs 横切）<br>[ ] 默认不新建**业务**模块；新建底座 starter 仅走下行<br>[ ] 启动壳未塞领域规则 |
| 新建 / 扩展底座 Starter 或填空壳 / SPI 解耦横切 | `architecture.md#多模块工程骨架`；下沉门槛见 `framework-patterns.md#下沉门槛` | [ ] 已获用户授权或 quickref 允许<br>[ ] 先判业务域 vs 底座<br>[ ] 有空壳则填实，不另开同名<br>[ ] 底座未依赖业务域/`*-api`<br>[ ] 碰表/规则走 SPI<br>[ ] BOM + AutoConfiguration.imports 已齐<br>[ ] 业务按需引入 |
| 新增 / 改 Controller 接口或 Swagger 注解 | `architecture.md#swagger--openapi-文档强约束` | [ ] 文档栈跟项目（Swagger2 / springdoc，不混用）<br>[ ] Param/Vo **每个字段**有 `@Schema` 或 `@ApiModelProperty`<br>[ ] `List` 返回已加 `responseContainer = "List"`（Swagger2）<br>[ ] `response` 为 Vo 非 Entity<br>[ ] 继承 Entity 的 Vo 有独立 `@ApiModel`/`@Schema`<br>[ ] 多端未复制整套 Service |
| 新增 / 改 Entity / DTO / Param / VO | `architecture.md`、`coding-patterns.md#lombok`；若为接口入出参再读 `#swagger--openapi-文档强约束` | [ ] 数据类用 Lombok<br>[ ] 未手写 get/set/equals/hashCode/toString<br>[ ] 继承基类时 `callSuper` 跟邻<br>[ ] **接口 Param/Vo：字段文档注解已齐** |
| 枚举状态 / 入参枚举校验 | `coding-patterns.md#富枚举与入参校验` | [ ] 无魔法状态数<br>[ ] 校验与合法值同源 |
| 500+ 行大类局部修改 | `incremental-change.md` | [ ] 只改相关片段<br>[ ] 未顺手重构整类 |
| 判空 / 比较 / 工具类 / Optional / Lombok / Stream / MapStruct / 耗时 | `coding-patterns.md` | 见 [coding-patterns.md](coding-patterns.md) 各节；交付前扫 [pitfalls §3](pitfalls.md#3-判空--比较--工具类--optional--stream摘要) |
| SQL / Mapper XML / 分页 | `data-access.md`；分页选型见 `versions.md` | [ ] 无内存分页<br>[ ] 无循环查库<br>[ ] 复杂查询一次返 VO<br>[ ] 动态条件用原生条件参数（勿自造 IfPresent） |
| 回调 / MQ / 幂等 / 状态流转 | `data-access.md`、`tests-policy.md` | [ ] 幂等键明确<br>[ ] CAS/终态短路<br>[ ] 幂等≠分布式锁<br>[ ] 重复调用有验证 |
| 金额 / 税费 / 汇总 | `data-access.md`、`tests-policy.md` | [ ] `BigDecimal`<br>[ ] 单位明确<br>[ ] `compareTo` / 舍入说清 |
| 鉴权 / 数据范围 / 脱敏 | `security.md`；数据权限 SPI/切面再读 `architecture.md#多模块工程骨架`、`framework-patterns.md#上下文作用域` | [ ] 不信任前端 userId<br>[ ] 日志未泄露敏感信息<br>[ ] 有 Context 时禁裸改 ThreadLocal |
| 注释 / 类头 / 长方法 | `comments.md`、`method-structure.md`（去 AI 味/过口语/形状按需再开 `comments-examples`；检索型见 `examples-method-structure` 范本 G） | [ ] 类三要素<br>[ ] 触发时：总结性步骤 + 块间空行<br>[ ] 难懂处有解释性单行<br>[ ] 步数按实际阶段（非机械 3 步）<br>[ ] 只写业务口径/为什么<br>[ ] 语气：通俗书面，无口语<br>[ ] 无 AI/机翻味 |
| 多供应商 / 渠道 SDK | `architecture.md#多供应商适配`；类内拆法 `coding-patterns.md#多供应商模板方法`；**已拆底座三层的仓**再读 `#多模块工程骨架` | [ ] Service 不直调多家 SDK<br>[ ] 多模块底座仓：先定 starter/SPI（空壳先填实） |
| 租户/数据权限等上下文切换 | `framework-patterns.md#上下文作用域` | [ ] 用 Utils.execute*，禁裸改 ThreadLocal<br>[ ] 注解与 Util 语义一致 |
| 新声明式横切（幂等/限流等） | `framework-patterns.md#声明式横切`、`data-access.md#幂等与分布式锁` | [ ] 注解+Aspect+策略分层<br>[ ] 幂等未当成锁<br>[ ] ≥2 处复用才下沉共享（见下沉门槛） |
| 订单等生命周期扩展 | `framework-patterns.md#生命周期-handler`、`method-structure.md#扩展点抽取` | [ ] default 钩子 + List 注入<br>[ ] 无第二扩展预期不硬抽 |
| 状态机 CAS 写库 | `framework-patterns.md#cas-更新封装`、`data-access.md#外部回调与幂等` | [ ] 条件更新<br>[ ] 检查影响行数<br>[ ] 回调终态短路 |
| YAML 配置注释 | `config-yml.md` | [ ] 有段分割符<br>[ ] 用途/单位/开关已写<br>[ ] 无「本节用于…」「下面是…」 |
| 画像不明 / 换仓首次 | `versions.md`（含 [java-版本语法](versions.md#java-版本语法)、[分页三分支](versions.md#分页三分支)） | [ ] JDK / Boot 代际与可用语法<br>[ ] javax/jakarta<br>[ ] 持久层与注入风格<br>[ ] 分页选型 |
| 交付前声称「改好了」 | `pitfalls.md`、`tests-policy.md` | [ ] 有验证证据<br>[ ] 风险说清<br>[ ] 未越权改依赖/敏感配置 |

硬门禁摘要（Entity / N+1 / 类三要素 / 局部小改）见 [SKILL.md](SKILL.md#硬门禁)。

## 超短口令

```text
1. 画像明确了吗？（项目 quickref / CLAUDE / versions 捷径；无 quickref 时是否已提示用户？）
2. 落点对了吗？有没有越层、Entity 穿透、N+1？跨业务是否只走对方对外接口（形态 B 未依赖对方实现模块）？启动壳有没有写业务？改底座是否已授权？新建 starter 是否违反「底座不依赖业务/`*-api`、碰表走 SPI、按需引入」？
3. 这次改动会不会改变行为？会的话证据在哪？
4. 有幂等/CAS/金额/权限/敏感日志风险吗？幂等有没有被当成锁？上下文有没有裸改 ThreadLocal？
5. 判空/比较/Optional/Stream 是否恰当（工具类跟邻、非裸 equals、非层层 Optional、无流内查库）？
6. 注释和实现还一致吗？有没有写清「为什么」/设计问答？语气是通俗书面（非口语）吗？有没有 AI/机翻味？长方法是否 `// 1.` 编号分块？
7. 只读了本表对应的 1～2 个细则，没有通读 examples 吧？横切高阶是否误读成必须新造框架？≥2 处相同横切是否该下沉？
8. 改了接口入出参？Param/Vo 每个字段是否有文档注解（`@Schema` / `@ApiModelProperty`，跟项目文档栈）？`List` 是否 `responseContainer = "List"` 且 `response` 为 Vo？多端是否共用 Service？
9. 抛业务异常时：错误码是否与本分支真实失败原因一一对应？有无「入参空却报资源未启用」这类混用？
```
