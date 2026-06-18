# 常见陷阱与交付清单

版本相关陷阱的完整对照表见 [versions.md](versions.md)「常见误判」。

## 一、分层与对象

| 陷阱 | 后果 | 对策 |
|------|------|------|
| Entity 作 API 入参 | Mass Assignment、字段被恶意覆盖 | DTO + 校验分组 |
| Entity 作 API 出参 | 泄露内部字段、耦合表结构 | VO |
| Controller 调 Mapper | 难测、事务边界乱 | 经 Service |
| Service 循环调 Mapper | N+1、性能差 | 批量查询 / XML 一次查 VO |
| 在 Convert 里写业务 if | 职责混乱 | 挪到 Service |

## 二、事务与并发

| 陷阱 | 对策 |
|------|------|
| 未指定 `rollbackFor` | 显式 `rollbackFor = Exception.class` |
| 同类内自调用 `@Transactional` 方法 | 拆类或理解 Spring 代理机制 |
| 无幂等的支付/回调接口 | 唯一键 + 状态判断或分布式锁，注释写清 |
| 先改缓存再改库 | 以项目为准，一般先库后缓存，失败可回滚 |

## 三、异常与日志

| 陷阱 | 对策 |
|------|------|
| `catch (Exception e) { return null; }` | 打日志 + 抛业务异常或向上抛 |
| 空 catch | 至少 log.warn/error |
| 业务失败只 `return false` 无说明 | 抛 `BusinessException` 或返回明确错误码 |
| `log.info` 打大对象/敏感信息 | 脱敏；密钥永不落日志 |

## 四、SQL 与性能

| 陷阱 | 对策 |
|------|------|
| `SELECT *` 返大宽表到 Entity 再转 VO | XML 只查 VO 需要的列 |
| 内存分页 | DB 分页（见 data-access.md 三分支） |
| 动态 SQL 拼接用户输入 | `#{}` 参数化 |
| 缺索引的模糊 `%keyword%` | 评估索引、限制场景或走 ES |

## 五、注释反面教材

完整规范、分级表、润色示例见 [comments.md](comments.md)。常见反面：

| 问题 | 示例 |
|------|------|
| 复述代码 | `// 遍历订单列表` |
| 机翻被动 | `// 该订单被用来判断是否可退款` |
| 字段名复述 | `/** companyName 匹配权重 */` |
| 与实现不符 | `// 异步通知下游` ← 实际同步 HTTP |
| 机械全覆盖 | 每个 getter / 一行委托都加多行 Javadoc |
| Swagger 与 Javadoc 双写同义 | Controller 二选一即可 |

## 六、方法结构

