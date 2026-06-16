# 常见陷阱与交付清单

版本相关见 [versions.md](versions.md)。注释反面教材与语气见 [comments.md](comments.md)。

## 一、分层与对象

| 陷阱 | 后果 | 对策 |
|------|------|------|
| Entity 作 API 入参 | Mass Assignment | DTO + 校验分组 |
| Entity 作 API 出参 | 字段泄露 | VO |
| Controller 调 Mapper | 事务乱、难测 | 经 Service |
| Service 循环调 Mapper | N+1 | 批量 / XML 返 VO |
| Convert 里写业务 if | 职责乱 | 挪到 Service |
| Feign 接口写业务逻辑 | 难测、难复用 | 逻辑放 Service |

## 二、事务与并发

| 陷阱 | 对策 |
|------|------|
| 未指定 `rollbackFor` | `rollbackFor = Exception.class` |
| 同类自调用 `@Transactional` | 拆类或理解代理 |
| 无幂等回调/支付接口 | 唯一键 + 状态判断；注释写清 |
| 先改缓存再改库 | 一般先库后缓存 |

## 三、异常与日志

| 陷阱 | 对策 |
|------|------|
| 空 catch / catch 后 return null | log + 抛业务异常 |
| 业务失败只 return false | 抛异常或明确错误码 |
| 日志打敏感信息 | 脱敏 |

## 四、SQL 与性能

| 陷阱 | 对策 |
|------|------|
| `SELECT *` 大宽表再转 VO | XML 只查 VO 列 |
| 内存分页 | DB 分页 |
| 拼接用户输入进 SQL | `#{}` |
| XML 无方法注释 | 标签正上方写口径 |

<a id="五注释反面教材"></a>

## 五、注释反面教材

见 [comments.md](comments.md)、[config-yml.md](config-yml.md)、[data-access.md](data-access.md)。典型差例：

- `// 遍历列表`、`<!-- 关联表 -->`、`# enable-ansi 配置`
- YAML 无 start/end 分割符；`本节 start` 无具体简述
- `值得注意的是…`、`该配置项用于…`

<a id="六方法结构"></a>

## 六、方法结构

| 陷阱 | 对策 |
|------|------|
| 只有 Javadoc、无步骤分块 | 补 `//` 总注释 + 块间空行 |
| 步骤注释只有名词 | 补动词与边界 |
| 80+ 行无分块 | 分块或抽 `validate*` |

## 七、格式与排版

| 陷阱 | 对策 |
|------|------|
| 多注解长短交错 | 从短到长排列（coding-patterns.md） |
| 长行与首参数对齐空洞 | 固定缩进续行 |

## 八、版本与环境

| 陷阱 | 对策 |
|------|------|
| 无 Cloud 却生成 Feign | 先探测 spring-cloud-dependencies |
| SB2 用 jakarta | 保持 javax |
| Java 8 用 record/var | 降级写法 |
| 无 MP 却用 Wrappers | 改 XML |
| 擅自升依赖 | 禁止 |

## 九、AI 生成代码气味

- 千篇一律 Javadoc、首先/其次/最后三段式
- 过度 Optional 嵌套、无意义 `processXxx`
- 擅自引入项目没有的设计模式或依赖
- 类注释写 CRUD 列表

<a id="十交付清单完整"></a>

## 十、交付清单（完整）

### 环境与版本

- [ ] 已完成环境画像（Java、SB 代际、**是否 Spring Cloud**、持久层）
- [ ] 语法、javax/jakarta、注入、分页与项目一致
- [ ] Cloud 项目：Feign/Gateway 用法与邻模块一致

### 结构与 API

- [ ] 多注解从短到长排列
- [ ] Controller 无业务、无直调 Mapper/Repository
- [ ] 入参 DTO/Param，出参 VO，无 Entity 穿透
- [ ] URL、响应包装与邻接口一致

### 数据与事务

- [ ] 无循环单条查库
- [ ] 复杂查询 XML 返 VO（如适用）
- [ ] 写操作有 `@Transactional(rollbackFor = Exception.class)`

### 质量

- [ ] 判空、业务异常、日志与项目一致

### 注释

- [ ] 新增类注释见 [comments.md](comments.md)
- [ ] 方法 Javadoc、步骤分块、块内注释达标
- [ ] Mapper XML / YAML 注释与分割符达标
- [ ] 自然中文，无 AI/机翻味

### 方法结构

- [ ] 公开 Service 方法编排为主，已分块或行数合理

### 范围

- [ ] 未改敏感配置、未升依赖、未动用户未要求模块
- [ ] 未提交密钥

### 提交说明（若需提交）

- [ ] 标题与正文符合 [SKILL.md#git-提交说明](SKILL.md#git-提交说明)；正文用 `-` 列表
