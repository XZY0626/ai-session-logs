# AI 对话日志仓库

## 📌 用途

本仓库记录所有 AI 助手与用户 XZY0626 的对话执行日志，确保所有 AI 行为**有迹可循**。

## 📁 目录结构

```
ai-session-logs/
├── README.md                    # 本文件
├── INDEX.md                     # 日志总索引（按AI和时间分类）
└── logs/
    ├── xiaoyue/                 # 小跃 (StepFun Desktop) 的日志
    │   └── 2026-03/
    │       ├── 20260310_飞书机器人与openclaw配置.md
    │       └── 20260311_规则文件建立与模型修复.md
    ├── openclaw/                # OpenClaw Agent 的日志
    │   └── 2026-03/
    │       └── ...
    ├── claude/                  # Claude 的日志
    ├── gpt/                     # GPT 的日志
    └── other/                   # 其他AI的日志
```

## 📋 日志格式

每条日志包含：
- **AI标识**: 哪个AI、哪个对话窗口
- **执行摘要**: 本次对话做了什么
- **操作记录**: 逐条记录文件操作
- **文件变更清单**: 新增/修改/删除了哪些文件
- **安全审查**: 凭证泄露检查、危险操作检查
- **GitHub提交**: 关联的commit和版本号

详细格式规范见 [AI_RULES.md L2.2](https://github.com/XZY0626/ai-rules/blob/main/AI_RULES.md)

## 🔍 如何查找日志

1. 查看 `INDEX.md` 获取完整日志索引
2. 按AI名称进入对应目录
3. 按年月查找具体日志

## 🔒 安全说明

- 所有日志中的凭证信息已脱敏（前8位+末4位）
- 密码完全移除，替换为 `<REDACTED>`
- 内网IP地址保留，公网IP脱敏
