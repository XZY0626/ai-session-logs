# [2026-03-14] 龙虾成长计划 — 飞书权限修复 + workspace 初始化 + 技能部署

**时间**：2026-03-14 23:15 (CST)
**执行者**：WorkBuddy
**触发**：用户请求配置龙虾飞书能力 + 移植 WorkBuddy 核心能力

---

## 任务描述

1. 修复飞书 `contact:contact.base:readonly` 权限缺失问题（导致龙虾收消息无回复）
2. 移植 WorkBuddy 核心能力给龙虾：规则遵守、任务管理、日志同步、文件解析

---

## 执行过程

### 阶段 1：飞书权限分析

通过查阅 OpenClaw 官方文档确认：
- 旧版 `drive:drive:readonly` / `drive:file:readonly` 已被新版权限体系取代
- 核心缺失权限：`contact:contact.base:readonly`（龙虾无法识别消息发送者）
- 提供完整权限 JSON，用户通过飞书开放平台批量导入

正确权限 scope 清单：
```json
{
  "scopes": {
    "tenant": [
      "contact:user.employee_id:readonly",
      "contact:contact.base:readonly",
      "im:message",
      "im:message:send_as_bot",
      "im:resource",
      ...
    ]
  }
}
```

用户完成飞书权限配置后，重启 gateway 生效。

### 阶段 2：读取规则体系

从 `github.com/XZY0626/ai-rules` 读取 AI_RULES.md v2.4.0：
- L0-L5 六层规则体系
- 核心约束：C盘禁区、凭证零泄露、危险命令封禁、GitHub上传前扫描

### 阶段 3：workspace 初始化

部署文件清单：

| 文件 | 用途 |
|------|------|
| `AI_RULES.md` | AI_RULES v2.4.0 适配版（含 OpenClaw 适配说明） |
| `SOUL.md` | 龙虾人格定义：身份、核心原则、工作习惯、语气 |
| `USER.md` | 主人档案：联系方式、基础设施、模型偏好 |
| `TOOLS.md` | 工具备注：执行环境、GitHub规范、技能位置 |
| `skills/rules-loader.md` | 每次会话自动拉取最新规则 |
| `skills/task-planner.md` | 复杂任务拆解与进度汇报 |
| `skills/github-sync.md` | 操作日志规范推送 GitHub |
| `skills/feishu-file-reader.md` | 飞书文件/云文档解析 |

workspace git 初始提交：`574a5a4`

### 阶段 4：systemPrompt 尝试与回滚

尝试将启动指令写入 `agents.defaults.systemPrompt`，验证发现 openclaw 不支持此字段。
已回滚，改为通过 `SOUL.md` + `AGENTS.md` 中的规则控制启动行为（openclaw 原生机制）。

---

## 结果

- Gateway 状态：active，config valid
- 安全审计：0 critical（与上次相同）
- Workspace 文件：8 个核心文件 + 4 个技能文件，首次 git 提交完成
- 飞书权限：用户已手动开通，gateway 重启后生效

## AI_RULES 适配要点

| 问题 | 解决方案 |
|------|---------|
| 规则面向 Windows 宿主机编写 | 新增适配说明表格，说明 OpenClaw 环境下的对应解释 |
| `systemPrompt` 字段不支持 | 改用 workspace 文件 SOUL.md/AGENTS.md 传达启动规则 |
| `skills/` 机制 | openclaw 通过 workspace 文件系统注入 context，无需 JSON 配置 |

## 遗留事项

- 龙虾尚未完成"出生仪式"（BOOTSTRAP.md 未删除），待龙虾首次对话时自行完成
- 两个飞书历史任务（60句咒语文件 + GitHub规则学习）权限修好后可重新触发
