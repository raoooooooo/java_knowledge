# .claude 目录说明

本目录用于存放 Claude Code 相关的自定义配置和技能文件。

## 目录结构

```
.claude/
├── README.md                    # 本说明文件
├── settings.json                # Claude Code 项目级配置
├── skills/                      # 自定义技能目录
│   └── *.md                     # 技能定义文件
├── scheduled_tasks.json         # 定时任务配置
└── memories/                    # 记忆文件目录
```

## 说明

- 本目录内容会提交到 Git 仓库，用于团队共享 Claude Code 技能和配置
- 敏感信息（如 API Key、Token 等）不要提交到仓库
