# 30 秒按改动类型自检

只想快速过一遍时，从下表按改动类型选对应行；详细规则再回主文档。

| 我在改什么 | 先看 | 至少确认 |
|-----------|------|----------|
| 新增 / 改 Java 业务方法 | `SKILL.md`、`architecture.md` | 无 Entity 穿透、分层没乱、必要注释已补 |
| 500+ 行大类局部修改 | `incremental-change.md` | 只改相关片段，不顺手重构整类 |
| 判空 / Stream / MapStruct | `coding-patterns.md` | 新写片段跟邻代码；无 N+1；无炫技 Stream |
| SQL / Mapper XML / 分页 | `data-access.md` | 无内存分页、无循环查库、复杂查询一次返 VO |
| 回调 / MQ / 幂等 / 状态流转 | `data-access.md`、`tests-policy.md` | 幂等键明确、终态短路、重复调用有验证 |
| 金额 / 税费 / 汇总 | `data-access.md`、`tests-policy.md` | `BigDecimal`、单位明确、`compareTo`、舍入规则清楚 |
| 鉴权 / 数据范围 / 脱敏 | `security.md` | 不信任前端 userId；日志不泄露敏感信息 |
| 注释 / 类头 / 长方法 | `comments.md`、`method-structure.md` | 类三要素、步骤分块；只写业务口径；无 AI/机翻味；无“遍历/封装/查库”噪音 |
| YAML 配置注释 | `config-yml.md` | 段分割符；用途/单位/开关差异；无「本节用于…」「下面是…」 |
| 交付前声称“改好了” | `pitfalls.md`、`tests-policy.md` | 有验证证据、风险说清、未越权改依赖/敏感配置 |

## 超短口令

```text
1. 画像明确了吗？
2. 有没有越层、Entity 穿透、N+1？
3. 这次改动会不会改变行为？会的话证据在哪？
4. 有幂等/金额/权限/敏感日志风险吗？
5. 注释和实现还一致吗？
```
