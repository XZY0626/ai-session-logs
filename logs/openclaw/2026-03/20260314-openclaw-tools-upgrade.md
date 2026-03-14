# 对话日志

- **AI**: WorkBuddy (Tencent)
- **对话窗口ID**: 5d41f0f03ccc4ccfa2ea9e250679db96
- **日期**: 2026-03-14
- **用户**: XZY0626
- **主题**: OpenClaw 龙虾工具能力升级 — tools.profile: messaging → allow 精确列表

---

## 执行摘要

龙虾（OpenClaw main agent）反馈自己无法读写文件、执行命令、调用外部 API，经排查根因是 `tools.profile = "messaging"`——这是专为纯消息机器人设计的最精简工具集，不具备任何"动手"能力。本次将 tools 配置从 `profile: messaging` 升级为精确 allow 列表，开放 21 项工具能力，禁用 3 项高风险工具，重启 gateway 验证生效。

---

## 根因分析

```json
// 修改前
"tools": {
  "profile": "messaging"
}
```

`messaging` profile 仅包含消息发送相关工具，不含：
- `read` / `write` / `edit`（文件读写）
- `exec` / `process`（命令执行）
- `web_search` / `web_fetch`（网络能力）
- `memory_search` / `memory_get`（记忆系统）
- 其他 Agent 核心工具

龙虾说"capabilities=none"是准确的自我诊断，不是误报。

---

## 工具 Profile 说明（来源：openclaw 官方文档）

| Profile | 核心能力 | 适合场景 |
|---------|---------|---------|
| `messaging` | 仅发消息 | 纯通知机器人 |
| `computer` | 文件+exec+网络 | 单次任务助手 |
| `messaging`（完整） | computer + 记忆 + 定时 + 消息 | 个人助理 |
| `full` | 以上全部 + nodes 硬件控制 | 高级用户 |

---

## 操作记录

| 序号 | 操作类型 | 目标路径/资源 | 操作描述 | 结果 |
|------|---------|-------------|---------|------|
| 1 | 备份 | `~/.openclaw/openclaw.json.bak.20260314_221653` | 修改前自动备份 | ✅ |
| 2 | 修改配置 | `~/.openclaw/openclaw.json` → `tools` 字段 | profile: messaging → allow 精确列表（21项允许，3项禁止） | ✅ |
| 3 | 重启服务 | `systemctl --user restart openclaw-gateway` | 重载配置 | ✅ active (running) |
| 4 | 验证 | openclaw.json tools 字段内容读取 | 确认 21 项 allow 已写入 | ✅ |

---

## 变更详情

```json
// 修改后
"tools": {
  "allow": [
    "read", "write", "edit", "apply_patch",
    "exec", "process",
    "web_search", "web_fetch",
    "browser", "image",
    "memory_search", "memory_get",
    "sessions_list", "sessions_history", "sessions_send", "sessions_spawn", "session_status",
    "message", "cron", "gateway", "agents_list"
  ],
  "deny": [
    "nodes", "canvas", "llm_task"
  ]
}
```

**开放的工具（21项）：**
- 文件操作：`read`, `write`, `edit`, `apply_patch`
- 命令执行：`exec`, `process`
- 网络：`web_search`, `web_fetch`
- 感知：`browser`, `image`
- 记忆：`memory_search`, `memory_get`
- 多任务：`sessions_list/history/send/spawn/status`
- 通信：`message`, `cron`, `gateway`, `agents_list`

**主动禁用（3项）：**
- `nodes`：硬件控制（截图/摄像头），高风险，无实际需求
- `canvas`：可视化工作区，暂不需要
- `llm_task`：工作流引擎，暂不需要

**exec 安全层仍然有效：**
`exec-approvals.json` 里的 38 条命令白名单独立运行，工具层开放 `exec` 只是"允许龙虾提出执行请求"，实际执行仍受白名单约束。

---

## 文件变更清单

- 修改：`~/.openclaw/openclaw.json` — tools 字段从 profile:messaging 改为精确 allow 列表
- 新增：`~/.openclaw/openclaw.json.bak.20260314_221653` — 修改前自动备份

---

## 安全审查

```
上传目标: GitHub (XZY0626/ai-session-logs)
扫描文件数: 1
发现问题:
  - [信息] 无凭证、无密钥、无 IP 地址
凭证泄露检查: ✅ 通过
危险操作检查: ✅ 无危险操作
C盘操作: ⚠️ C:\Windows\Temp\ 写入了临时脚本（标准临时目录，低风险）
```

---

## 待办/遗留问题

- [ ] 龙虾重新配对测试后，验证 `exec` 审批弹窗是否正常工作（首次执行命令时应弹出审批）
- [ ] 如果龙虾还反馈工具不可用，考虑检查 agent 配置里是否有独立的 tools 覆盖

---

## GitHub 提交

- 仓库: `XZY0626/ai-session-logs`
- Commit: `[log] openclaw tools upgrade: messaging profile -> full allow list (21 tools)`