| 陷阱 | 后果 | 对策 |
|------|------|------|
| 已触发分块条件却体内无步骤块 | 难跟流程 | 见 [comments.md#步骤分块](comments.md) |
| 步骤注释只有名词无边界 | 看不懂跳过条件 | 写「不通过则不执行」「无数据则清空」等 |
| 块间无空行 | 视觉拥挤 | 每步后空一行 |
| 公开方法 80+ 行无分块 | 难读、难审 | 步骤分块或抽 `validate*` / `build*` |
| 公开方法用单行 `/** ... */` | 缺 `@param`/`@return` | 改多行 Javadoc |
| 私有辅助误用多行 Javadoc | 噪音 | 口径一句话可用单行 `/** ... */` |
| 给一行 Controller 委托写步骤块 | 冗余 | 不必分块，见 method-structure.md |
| 分块用 `----` 装饰线 | 视觉噪点 | 删除，靠步骤注释 + 空行 |
| 复杂逻辑无块内注释 | 难懂口径与分支 | 在 if/循环/计算处补单行 `//` |
| validate 散落在多处 | 改规则易漏 | 写操作入口统一 `validateXxx` |
| 过度抽私有方法 | 跳转多次才懂流程 | 只抽重复或难一眼读懂的块 |
| 机械 `processXxx` 空壳 | 无信息增量 | 内联或改有意义命名 |

详见 [method-structure.md](method-structure.md)。

## 七、格式与排版（版本无关）

以下与 JDK、Spring Boot / Cloud、持久层选型无关；换项目仍适用。

| 陷阱 | 对策 |
|------|------|
| 多注解长短交错、视觉杂乱 | 同一声明处按注解文本从短到长排列（见 [coding-patterns.md](coding-patterns.md#注解排列)） |
| 长行与首参数对齐造成左侧空洞 | 续行固定缩进（见 coding-patterns.md「长行换行」） |
| skill 示例绑定某一仓库的类名或自定义注解 | 示例用占位名；具体注解以邻代码与 versions.md 画像为准 |

## 八、版本与环境（要点）

| 陷阱 | 对策 |
|------|------|
| SB3 仍 `import javax.*` | 改 `jakarta.*`（见 versions.md） |
| Java 8 仓库用 `record`/`var`/`List.of()` | 改 Java 8 等价写法 |
| 无 MP 却生成 `Wrappers` / `BaseMapper` | 改 XML 或原生 MyBatis |
| 混用 Swagger2 与 springdoc 注解 | 跟同模块探测结果 |
| 擅自改 `pom`/`gradle` 版本 | 禁止，除非用户明确要求 |

语法表、持久层分支、注入风格详见 [versions.md](versions.md)。

## 九、AI 生成代码的气味

写代码和写注释时警惕：

- 无意义的 `// 处理异常` 包裹已有 try-catch
- 违背 [comments.md 分级表](comments.md)：给可省略项也加多行 Javadoc
- 三层对称 bullet：`首先 / 其次 / 最后`
- 过度防御：处处 `Optional` 嵌套却无 null 风险
- 过度抽私有方法：`handleXxx` / `processData` 只调用一次
- 引入项目未使用的工具类、设计模式（策略工厂全家桶）
- 擅自升级依赖或改 `pom` 版本
- 类注释 `@description` 为空或写 CRUD 功能列表

**原则**：与邻文件比「多出来的抽象」若无明确收益，删掉。

## 十、交付清单（完整）

### 环境与版本

- [ ] 已读 `versions.md` 并完成环境画像
- [ ] Java 语法与 `pom`/`gradle` 中 `java.version` 一致
- [ ] `javax` / `jakarta` 与 Spring Boot 代际一致（邻文件 import 已核对）
- [ ] 持久层 API 与依赖一致（MP / MyBatis / JPA，非误判）
- [ ] 分页方式与项目一致（PageHelper / MP Page / Pageable）
- [ ] 注入风格与邻类一致

### 结构与 API

- [ ] 类/方法/参数等多注解处已按文本从短到长排列
- [ ] Controller 无业务逻辑、无直调 Mapper/Repository
- [ ] 入参 DTO/Param，出参 VO，无 Entity 穿透
- [ ] URL、响应包装与模块内现有接口一致
- [ ] OpenAPI 注解与项目文档体系一致

### 数据与事务

- [ ] 无循环单条查库
- [ ] 复杂查询 XML 返 VO（如适用）
- [ ] 写操作有 `@Transactional(rollbackFor = Exception.class)`
- [ ] 金额单位、时间类型与项目一致

### 质量

- [ ] 空值/集合判空方式与项目一致
- [ ] 业务异常用项目统一异常类
- [ ] 日志带上下文，异常保留堆栈

### 注释与类头

按 [comments.md](comments.md) 分级表与 30 秒自检：

- [ ] 新增 Java 类含 `@description`（或首行业务说明）+ `@author` + `@since`
- [ ] 「必须」项已补：公开业务方法、ExceptionHandler/Bean、配置/Document/对外 VO 字段等
- [ ] 已触发步骤分块条件的方法：块间空一行，步骤注释含边界
- [ ] 无 AI 套话、机翻味、复述代码、字段名复述
- [ ] Controller：Javadoc 与 Swagger 未双写同义；与邻类风格一致

### 方法结构

- [ ] 公开 Service 方法以编排为主，约 15–40 行或已分块
- [ ] 重复逻辑已抽私有方法；无意义 `processXxx` 空壳已删除
- [ ] 写操作校验集中在 `validate*`（如适用）

### 范围

- [ ] 未修改用户未要求的模块/配置/依赖
- [ ] 未提交密钥、密码、token
