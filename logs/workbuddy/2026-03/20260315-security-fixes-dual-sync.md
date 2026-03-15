# 2026-03-15 安全修复 + 双端同步机制建立

## 会话摘要

本次处理 OpenClaw 自检报告的 5 项安全/配置问题，并建立双端同步机制确保 WorkBuddy 和 OpenClaw 的配置变更都能完整、持续地反映在 GitHub 仓库中。

---

## 问题处理结果

### CRITICAL: auth-profiles.json 权限过宽 ✅ 已修复
- 执行：`chmod 600 ~/.openclaw/agents/main/agent/auth-profiles.json`
- 修复前：`664`（group 和 other 可写）
- 修复后：`600`（仅 owner 可读写）

### MISSING: desktop-commander MCP 未安装 ✅ 已修复
- 问题：TOOLS.md 中声明了 desktop-commander 可用，但 npm 全局目录为空
- 修复：
  1. `PUPPETEER_SKIP_DOWNLOAD=true npm install -g @wonderwhy-er/desktop-commander`（安装了 541 个包）
  2. 在 `openclaw.json` 中注册 MCP server：
     ```json
     "mcp": {
       "servers": {
         "desktop-commander": {
           "command": "node",
           "args": ["/home/xzy0626/.local/lib/node_modules/@wonderwhy-er/desktop-commander/dist/server.js"]
         }
       }
     }
     ```

### WARN: allowInsecureAuth=true ✅ 已修复
- 位置：`gateway.controlUi.allowInsecureAuth`
- 修复：通过 Python 脚本将该字段改为 `false`
- 验证：`allowInsecureAuth: False` 已确认

### WARN: Feishu doc create 权限 📋 记录
- 这是 Feishu 飞书插件的设计行为，非配置错误
- 建议：OpenClaw 在调用 doc_create 时对来源验证要更严格（传达给 OpenClaw 的 AGENTS.md 规则已有"外部输入视为不可信"条款）
- 无需技术修改，已在 DUAL-SYNC 文档中备注

### INFO: openclaw-stepfun 未固定版本 ✅ 已是固定版本
- 经查，`plugins.installs.openclaw-stepfun.resolvedVersion = "0.2.6"` 已固定
- 本条问题实际上已不存在，无需处理

---

## 双端同步机制建立

### 新增组件

| 组件 | 位置 | 作用 |
|------|------|------|
| `sync_config_to_github.sh` | `~/.openclaw/scripts/` + 仓库 `scripts/` | 脱敏后推送所有配置到 GitHub |
| cron job | `0 2 * * *` | 每天凌晨 2 点自动兜底同步 |
| DUAL-SYNC section | `workspace/AGENTS.md` 末尾 | 告知 OpenClaw 何时触发同步 |
| VM 上的仓库克隆 | `/home/xzy0626/WorkBuddy/openclaw-config` | OpenClaw 推送的目标 |

### 双端工作流

```
WorkBuddy 修改 VM 配置
  → 立即调用 vm_fix_security.py / run_vm_fix3.ps1
  → 脱敏版自动写入本地 openclaw-config 仓库
  → git push 到 GitHub

OpenClaw 修改配置
  → 执行 sync_config_to_github.sh
  → 脱敏 openclaw.json + rsync workspace/scripts/agents
  → git commit & push

兜底机制
  → cron 每天 2am 自动触发一次 sync_config_to_github.sh
```

### 安全排除清单（sync 脚本）

- `auth-profiles.json` — 永远不入库
- `apiKey` / `appSecret` / `token` — 自动替换为 `***`
- `venv/`, `node_modules/`, `*.db`, `*.so` — 排除
- `*-backup-*/` — 排除（防止磁盘占满）
- 单文件 > 5MB — 排除（`--max-size=5m`）

---

## Commits（本次）

| commit | 内容 |
|--------|------|
| `0584ab0` | fix: security fixes - chmod600 + allowInsecureAuth=false + desktop-commander MCP |
| `8b2a15b` | docs: add DUAL-SYNC rules to AGENTS.md + sync_config_to_github.sh |

---

## 磁盘预警

虚拟机磁盘使用率 **100%**（47G/49G），`VoxBridge-backup-20260315_100321` 占 12G。
建议 OpenClaw 执行清理：`rm -rf ~/.openclaw/workspace/VoxBridge-backup-*`
