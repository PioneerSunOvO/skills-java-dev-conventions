---
name: java-dev-conventions
description: >-
  编写或修改 Java/Spring Boot 后端（含 MyBatis/MP/JPA）及配套 Mapper XML、
  Spring 配置（yml/yaml/properties）时使用；审查分层、注释、N+1、Entity 穿透时使用。
  触发词：Java, Spring Boot, .java, Mapper.xml, ServiceImpl, Java规范, 分层,
  中文注释, 机翻, AI味, 占位词, 套话, deslop, Git 提交, MyBatis, MapStruct,
  javax, jakarta, N+1, Entity穿透, 幂等, BigDecimal, Spring Boot 2→3 迁移。
  若仓库存在项目专属 skill（如 *-quickref），先加载项目 skill 再加载本 skill。
  未发现 *-quickref 且将改 Java/XML/配置时，须提示用户按 project-quickref.md 创建，默认暂停业务改动。
  不适用：非 Spring 后端（如纯 Android）；仅项目 API 名/业务词（用项目 skill）；
  不改代码的纯概念问答；与本规范无关的纯文档 commit 说明。
---

# Java 开发约束

跨项目通用的 Java / Spring Boot 编写与审查底线。项目专属 skill（若有）负责版本锁定、API 类名、业务词汇；冲突时服从项目 rules / 项目 skill。

**本文件是路由 + 硬门禁。** 细则按需打开下方索引中的单文件；**禁止**为小改通读 `examples*.md` 或整份 `comments-examples.md`。

**触发（跨工具）**：靠 YAML `description` 发现（Spring Boot 后端 / Mapper XML / 配置 / N+1 等场景词）。有项目 skill（如 `*-quickref`）时**先项目、后本 skill**。Cursor 仓内可另配 rules；其他工具主要依赖 description 与 AGENTS.md / 用户指令。

## 默认路径（先走这里）

1. 读 [quick-check.md](quick-check.md)，按「我在改什么」选对应行  
2. **只打开**该行指出的 1～2 个细则文件  
3. 读邻代码，按画像写实现  
4. 交付前：再扫 [quick-check.md](quick-check.md) 超短口令；完整清单见 [pitfalls.md](pitfalls.md)

有项目 skill 时：`Superpowers（见下）→ 项目 skill → 本 skill`。

### 项目 quickref 探测（改代码前）

