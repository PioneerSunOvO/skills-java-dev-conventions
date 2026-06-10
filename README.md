# java-dev-conventions

国内企业级 Java 后端（Spring Boot 2.x/3.x、MyBatis/MyBatis-Plus/JPA）编写与审查约束 Skill，供 Claude Code、Cursor 等 Agent 使用。

## 能力概览

- 分层边界：Controller / Service / Mapper、Param/DTO 入参、VO 出参
- 版本无关核心：注解从短到长排列、长行换行、步骤分块、五层中文注释
- 版本自适应：通过 `versions.md` 探测 JDK、javax/jakarta、持久层与文档注解体系
- 交付自检：`pitfalls.md` 清单

## 安装

### 从 GitHub（推荐）

```bash
npx skills add <你的GitHub用户名>/skills-java-dev-conventions -g --agent claude-code -y --copy
```

仅安装到 Claude Code 全局目录（`~/.claude/skills/`）。

同时支持 Cursor：

```bash
npx skills add <你的GitHub用户名>/skills-java-dev-conventions -g --agent claude-code cursor -y --copy
```

### 从本地路径（开发调试）

```bash
npx skills add ./skills-java-dev-conventions -g --agent claude-code -y --copy
```

### 手动复制

将整个目录复制到：

| 工具 | 全局路径 | 项目路径 |
|------|----------|----------|
| Claude Code | `~/.claude/skills/java-dev-conventions/` | `.claude/skills/java-dev-conventions/` |
| Cursor | `~/.cursor/skills/java-dev-conventions/` | `.cursor/skills/java-dev-conventions/` |

## 使用

在 Claude Code 中：

```
/java-dev-conventions
```

或自然语言：

```
使用 java-dev-conventions skill，按规范新增 OrderController
```

## 文件结构

```
SKILL.md              # 入口（必需）
versions.md           # 环境探测与版本分支
architecture.md       # 分层与 URL 约定
coding-patterns.md    # 编码习惯、注解排列、长行换行
comments.md           # 五层注释规范
method-structure.md   # 方法步骤分块
data-access.md        # SQL/XML 与分页
pitfalls.md           # 陷阱与交付清单
examples.md           # 示例
```

## 与项目规则的关系

本 Skill 保持跨项目通用。目标仓库的 `CLAUDE.md`、`AGENTS.md`、`.cursor/rules` 中的模块禁改、版本锁定、注入风格等项目约定**优先于**本 Skill。

## License

MIT
