# ClaudeUtil

> 个人长期维护的 Claude Code 使用工具集 —— 规范、规则、提示词模板的积累与沉淀。

## 这是什么

这个仓库是我在日常使用 [Claude Code](https://claude.com/claude-code) 过程中，持续总结和整理的辅助工具集。

核心目标：**让 Claude 按照团队/个人的编码规范来写代码**，而不是用它的"默认风格"。

## 仓库结构

```
ClaudeUtil/
├── CLAUDE.md                      Claude Code 全局规则文件（可直接复制到 ~/.claude/CLAUDE.md）
├── SmartAdmin-Java规范-V3.0.md     完整参考文档（含正反例、详细解释）
└── README.md
```

## 文件说明

### CLAUDE.md —— Claude Code 指令规则

这是 Claude Code 的核心配置文件。放到项目根目录或 `~/.claude/CLAUDE.md`（全局生效），Claude 在写代码时会自动遵守其中的规则。

当前包含的规则：

| 分类 | 规则内容 |
|------|----------|
| **通用** | 使用中文回答；Vue/React 接口 URL 添加时间戳防缓存 |
| **Java 项目结构** | 项目命名、目录结构、方法参数限制 |
| **MVC 四层架构** | Controller / Service / Manager / DAO 各层职责与禁止事项 |
| **JavaBean 规范** | Entity / Form / VO / DTO / BO 命名与约束 |
| **数据库规范** | 表命名、必备字段、枚举注释要求 |

### SmartAdmin-Java规范-V3.0.md —— 完整参考文档

这是 [SmartAdmin](https://smartadmin.vip) Java 后端开发规范的完整版本，包含：
- 每条规则的详细解释和背景说明
- 正例与反例的代码对比
- Manager 层的设计思路和解决的问题
- 数据库建表完整示例

适合**学习和理解**规范背后的原因，而 `CLAUDE.md` 是其精炼的**指令化**版本。

## 使用方式

### 方式一：全局生效（推荐）

将 `CLAUDE.md` 的内容复制到全局配置中：

```bash
cp CLAUDE.md ~/.claude/CLAUDE.md
```

之后在任何项目中打开 Claude Code，都会自动遵守这些规范。

### 方式二：项目级生效

将 `CLAUDE.md` 复制到你的 Java 项目根目录：

```bash
cp CLAUDE.md /path/to/your-java-project/CLAUDE.md
```

仅在该项目中生效。

## 后续计划

这个仓库会长期更新，持续积累以下内容：

- [ ] 前端（Vue/React）开发规范
- [ ] Git 提交信息规范
- [ ] API 接口设计规范
- [ ] 代码审查 Checklist
- [ ] 常用提示词模板
- [ ] Claude Code 使用技巧与最佳实践

## 参考来源

- [SmartAdmin 官方文档](https://smartadmin.vip/views/doc/standard/back.html)
- [SmartAdmin Manager 层说明](https://smartadmin.vip/views/doc/back/ManagerLayer.html)
- [阿里巴巴 Java 开发手册](https://github.com/alibaba/p3c)