改本仓库 Java / Mapper XML / Spring 配置前，先找 `*-quickref/SKILL.md`（路径与缺失时动作见 [project-quickref.md](project-quickref.md#agent-未发现-quickref)）。

| 情况 | 动作 |
|------|------|
| 已有项目 quickref | **先读它**，再读本 skill |
| **没有** quickref，且画像不足 | **提示用户**按 [project-quickref.md](project-quickref.md) 创建；默认**暂停业务改动**，可提供模板草稿 |
| 没有 quickref，但 `CLAUDE.md` 已含响应类/分页/禁改 | 说明建议补 quickref；用户未反对时可临时以 CLAUDE 为画像 |

<a id="superpowers-620-衔接"></a>
<a id="Superpowers-衔接"></a>

## Superpowers 衔接

编码约束，不替代流程 skill。项目 rules / 项目 skill **优先于** Superpowers 默认行为。

| 阶段 | Superpowers | 本 skill |
|------|-------------|----------|
| 新功能 / 改业务行为 | `brainstorming` → `writing-plans` | 设计阶段不写实现 |
| 多步实施 | `executing-plans` / `subagent-driven-development` | 写代码时遵守本 skill |
| 写/改 Java | — | 本 skill + 项目 skill |
| 行为变更需回归 | `test-driven-development` 或项目既有流程 | [tests-policy.md](tests-policy.md) |
| 声称完成 | `verification-before-completion` | [pitfalls.md](pitfalls.md) |
| bug / 测试失败 | `systematic-debugging` | 定位后再改代码 |

<a id="局部小改例外与官方-brainstorming-并存"></a>

### 局部小改例外

下列**全部满足**时可跳过 `brainstorming` / `writing-plans`（用户可说「局部小改，直接实现」）：

| 条件 | 说明 |
|------|------|
| 单文件或局部 | 1 个类，或配套 1 个 Mapper XML / 配置 |
| 规模小 | 净增删 **&lt; 50 行**（不含纯格式化） |
| 无新契约 | 不新增 REST、表结构、状态机、跨模块 API |
| 方案已定 | 用户已指明改法，或 bug 根因明确 |

**仍须 brainstorm**：新接口、新表/字段、核心业务规则、多方案取舍、跨模块/子系统。

任一命中则**不是**局部小改，应先 `brainstorming`（通常再 `writing-plans`）：改 REST/RPC/Feign/MQ 契约、改共享 DTO/VO/`api` 模块、改分布式事务/幂等/补偿、改注册中心/网关/配置中心、多模块同时改。

## 适用边界

| 项 | 说明 |
|----|------|
| 默认栈 | Java 8–21、SB 2.x/3.x、可选 Cloud、MyBatis / MP / JPA |
| 不假设 | 包名、基类、统一响应类名、依赖版本 → 以**当前仓库**为准 |
| 改动范围 | 只动任务相关业务模块；不动框架/安全底层与敏感配置（除非用户明确要求） |
| 不直接覆盖 | Dubbo、Vert.x、Quarkus、Micronaut、纯 WebFlux → 跟邻代码与项目规则 |

**版本无关**（换仓仍有效）：分层、注解从短到长、长行换行、步骤分块、注释分级、禁 N+1、禁 Entity 穿透。  
**版本相关**：先完成环境画像再选 API（见 [versions.md](versions.md)）。不得把某仓专属注解/基类/响应类写进本 skill 示例。

### versions 捷径

`CLAUDE.md`、项目 skill 或 `pom`/`gradle` **已写明** JDK、Boot 代际、持久层、分页时：直接采用，只核对邻文件 `import`。  
**仍须全量探测**：陌生仓、依赖刚变、画像与 import 矛盾。画像未明前不生成高版本语法、`jakarta` 或未用的持久层 API。

语法上限速查 → [versions.md#java-版本语法](versions.md#java-版本语法)；分页选型 → [versions.md#分页三分支](versions.md#分页三分支)。

<a id="硬门禁"></a>

## 硬门禁

### 分层（Entity 不穿透）

| 层 | 只做 | 禁止 |
|----|------|------|
| Controller | 校验、调 Service、统一响应 | 业务判断、直调 Mapper、拼复杂 VO |
| Service / Impl | 编排、事务、领域规则 | 解析 HTTP、拼 SQL；Impl 不越层碰 Web 上下文（除非项目已有封装） |
| Mapper / Repository | 数据读写 | 业务分支、事务注解 |
| Convert | 简单字段映射 | 业务 if/else |
| Entity | 表映射 | 作 API 入参/出参 |
| DTO / Param | 写入参 | 仅当出参 |
| VO | 读出参 | 写库对象 |

细则：[architecture.md](architecture.md)。

### 新增类三要素

项目模板优先；否则必须有 `@description`（类职责一句）、`@author`（真人/项目约定，禁 AI 名）、`@since`（`YYYY-MM-DD HH:mm`）。缺一或 description 空 → 不合格。详见 [comments.md#author-取值](comments.md#author-取值)。

### 数据访问 / N+1 / 幂等

- 多表、聚合、动态展示：**一次查询返 VO**；禁 Service 循环查库拼 Map  
- 分页走项目封装；禁全量查再内存截取  
- 禁 `for` / `stream` 内单条查库或远程调用  
- 回调 / MQ / 补偿 → [data-access.md#外部回调与幂等](data-access.md#外部回调与幂等)  
- 金额 → [data-access.md#金额与时间](data-access.md#金额与时间)

### 编码底线（新写片段）

细则见 [coding-patterns.md](coding-patterns.md)。摘要：

- 判空：禁手写 `== null` 判对象；集合用 `CollectionUtils`；字符串用 `StringUtils.isBlank()`；import **跟邻文件**；**不顺手改旧代码判空**  
- Stream：可读优先；禁流内 I/O；链 ≤ 4 步；不用 `peek` 写业务  
- 异常：业务失败抛项目等价 `BusinessException`；禁空 catch / catch 后 `return null`；日志末参带 `Throwable`  
- 写操作：`@Transactional(rollbackFor = Exception.class)`（或项目等价）；注意自调用失效  
- 注入 / 校验包 / 文档注解：跟画像与邻代码，不引入项目没有的风格  
- 注解从短到长；长行固定缩进续行（见 coding-patterns）

### 测试

框架跟 `src/test` 邻类。何时必须验证见 [tests-policy.md](tests-policy.md)：行为/状态/金额/分页口径变更须补测；回调/MQ/幂等须覆盖重复与边界；无基建时交付说明写清手工步骤，不擅自大造测试框架。

### 安全

不绑具体框架。权限在约定入口层；数据范围 Service 复核；日志/导出脱敏；动态 SQL/排序白名单。清单：[security.md](security.md)。

### 中文注释

**强约束**（类三要素见上「硬门禁」）；权威在 [comments.md](comments.md)；占位词/形状/润色按需见 [comments-examples.md](comments-examples.md)。只写业务口径；禁复述代码、AI 套话、机械全覆盖 Javadoc。

<a id="git-提交说明"></a>

### Git 提交说明

`type(scope): 一句话` + 正文 `-` 列表。`type` / `scope` **跟仓库 `git log`**。不提交密钥；未要求不主动 commit。项目 scope/署名见各仓 skill。

## 规则等级

| 等级 | 内容 | 说明 |
|------|------|------|
| **硬门禁** | Entity 不穿透、无 N+1、事务/幂等、敏感配置不乱动、类三要素 | 零容忍：不满足即缺陷 |
| **强约束** | 注释分级、步骤分块、判空与日志、对象边界、YAML 段分割符 | 默认遵守；有稳定邻风先跟邻 |
| **风格建议** | 注解排序、长行换行、YAML `#` 视觉对齐 | 无收益不对旧代码大面积格式化 |

正文用词与上表对齐：勿把「风格建议」写成硬门禁；勿把硬门禁降成「最好遵守」。

## 参考文件索引

| 文件 | 何时读 |
|------|--------|
| [quick-check.md](quick-check.md) | **默认入口**：按改动类型选 1～2 个细则 |
| [versions.md](versions.md) | 画像不明 / 跨仓首次 |
| [architecture.md](architecture.md) | 分层、URL、DTO/VO |
| [data-access.md](data-access.md) | SQL/XML、分页、幂等、金额 |
| [incremental-change.md](incremental-change.md) | 500+ 行大类局部改 |
| [comments.md](comments.md) / [comments-examples.md](comments-examples.md) | 注释规则；占位词/形状/润色（按需） |
| [method-structure.md](method-structure.md) | 长方法编排 |
| [coding-patterns.md](coding-patterns.md) | 判空、Stream、注解、MapStruct |
| [tests-policy.md](tests-policy.md) | 行为变更、bug、回调/金额 |
| [security.md](security.md) | 鉴权、脱敏、排序白名单 |
| [config-yml.md](config-yml.md) | `application*.yml` |
| [migration.md](migration.md) | Boot / Cloud / javax→jakarta 升级 |
| [pitfalls.md](pitfalls.md) | 交付前完整清单 |
| [examples.md](examples.md) | **仅**版本/分页/MapStruct 对照 |
| [examples-method-structure.md](examples-method-structure.md) | **仅**步骤分块完整范本 |
| [project-quickref.md](project-quickref.md) | **新仓接入** / 未发现 `*-quickref` 时：创建指南 + Agent 提示用户 |

## 跨项目复用

复制本目录到项目 `.cursor/skills/` 或全局 `~/.cursor/skills/`。业务画像、模块禁改、响应类名放各仓 **项目 quickref**（推荐）或 `CLAUDE.md`，不要写进本 skill 正文示例。创建步骤 → [project-quickref.md](project-quickref.md)。
