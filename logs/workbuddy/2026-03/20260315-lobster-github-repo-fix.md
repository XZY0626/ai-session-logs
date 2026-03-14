# 2026-03-15 龙虾 GitHub 日志仓库配置 + 自动化任务机制修复

**时间**：2026-03-15（下午）
**执行者**：WorkBuddy
**触发渠道**：用户反馈龙虾未自动执行昨晚指定的任务

---

## 问题分析

### 问题一：自动化任务为什么没有执行

**根本原因：openclaw 没有后台定时器机制。**

之前在 HEARTBEAT.md 里写的"空闲时定期执行"是错误描述，openclaw 的工作模式是：

```
有消息 → 龙虾被唤醒 → 执行 → 睡觉
没消息 → 什么都不发生
```

龙虾没有 cron、没有 systemd timer、没有独立的任务调度器。HEARTBEAT.md 里的任务只有在龙虾收到飞书消息时才会被读取执行，不会"无人值守自动运行"。

**昨晚观察到的日志（06:09）**：龙虾确实被唤醒了（收到了消息），但：
1. 尝试读 `~/.openclaw/memory/round_counter.txt` → 报 `Path escapes sandbox root`（沙箱外路径）
2. 尝试读 `~/.openclaw/workspace/memory/rule_sync_time.txt` → 文件不存在

### 问题二：GitHub 日志仓库未配置

龙虾 workspace 内的 `github-sync.md` 描述了推送到 `~/ai-session-logs/` 的流程，但虚拟机上这个目录从未被初始化。

---

## 修复内容

### 1. memory 路径修正（沙箱路径统一）

| 项目 | 旧路径（错误） | 新路径（正确） |
|------|-------------|-------------|
| round_counter.txt | `~/.openclaw/memory/` | `~/.openclaw/workspace/memory/` |
| rule_sync_time.txt | `~/.openclaw/memory/` | `~/.openclaw/workspace/memory/` |
| last_heartbeat.txt | 不存在 | `~/.openclaw/workspace/memory/` |

所有 memory 文件统一放在 workspace 沙箱内，龙虾可正常读写。

### 2. ai-session-logs 本地仓库初始化

```
~/ai-session-logs/
├── README.md
├── logs/openclaw/2026-03/
└── .git/  (初始化，remote 已配置)
```

- git user.name：Lobster-OpenClaw
- git remote：`git@github.com:XZY0626/ai-session-logs.git`
- SSH key：`~/.ssh/id_github`（ed25519，已生成）

**待完成（需人工）**：将公钥添加到 GitHub Settings → SSH Keys  
公钥：`ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINR+I0migLqCMdYcdggIVb/zvopi4OuIPchs9zEwfbvE lobster-openclaw-vm@github`

### 3. HEARTBEAT.md 修正

明确说明 openclaw 没有后台定时器，任务只在收到消息时执行。真正的定时保障由宿主机 Windows 任务计划程序（LobsterRuleSync，每天 03:00）提供。

---

## 合规说明

- 无 L0 违规
- SSH 公钥仅记录（非私钥），可公开
- 无 API Key 或密码泄露

---

*日志由 WorkBuddy 自动记录 | 2026-03-15*
