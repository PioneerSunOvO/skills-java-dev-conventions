# java-dev-conventions

跨项目通用的 Java 企业级后端（Spring Boot / Spring Cloud、MyBatis/MyBatis-Plus/JPA）编写与审查 Skill。

供 **Cursor、Claude Code、Codex** 等 Agent 引用，也可作为人工 Code Review 检查清单。本仓库为规范正本；业务项目通过安装或软链引用，**不在此仓库内维护具体业务代码**。

## 能力概览

- **分层**：Controller / Service / Mapper；Param/DTO 入参、VO 出参
- **环境自适应**：Java 8–21、SB2/SB3、Spring Cloud、MP/MyBatis/JPA（见 `versions.md`）
- **注释**：Java 五层 + Mapper XML 方法级注释 + YAML 段落分割符；统一自然中文、去 AI/机翻味
- **交付**：`pitfalls.md` 完整自检清单；`examples.md` 精简对照示例

## 安装

### 从 GitHub（推荐）

```bash
npx skills add PioneerSunOvO/skills-java-dev-conventions -g --agent claude-code cursor -y --copy
```

### 手动复制 / 软链

将整个 `java-dev-conventions` 目录放到目标机器的 skills 目录，例如：

| 工具 | 全局路径 | 项目路径 |
|------|----------|----------|
| Cursor | `~/.cursor/skills/java-dev-conventions/` | `.cursor/skills/java-dev-conventions/` |
| Claude Code | `~/.claude/skills/java-dev-conventions/` | `.claude/skills/java-dev-conventions/` |
| Codex / 其他 | `~/.agents/skills/java-dev-conventions/` | 按工具约定 |

业务项目可用软链指向全局目录，避免在业务仓库内复制多份。

## 使用

在 Agent 对话中引用：

```
@java-dev-conventions
```

建议执行顺序：

- 先读目标项目 `CLAUDE.md`、`AGENTS.md` 或 `.cursor/rules`
- 若有项目专属 skill（如 `*-quickref`），再叠加阅读
- 最后按本 skill 的 `SKILL.md` 执行顺序编写或审查代码

## 与项目规则的关系

| 优先级 | 来源 |
|--------|------|
| 最高 | 目标仓库 `CLAUDE.md`、`AGENTS.md`、`.cursor/rules` |
| 次之 | 项目专属 skill（包名、统一响应类、Git scope 等） |
| 再次 | 本 skill |

**本 skill 不写死**某一项目的包名、API 类名或配置路径；此类约定由各业务仓库自行维护。

## 文件结构

| 文件 | 内容 |
|------|------|
| `SKILL.md` | 入口、执行顺序、分层速查、Git 通用说明 |
| `versions.md` | 环境探测、Spring Cloud、持久层分支 |
| `architecture.md` | 分层、对象、URL、测试、定时任务、安全 |
| `comments.md` | 类注释与语气（唯一权威） |
| `method-structure.md` | 方法步骤分块 |
| `data-access.md` | SQL/XML 与注释 |
| `config-yml.md` | YAML 分割符与注释 |
| `coding-patterns.md` | 编码习惯速查 |
| `pitfalls.md` | 陷阱与完整交付清单 |
| `examples.md` | 精简对照示例 |

## 维护说明

- 通用规范变更：在本仓库提 PR 或 push 到 `main`
- 业务项目同步：`git pull` 全局安装目录，或更新软链目标
- 勿将单一业务项目的类名、表名、配置路径写入本 skill 正文

## License

MIT
