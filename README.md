# shaoeric-skills

AI 应用技术方案技能集 — 包含 `tech-proposal` 等实用技能，适用于 **Claude Code**。

## 包含的技能

| 技能 | 说明 | 触发方式 |
|------|------|----------|
| **tech-proposal** | AI 应用技术方案架构师。从需求对齐 → 技术调研 → 方案生成 → 审查 → 迭代优化，全流程产出可落地的技术方案报告。 | `/tech-proposal` 或自然语言描述 |

## 环境要求

- **Claude Code** CLI 已安装（[官方安装指南](https://docs.anthropic.com/en/docs/claude-code/overview)）

## 安装步骤（推荐：直接从 GitHub 安装，无需克隆）

### 1. 添加 Marketplace

在 Claude Code 会话中执行：

```
/plugin marketplace add shaoeric/SKILLS
```

或者使用完整 URL：

```
/plugin marketplace add https://github.com/shaoeric/SKILLS
```

### 2. 安装插件

```
/plugin install tech-proposal
```

### 3. 验证安装

```
/plugin list
```

如果看到 `tech-proposal` 出现在列表中，说明安装成功。

> **备选方案：本地安装**
>
> 如果你需要从本地路径安装（例如离线环境），可以先 clone 仓库再注册：
> ```bash
> git clone https://github.com/shaoeric/SKILLS.git
> ```
> 然后在 Claude Code 中执行：
> ```
> /plugin marketplace add ./SKILLS
> /plugin install tech-proposal
> ```

## 使用方式

### 通过命令触发

在 Claude Code 对话中直接输入：

```
/tech-proposal
```

### 通过自然语言触发

直接描述你的需求，技能会自动激活：

```
帮我设计一个智能客服系统的技术方案
```

**触发关键词**：技术方案、架构设计、方案设计、tech proposal、solution design、architecture design

## 工作流程

```
需求对齐 → 技术调研 → 方案生成 → 方案审查 → 迭代优化
```

1. **Phase 0 — 需求对齐**：Agent 通过选择题逐轮提问，明确你的技术需求
2. **Phase 1 — 深度技术调研**：针对需求进行技术选型和调研
3. **Phase 2 — 方案生成**：生成结构化的技术方案文档
4. **Phase 3 — 方案审查**：多维度审查方案质量
5. **Phase 4 — 迭代优化**：根据审查意见持续改进方案

## 项目结构

```
SKILLS/
├── .claude-plugin/
│   └── marketplace.json          # Marketplace 配置
├── plugins/
│   └── tech-proposal/            # tech-proposal 插件
│       ├── .claude-plugin/
│       │   └── plugin.json       # 插件元信息
│       └── skills/
│           └── tech-proposal/
│               ├── SKILL.md      # 技能定义
│               ├── agents/       # Agent 配置
│               └── references/   # 参考模板
└── .gitignore
```

## 常见问题

### Q: 提示 "claude 命令未找到"？

请确保已安装 Claude Code CLI，参考 [官方文档](https://docs.anthropic.com/en/docs/claude-code/overview)。

### Q: 安装插件时报错？

确认 marketplace 已正确注册，在 Claude Code 中执行 `/plugin marketplace list` 查看已注册的市场。

### Q: 如何更新插件？

通过 GitHub 安装的 marketplace 会自动获取最新版本。你也可以手动刷新：

```
/plugin marketplace update shaoeric/SKILLS
```

然后重新安装插件即可获取最新版本。

## 许可

MIT License
