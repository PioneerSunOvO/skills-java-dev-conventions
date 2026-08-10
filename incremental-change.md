# 存量大文件增量修改

与 [SKILL.md](SKILL.md) 配套。适用于**已有、运行中的代码库**里对超大 `*ServiceImpl`、冗长 Controller 等做**局部改动**——不限于某一业务域或某一框架脚手架。

## 何时读本文

- 目标类 **500+ 行**，或单文件承担多个子流程
- 任务只是修 bug、补字段、加一条分支，**不要求**整类重构
- 单体（Spring Boot）与微服务（Spring Cloud + Feign）均适用：原则相同，跨服务调用注意 DTO 边界

## 原则

| 原则 | 说明 |
|------|------|
| **只动相关范围** | 改哪个方法、哪段逻辑，就只编辑该处及直接依赖的私有方法 / Mapper |
| **不顺手重构** | 除非用户明确要求，不做「顺便重命名全类」「顺便统一判空风格」 |
| **新逻辑优先外提** | 新增校验 → `validate*`；组装 → `build*` / `resolve*`；见 [method-structure.md](method-structure.md) |
| **先定位再改** | 行为异常时先走 Superpowers `systematic-debugging`，禁止边通读千行边大改 |
| **注释写边界** | 涉及状态流转、外部回调、金额、幂等时，在方法 Javadoc 或步骤块写明前置状态与跳过条件 |

## 推荐做法

```
1. 用 IDE / codegraph / grep 定位入口方法与调用链
2. 只读相关方法 ± 一层私有方法，不强制读完整个类
3. 新代码风格跟同文件邻段（判空、日志、Wrapper 链）→ [coding-patterns.md#多工具库并存](coding-patterns.md#多工具库并存)
4. 若新增逻辑 ≥15 行或多阶段，对**新方法**做步骤分块，不要求重写旧方法
5. 交付前只跑与改动相关的验证（单测 / 接口 / 手工场景）
```

## 何时应抽方法或拆类

| 信号 | 建议 |
|------|------|
| 同一逻辑在类内复制 2 次以上 | 抽私有方法 |
| 单个 `public` 方法超过 80 行且无分块 | **存量重构信号**（新代码视为不合格，见 method-structure）；在本类内拆 `validate*` / `build*`，或拆新 Service（需设计） |
| 类内混杂多个无关子域（如「订单 + 对账 + 导出」） | 与用户确认是否拆 Service；**不**在单次小改中静默拆分 |
| 仅新增 10 行校验 | 内联或一个 `validateXxx` 即可 |

## 微服务场景补充

检测到 Spring Cloud（见 [versions.md#Spring Cloud 分支](versions.md#spring-cloud-分支)）时：

- 本服务内仍按 Controller → Service → Mapper；**不在 Controller 调 Feign**
- 跨服务改动涉及 **api 模块 DTO** 与提供方 Service 契约，通常**不满足** [SKILL.md 局部小改例外](SKILL.md#局部小改例外与官方-brainstorming-并存)，应先 `brainstorming`
- 分布式幂等、最终一致见 [data-access.md#外部回调与幂等](data-access.md#外部回调与幂等)

## 常见陷阱

| 陷阱 | 对策 |
|------|------|
| 为「统一风格」整文件改判空 / 改 Stream | 禁止；只改任务相关片段 |
| 在巨型方法中间插入大段新逻辑 | 抽私有方法，公开方法保持编排 |
| 改 A 功能却动 B 功能的私有方法 | 新增方法或复制最小片段，避免回归 |
| 无测试的大类「顺手优化」 | 无用户授权不做 |

更多见 [pitfalls.md](pitfalls.md)。
