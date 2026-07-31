# 升级与迁移

适用于 **Java / Spring Boot / Spring Cloud / 文档体系 / JSON 组件**升级任务。本文只给**跨项目通用**原则；具体版本号、BOM、组件名单以仓库 `pom` / `gradle`、`CLAUDE.md`、项目 skill 为准。

## 何时读本文

- Java 8 → 17 / 21
- Spring Boot 2.x → 3.x
- Spring Cloud 旧版 → 新版 BOM
- Swagger2 → springdoc
- `javax.*` → `jakarta.*`
- JSON / HTTP / 注册中心 / 熔断组件专项升级

## 升级任务默认不算“局部小改”

出现下列任一项，先 `brainstorming`，通常还需要 `writing-plans`：

- 需要调整多个模块依赖或 BOM
- 需要批量替换注解、包名、配置项
- 需要同时改业务代码与部署配置
- 需要兼容双版本运行或灰度发布

## Java 8 → 17 / 21

| 关注点 | 说明 |
|--------|------|
| 语法上限 | `record`、`var`、`switch` 表达式、`Stream.toList()` 等仅在目标版本允许后再用 |
| 第三方依赖 | 先看旧依赖是否支持新 JDK；不只改 `java.version` |
| 编译插件 | Maven/Gradle 的 `source` / `target` / toolchain 一并确认 |
| 测试 | 关注反射、字节码、序列化、时间 API 差异 |

## Spring Boot 2.x → 3.x

| 关注点 | 说明 |
|--------|------|
| `javax.*` → `jakarta.*` | Validation、Servlet、JPA、注解包名统一迁移 |
| 最低 Java | Boot 3 通常要求 Java 17+ |
| 文档体系 | Swagger2 / springfox 常需迁到 springdoc |
| 安全与过滤器 | 过滤器链、Security 配置类按邻代码和官方迁移指南调整 |
| 测试基类 | `@SpringBootTest`、MockMvc、切片测试兼容性一并验证 |

## Spring Cloud 升级

| 关注点 | 说明 |
|--------|------|
| BOM 对齐 | Cloud 版本跟 Boot 版本成套升级，不单独升某个 starter |
| OpenFeign | 共享 DTO / api 模块是否兼容；超时、拦截器、降级策略是否仍生效 |
| Gateway | 路由配置、过滤器顺序、鉴权头透传 |
| 注册/配置中心 | Nacos / Eureka / Config 客户端版本与服务端兼容性 |
| 熔断/限流 | Sentinel / Resilience4j / LoadBalancer 配置项是否变更 |

## Swagger2 → springdoc

| 旧 | 新 |
|----|----|
| `@Api` / `@ApiOperation` / `@ApiModelProperty` | `@Tag` / `@Operation` / `@Schema` |

原则：

- 先确认仓库是否真的切到 springdoc，**不要混用**
- Controller Javadoc 与 API 注解避免双写同义内容
- 配置与 starter 一起迁，不只改注解

## JSON / HTTP / 其他组件

| 类型 | 迁移原则 |
|------|----------|
| JSON（Fastjson / Jackson / Gson） | 一次只保留一套主序列化体系；避免并行新增第二套注解 |
| HTTP（RestTemplate / WebClient / OkHttp / Feign） | 跟项目现有主路径；升级时先查超时、拦截器、错误处理 |
| 注册中心 / 配置中心 | 跟 BOM 统一升级；本地配置、测试环境、部署脚本一起核对 |

## 迁移任务的交付口径

迁移类任务除常规自检外，还应补充：

- 改了哪些依赖 / BOM / 插件
- 哪些包名或注解体系发生迁移
- 哪些测试或联调场景已验证
- 是否存在待灰度 / 待回归项

## 升级任务交付清单

### 依赖与构建

- [ ] Java 版本、Spring Boot / Spring Cloud BOM、关键 starter 已成套调整
- [ ] Maven / Gradle 编译插件、测试插件、镜像仓库配置已核对
- [ ] 未在同一次任务里混入无关依赖升级

### 代码与配置

- [ ] `javax.*` / `jakarta.*`、Swagger2 / springdoc、老旧 API 的替换范围已确认
- [ ] 配置文件、环境变量、注册中心 / 配置中心键名改动已同步
- [ ] 共享 DTO、Feign、Gateway、过滤器、拦截器等跨模块影响已评估

### 验证

- [ ] 单测 / 集成测试 / 启动验证至少覆盖关键入口
- [ ] 关键链路已联调：HTTP、Feign、MQ、回调、鉴权、配置加载
- [ ] 升级后日志、序列化、时间格式、异常处理无明显回归

### 交付说明

- [ ] 提交说明写清升级范围、兼容性影响、已验证场景
- [ ] 如需灰度 / 回滚 / 分批发布，已在说明中写清
