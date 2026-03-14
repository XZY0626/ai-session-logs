# 2026-03-14 OpenClaw 安全加固 — 安全文档全量实施

## 背景

龙虾（OpenClaw Agent）读取官方安全文档 https://docs.openclaw.ai/gateway/security 后，
列出完整加固清单，但因 gateway 超时无法自行操作，委托 WorkBuddy 实施。

用户特殊要求：`allowInsecureAuth` 保持 `true`（不做修改）。

---

## 实施内容

### 安全字段对照表（最终状态）

| 字段 | 操作 | 最终值 | 说明 |
|------|------|--------|------|
| `gateway.auth.mode` | 新增 | `"token"` | 明确启用 token 认证模式 |
| `gateway.controlUi.allowInsecureAuth` | 新增 | `true` | 用户要求保留 |
| `session.dmScope` | 已存在 | `"per-channel-peer"` | 不同 DM 对话隔离，无需改动 |
| `tools.fs.workspaceOnly` | 新增 | `true` | 文件访问限制在 workspace 目录内 |
| `tools.exec.security` | 新增 | `"allowlist"` | exec 使用白名单审批（龙虾给的 `"ask"` 为无效值，已修正） |
| `tools.elevated.enabled` | 新增 | `false` | 禁止提权操作 |
| `discovery.mdns.mode` | 新增 | `"minimal"` | 最小化 mDNS 广播，减少信息暴露 |
| `logging.redactSensitive` | 新增 | `"tools"` | 日志脱敏（龙虾给的 `true` 为无效值，已修正为 `"tools"`） |
| `gateway.nodes.denyCommands` | 删除 | — | 原值均为无效命令名，审计报 WARN，清理后消除 |

### 文件权限修复

```bash
chmod 600 ~/.openclaw/openclaw.json
chmod 700 ~/.openclaw
```

### tools 配置策略说明

龙虾建议将 `tools.profile` 改回 `messaging`（最小工具集），但这会撤销前一次会话中
WorkBuddy 已升级的 21 项能力白名单配置。本次保留现有 allow/deny 列表，
仅在其基础上叠加 fs/exec/elevated 安全约束，两者不冲突。

---

## 安全审计结果

```
openclaw security audit
Summary: 0 critical · 4 warn · 2 info
```

### WARN 详情（全部为预期/已知项）

| ID | 说明 | 处置 |
|----|------|------|
| `gateway.control_ui.insecure_auth` | `allowInsecureAuth=true` | 用户主动要求保留，已知风险 |
| `channels.feishu.doc_owner_open_id` | 飞书 doc 工具可授权给请求者 | 日常使用场景，暂时接受 |
| `config.insecure_or_dangerous_flags` | 同上，allowInsecureAuth 引发 | 同上 |
| `plugins.installs_unpinned_npm_specs` | openclaw-stepfun 未钉版本 | 低优先级，暂时接受 |

### INFO 详情

- `tools.elevated: disabled` ✅
- `gateway.tailscale_serve`：通过 tailnet 暴露，预期行为 ✅

---

## 修正的龙虾文档错误

龙虾提供的配置片段中有两处字段值与 openclaw 实际枚举不符：

1. `logging.redactSensitive: true` → 实际合法值为 `"off"` 或 `"tools"`，已修正为 `"tools"`
2. `tools.exec.security: "ask"` → 实际合法值为 `"deny"` / `"allowlist"` / `"full"`，已修正为 `"allowlist"`

这两个错误会导致 gateway 启动失败（config invalid），需要通过 `openclaw security audit` 输出的错误提示排查。

---

## 验证

- gateway `active (running)` ✅
- `openclaw security audit` 输出 `0 critical` ✅
- 相比加固前，warn 从 5 条降至 4 条（消除了 `nodes.denyCommands` 无效命令警告）

---

## 时间线

| 时间 | 操作 |
|------|------|
| 22:27 | 读取当前 openclaw.json，分析与安全文档的差距 |
| 22:28 | 生成 patch，发现两处枚举值错误，修正后重新部署 |
| 22:28 | 第一次审计：0 critical · 5 warn |
| 22:29 | 清理 nodes.denyCommands 无效字段，重新部署 |
| 22:29 | 最终审计：0 critical · 4 warn · 2 info ✅ |
