# 自然中文注释

与 [SKILL.md](SKILL.md) 配套。**注释权威文件**：分级表、步骤分块、字段规则以此为准。类三要素为**硬门禁**，其余注释规则为**强约束**。项目 `CLAUDE.md` / rules 中的类注释模板优先。

形状示例（标杆 Javadoc、ExceptionHandler、MapStruct 等）→ [comments-templates.md](comments-templates.md)（按需）。  
占位词 / 套话 / 正反例 → [comments-examples.md](comments-examples.md)；润色对照 → [comments-rewrites.md](comments-rewrites.md)。

## 一页速查

| 项 | 规则 |
|----|------|
| **必须写** | 类三要素；公开业务方法 Javadoc；ExceptionHandler / Bean；配置 / Document / 对外 VO 字段 |
| **不要写** | 复述代码；实现噪音；AI 套话；占位词；机械全覆盖 Javadoc |
| **步骤分块** | ≥15 行，或 ≥2 业务阶段，或非显而易见分支 → 块首 `//` + 块间空一行；范本见 [examples-method-structure.md](examples-method-structure.md) |
| **三层分工** | Javadoc=方法口径；步骤块=阶段意图；块内行=分支/幂等/单位 |
| **Map 用词** | 内存 Map 用「映射 / 汇总」，不说「构建索引」（库表索引除外） |
| **审查** | [#30-秒自检](#30-秒自检审查口令) + [占位词表](comments-examples.md#占位词禁用法)；`@author` 禁 AI 名（见 [#author-取值](#author-取值)） |

<a id="30-秒自检审查口令"></a>

## 30 秒自检（审查口令）

```
1. 新类有三要素吗？@description 是职责还是 CRUD 列表？
2. 公开方法：第一句能说清「做什么 + 何时不做」吗？
3. 方法 ≥15 行或有业务分支：有步骤块和块间空行吗？
4. 字段注释是业务语义，还是字段名复述？
5. 改逻辑处注释同步了吗？
6. 写的是业务规则/边界，还是「遍历列表」「设置值」？
7. 像同事交代规则，还是机翻 / AI 套话？
8. 有占位词吗？→ comments-examples.md
9. @author 是真人或项目约定吗？有无 Cursor/Codex/Auto 等？
```

## Javadoc / 步骤块 / 块内行 — 三层分工

| 层级 | 写什么 | 不写什么 |
|------|--------|----------|
| **方法 Javadoc** | 职责、口径、边界、幂等/跳过；`@param`/`@return` 写业务含义 | 逐步流水账；与首个步骤块重复 |
| **步骤块首行** | 本阶段意图；有分支用冒号展开 | 复述 Javadoc；空泛「处理数据」 |
| **块内行** | 分支原因、单位、兜底、并发/幂等（仅复杂处） | 机械 for/if/getter 解说 |

## 注释分级表（必须 / 建议 / 可省略）

| 对象 | **必须** | **建议** | **可省略** |
|------|----------|----------|------------|
| **类** | `@description` + `@author` + `@since`（或项目模板） | 事务、幂等、切面顺序等边界 | — |
| **公开业务方法** | 多行 Javadoc：职责 + 边界；有参 `@param`；有返回 `@return` | 步骤分块（见 [#步骤分块](#步骤分块)） | — |
| **Controller** | Javadoc **或** 等价 Swagger/OpenAPI（二选一） | 校验/权限/空结果语义时再补 | 一行委托不写步骤块 |
| **`@Override` 实现** | 有超出接口的边界时补 | 接口已完整可不重复 | 纯转发无语义增量 |
| **`@ExceptionHandler` / `@Bean`** | 多行 Javadoc | — | — |
| **MapStruct 方法** | `toXxx`、含业务拼装的 `default`/`@AfterMapping` | 纯映射可依赖类注释 | 生成类 `*Impl` 不手写 |
| **私有方法** | — | 口径不直观、≥15 行、`validate*`/`build*` | 机械辅助 |
| **字段** | 见 [#字段注释](#字段注释) | Entity/DTO 与邻类、同域 VO 对齐 | 类注释已覆盖的简单 VO |
| **枚举常量** | 业务枚举每个常量 | 技术枚举写用途 | 纯占位枚举 |

不要给每个方法千篇一律 Javadoc；缺「必须」项才不合格。

## 强约束原则

- 新增/修改业务代码按分级表补注释；改逻辑必改注释
- 可省略：getter、简单 setter、≤3 行无分支委托
- **只写业务相关**：为什么做、何时做、口径/边界/单位/幂等；不写 IDE 可见的实现动作
- 朗读自检：像同事交代规则 → 合格；像翻译腔 → 重写

## 注释五层结构

| 层级 | 形式 | 写什么 |
|------|------|--------|
| **类** | `@description` + `@author` + `@since` | 职责 + 关键边界 |
| **方法** | 多行 `/** ... */` | 做什么、口径；`@param`/`@return` |
| **步骤块** | `// 动词+对象` | 本步目的；分支用冒号；**块间空一行** |
| **块内行** | `//` | 口径、单位、分支、幂等（仅复杂处） |
| **字段** | `/** ... */` | 业务含义、单位、条件展示 |

禁止：`// 1. 获取…`、`// 查询数据库`、`// 遍历列表`、`// 下面…`、`// 首先…其次…最后…`

## 方法 Javadoc

- 第一句：动词短语 + 可选范围/约束，不只重复方法名
- 第二句（如有）：口径、归并、跨接口一致性、幂等/跳过
- `@param`：业务含义与取值约定；禁止「查询参数」「入参对象」
- `@return`：结构/口径；空/0 含义；可用 `→`（`void` 不写，副作用写正文）

骨架表示例与完整标杆代码 → [comments-templates.md](comments-templates.md)。

| 行 | 写什么 |
|----|--------|
| 第一句 | 动词短语 + 可选（范围/约束） |
| 第二句 | 口径、归并、跨接口一致性、处理层级 |
| `@param` / `@return` | 业务含义，不复述类型 |

- 接口 Javadoc 已完整 → 实现类可不重复；有额外边界 → 实现上必须补
- 公开业务方法**不允许**单行 Javadoc；私有辅助/字段/枚举常量允许单行

## Controller 与 Swagger / OpenAPI

- `@ApiOperation` / `@Operation` 已写清职责与返回 → Javadoc 可省
- 文档注解仅一句话、未写边界 → 仍须 Javadoc
- **禁止** Javadoc 与文档注解矛盾
- 一行 `return R.ok(service.xxx(param))` → 不需要步骤分块

<a id="步骤分块"></a>

## 步骤分块

**满足任一即必须：** 方法体 ≥15 行；≥2 个业务阶段；非显而易见分支（幂等/跳过/早退/容错）。

**不必：** ≤3 行无分支委托；纯 getter/setter/`@Bean`；Controller 一行调 Service。

**写法：** 块首 `//` → 块间空一行 → 默认「动词+对象」，有分支用冒号 → 不用 `----` → 机械循环不必逐步注。

| 级别 | 何时用 | 格式 |
|------|--------|------|
| **简** | 意图一目了然 | `// 动词+对象` |
| **详** | 含分支口径或跨层约定 | `// 动词+对象：条件A 用…，条件B 用…` |

完整范本 → [examples-method-structure.md](examples-method-structure.md)。好差对照与占位词 → [comments-examples.md](comments-examples.md)。

<a id="多路并行查询示例"></a>

多路并行：**总起一句 + 每路一行说明口径**，join 后写合并；禁「下面…一并拉取」。见 [范本 B](examples-method-structure.md#多路并行查询块内写法)。

<a id="类注释硬性"></a>

## 类注释（硬门禁）

```java
/**
 * @description: 层级报表服务；A 类读主表汇总、B 类读计划表，区域过滤与列表接口一致。
 *              查询与导出均固定 parentId 下一级；数量在业务层按 categoryId+业务键归并。
 * @author: 张三
 * @since: 2026-06-30 18:30
 */
```

<a id="author-取值"></a>

### `@author` 取值

**禁止** AI/工具名：Cursor、Codex、Auto、Claude、GPT、Copilot、ChatGPT 等。

1. 项目 rules / 项目 skill 已约定 → 用约定值  
2. `git config user.name` 非空且非 AI 名 → 用之  
3. 操作系统登录名 → 仅兜底  
4. 仍无法确定 → **先问用户**，不得填「开发者」

`@since`：`YYYY-MM-DD HH:mm`。`@description` 空或 CRUD 功能列表 → 不合格。

<a id="字段注释"></a>

## 字段注释

- 写业务含义、单位、条件展示；**禁止**用 Java 字段名当注释
- **必须写**：`@ConfigurationProperties`、ES Document/mapping、对外 Param/VO、含义不直观的枚举常量
- 同域字段在 Document/VO/DTO 间口径一致
- 位置与示例 → [comments-templates.md](comments-templates.md)

<a id="语气与排版"></a>

## 语气与排版

- 短句直说；DTO、SpEL、幂等等术语不硬翻
- 多句用分号衔接；范围放括号；转换可用 `→`；中英空格
- 去 AI/机翻味 → [comments-examples.md](comments-examples.md)
- 与业务无关的描述默认不写 → [见到就删](comments-examples.md#与业务无关的描述--见到就删)

| 场景 | 怎么说 |
|------|--------|
| 数据库索引 | 「索引」 |
| ES mapping | 「索引 mapping 字段」 |
| 内存 Map | 「映射 / 按 X 汇总」，不说「构建索引」 |

## 与代码同步

改逻辑必改注释。方法结构 → [method-structure.md](method-structure.md)；陷阱 → [pitfalls.md](pitfalls.md)；范本 → [examples-method-structure.md](examples-method-structure.md)。
