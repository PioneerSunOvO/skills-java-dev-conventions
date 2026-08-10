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
| 新增 / 改 Java 业务方法 | `architecture.md`；注释用 `comments.md` | [ ] 无 Entity 穿透<br>[ ] 分层未乱<br>[ ] 必要注释已补 |
| 500+ 行大类局部修改 | `incremental-change.md` | [ ] 只改相关片段<br>[ ] 未顺手重构整类 |
| 判空 / Stream / MapStruct | `coding-patterns.md` | [ ] 新写判空跟邻文件 import<br>[ ] 无循环内单条查库<br>[ ] Stream 链 ≤ 4 步<br>[ ] 未批量改旧判空 |
| SQL / Mapper XML / 分页 | `data-access.md`；分页选型见 `versions.md` | [ ] 无内存分页<br>[ ] 无循环查库<br>[ ] 复杂查询一次返 VO |
| 回调 / MQ / 幂等 / 状态流转 | `data-access.md`、`tests-policy.md` | [ ] 幂等键明确<br>[ ] 终态短路<br>[ ] 重复调用有验证 |
| 金额 / 税费 / 汇总 | `data-access.md`、`tests-policy.md` | [ ] `BigDecimal`<br>[ ] 单位明确<br>[ ] `compareTo` / 舍入说清 |
| 鉴权 / 数据范围 / 脱敏 | `security.md` | [ ] 不信任前端 userId<br>[ ] 日志未泄露敏感信息 |
| 注释 / 类头 / 长方法 | `comments.md`、`method-structure.md`（去 AI 味/形状按需再开 `comments-examples`） | [ ] 类三要素<br>[ ] 步骤分块（如触发）<br>[ ] 只写业务口径<br>[ ] 无 AI/机翻味 |
| YAML 配置注释 | `config-yml.md` | [ ] 有段分割符<br>[ ] 用途/单位/开关已写<br>[ ] 无「本节用于…」「下面是…」 |
| 画像不明 / 换仓首次 | `versions.md`（含 [java-版本语法](versions.md#java-版本语法)、[分页三分支](versions.md#分页三分支)） | [ ] JDK / Boot 代际与可用语法<br>[ ] javax/jakarta<br>[ ] 持久层与注入风格<br>[ ] 分页选型 |
| 交付前声称「改好了」 | `pitfalls.md`、`tests-policy.md` | [ ] 有验证证据<br>[ ] 风险说清<br>[ ] 未越权改依赖/敏感配置 |

硬门禁摘要（Entity / N+1 / 类三要素 / 局部小改）见 [SKILL.md](SKILL.md#硬门禁)。

## 超短口令

```text
1. 画像明确了吗？（项目 quickref / CLAUDE / versions 捷径；无 quickref 时是否已提示用户？）
2. 有没有越层、Entity 穿透、N+1？
3. 这次改动会不会改变行为？会的话证据在哪？
4. 有幂等/金额/权限/敏感日志风险吗？
5. 注释和实现还一致吗？有没有 AI/机翻味？
6. 只读了本表对应的 1～2 个细则，没有通读 examples 吧？
```
