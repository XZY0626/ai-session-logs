# AI 对话日志仓库

## 📌 用途

本仓库记录所有 AI 助手与用户 XZY0626 的对话执行日志，确保所有 AI 行为**有迹可循、可审计、可回溯**。

## 📁 目录结构

```
ai-session-logs/
├── README.md                          # 本文件
├── INDEX.md                           # 日志总索引（按AI和时间分类）
└── logs/
    ├── workbuddy/                     # WorkBuddy 的日志（当前主力AI）
    │   └── 2026-03/
    │       ├── 2026-03-13-openclaw-upgrade-v2026311.md
    │       ├── 2026-03-13-openclaw-frontend-v5-minimax.md
    │       ├── 2026-03-13-openclaw-network-tailscale-https.md
    │       ├── 2026-03-13-workbench-reorganize.md
    │       ├── 20260313_GitHub凭证泄露安全事件修复.md
    │       └── 20260313_MiniMax模型接入配置.md
    └── xiaoyue/                       # 小跃 (StepFun Desktop) 的日志（已退役）
        └── 2026-03/
            ├── 20260310_飞书机器人与openclaw全面配置.md
            ├── 20260311_补全openrouter模型.md
            ├── 20260311_规则文件建立与日志体系.md
            ├── 20260311_开机自启与飞书多模型切换.md
            ├── 20260311_openrouter恢复与step35flash接入.md
            ├── 20260312_GitHub_Secret_Scanning安全告警修复.md
            └── 20260312_OpenRouter地区封锁修复与三端联调.md
```

> 日志目录按月份新增子文件夹，格式为 `YYYY-MM/`，禁止直接放在 AI 目录根目录下。

## 🤖 AI 记录说明

### WorkBuddy（当前主力，2026-03-13 起）
腾讯云 WorkBuddy，目前负责所有系统运维、OpenClaw 管理、代码开发、规则维护等任务。日志存放于 `logs/workbuddy/`。

### 小跃（已退役，2026-03-10 至 2026-03-12）
StepFun Desktop AI 助手，负责 OpenClaw 早期配置、飞书机器人搭建、模型接入等工作，共产生 7 篇日志。日志存放于 `logs/xiaoyue/`，作为历史记录保留。

## 📋 日志格式规范

每条日志包含以下章节：

| 章节 | 内容 |
|------|------|
| 基本信息 | AI名称、日期、对话主题 |
| 执行摘要 | 3-5句话概括本次对话完成了什么 |
| 操作记录 | 逐条记录文件操作（类型/路径/结果） |
| 文件变更清单 | 新增/修改/删除的文件列表 |
| 安全审查 | 凭证泄露检查、危险操作检查、C盘操作检查 |
| 待办/遗留问题 | 未完成事项 |
| GitHub提交信息 | 仓库、Commit message |

详细格式规范见 [AI_RULES.md L2.2](https://github.com/XZY0626/ai-rules/blob/main/AI_RULES.md)

## 🔍 如何查找日志

1. 查看 `INDEX.md` 获取完整日志索引（按AI分类 + 按时间线）
2. 按 AI 名称进入对应目录（`workbuddy/` 或 `xiaoyue/`）
3. 按年月（`YYYY-MM/`）查找具体日志文件

## 🔒 安全与脱敏说明

所有上传到本仓库的日志均已经过安全审查和脱敏处理：

- **API Key / Token**：保留前8位+末4位，中间用 `...` 替代
- **密码**：完全移除，替换为 `<REDACTED>`
- **虚拟机 IP**：内网地址（192.168.x.x）保留，公网地址脱敏
- **Tailscale 域名**：脱敏为 `[TAILSCALE_HOSTNAME].[TAILNET_DOMAIN]`
- 脱敏规范详见 [AI_RULES.md L0.3](https://github.com/XZY0626/ai-rules/blob/main/AI_RULES.md)

## 📋 关联仓库

| 仓库 | 用途 |
|------|------|
| [ai-rules](https://github.com/XZY0626/ai-rules) | AI执行规则文件 |
| [ai-session-logs](https://github.com/XZY0626/ai-session-logs) | 本仓库 — AI对话日志 |
| [openclaw-config](https://github.com/XZY0626/openclaw-config) | OpenClaw配置管理 |
