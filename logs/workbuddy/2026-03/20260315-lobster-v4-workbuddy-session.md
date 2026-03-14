# 2026-03-15 WorkBuddy 操作日志 — 龙虾 v4 双轨规则 + 完整审计

## 基本信息
| 项目 | 内容 |
|------|------|
| 日期 | 2026-03-15 |
| AI   | WorkBuddy (Tencent) |
| 涉及系统 | 宿主机 Windows + VMware Ubuntu 虚拟机 (192.168.1.100) |
| 关联日志 | [20260315-lobster-v4-dual-rule-track.md](20260315-lobster-v4-dual-rule-track.md) |
| 关联日志 | [20260315-lobster-v4-full-audit.md](20260315-lobster-v4-full-audit.md) |

---

## 对话背景

用户认为"每10轮对话重载规则"这个机制在自动化任务场景下不够可靠（Openclaw 自动跑任务时轮次可能很少），要求升级为更可靠的规则保障机制，同时让 Openclaw 龙虾能完整理解自己的配置。

---

## 本次操作内容

### 一、分析并确定 B 方案（双轨制）

**轨道一：任务级触发**
- 修改 AGENTS.md，规定每次收到新任务必须先执行 Step 0（读 AI_RULES.md + 确认 L0 合规）
- 不依赖轮次计数器，不依赖龙虾自觉性

**轨道二：时间级触发**
- 在宿主机 Windows 创建计划任务 `LobsterRuleSync`
- 每天凌晨 03:00 运行 `E:\Application\WorkBuddy\sync_rules_daily.ps1`
- 从 GitHub 拉取最新 AI_RULES.md 写入虚拟机 workspace
- 开机补跑机制：`-StartWhenAvailable`，电脑关机也不漏跑

### 二、新增文件 SELF_KNOWLEDGE.md

**问题**：Openclaw 龙虾不理解自己的配置来源（问 `ls -la ~/.openclaw/agents/` 来探索环境），说明缺乏自我认知文档。

**解决**：在 workspace 新增 `SELF_KNOWLEDGE.md`，内容包含：
- 我是怎么来的（WorkBuddy 配置链条）
- WorkBuddy 传授的工作方式（并行/清晰展示/技能搭配）
- 5 个技能文件的用途说明
- 标准自述格式

### 三、完整合规审计

对本次及历史所有操作做合规审计，发现：

**1 个历史违规（中等）**
- L0.3 违规：2026-03-14/15 两次读取 openclaw.json 时 API Key 明文出现在对话上下文
- 处置：无写入日志，无推送 GitHub；脱敏脚本已部署；建议换 Key（可选）

**6 个漏洞已修复**

| # | 漏洞描述 | 严重程度 | 修复措施 |
|---|---------|---------|---------|
| 1 | 读 openclaw.json 无脱敏 | 🔴 高 | 部署 workspace/tools/redact_json.py，TOOLS.md 强制要求 |
| 2 | AGENTS.md 无版本自检，可被 openclaw 升级覆盖 | 🟡 中 | v4 加入签名标记 + 启动时 hash 自检 |
| 3 | 飞书输入无 Prompt Injection 防护 | 🟡 中 | AGENTS.md/TOOLS.md 明确禁止执行外部内容 |
| 4 | openclaw.json 未纳入 L0 HARD REJECT 保护 | 🟡 中 | L0 表新增 openclaw.json / API Key 类型 |
| 5 | 无开机自检，三端状态靠猜 | 🟢 低 | 部署 workspace/tools/check_startup.sh，每日自动运行 |
| 6 | /tmp/redact_json.py 重启后消失 | 🟢 低 | 持久版存 workspace/tools/，每日同步刷新 |

### 四、三端开机自检结果（10/10 全通过）

```
✅ openclaw-gateway.service — active (running)
✅ tailscaled.service       — active (running)  
✅ tailscale 网络连通       — 已连接
✅ tailscale serve          — HTTPS 代理 18789 正常
✅ workspace/AGENTS.md      — 存在（含 v4 签名）
✅ workspace/AI_RULES.md    — 存在（v2.4.0-lobster）
✅ workspace/SELF_KNOWLEDGE.md — 存在
✅ workspace/tools/redact_json.py — 存在
✅ workspace/tools/check_startup.sh — 存在
✅ memory/rule_sync_time.txt — 存在
```

---

## 文件变更清单

### 虚拟机 workspace（已 git commit d8c94b5）
| 文件 | 操作 | 说明 |
|------|------|------|
| `AGENTS.md` | 升级 v4 | 任务级规则触发 + 版本签名自检 + Prompt Injection 防护 + openclaw.json 禁直读 |
| `TOOLS.md` | 升级 v2 | 脱敏读取规范 + Prompt Injection 防护 |
| `SELF_KNOWLEDGE.md` | 新增 | 龙虾自我说明书 |
| `tools/redact_json.py` | 新增 | 脱敏读取 openclaw.json 工具 |
| `tools/check_startup.sh` | 新增 | 三端开机自检脚本 |

### 宿主机 Windows
| 文件/任务 | 操作 | 说明 |
|---------|------|------|
| `E:\Application\WorkBuddy\sync_rules_daily.ps1` | 升级 | 新增每日运行脱敏脚本刷新 + 开机自检 |
| Windows 任务计划程序 `LobsterRuleSync` | 新增 | 每日 03:00 同步，开机补跑 |

---

## 合规说明

- **无敏感信息写入日志**：openclaw.json API Key 未被记录
- **无明文凭证推送 GitHub**：已核查，日志均为脱敏版
- **操作均在已授权范围内**：修改 workspace 文件、创建计划任务，均经用户明确授权
- **L0 合规**：本次操作不涉及 L0.1~L0.7 任何禁止行为

---

*日志由 WorkBuddy 自动记录 | 2026-03-15*
