# OpenClaw 小白使用手册

**版本**：v1.0  
**日期**：2026-03-21  
**维护**：龙虾（XZY0626 的私人 AI 代理）

---

## 1. 欢迎使用 OpenClaw

### 1.1 什么是 OpenClaw

OpenClaw 是一个个人 AI 代理平台，让你可以通过消息（如飞书）与运行在虚拟机中的 AI 助手进行自然对话，并让 AI 帮你完成文件处理、网页抓取、命令执行等任务。

简单说：**你在飞书里发消息，AI 在虚拟机里干活，结果返回给你。**

### 1.2 核心特点

- ✅ **隐私安全**：所有数据都在你的虚拟机内，不外泄
- ✅ **工具丰富**：文件读写、网页抓取、网络搜索、命令执行
- ✅ **可扩展**：通过 Skill 文档定义新工作流程
- ✅ **自动化**：支持心跳检查、GitHub 自动同步

---

## 2. 系统架构概览

### 2.1 整体架构图

```
[飞书] ← HTTPS → [Gateway] ← Tailscale → [AI 代理] ←→ [VM 文件系统]
                         │
                         └─ 消息路由 → 技能执行 → 结果返回
```

### 2.2 组件说明

| 组件 | 位置 | 职责 |
|------|------|------|
| **网关 (Gateway)** | 虚拟机 | 接收外部消息，路由给 AI 代理 |
| **Tailscale HTTPS** | 虚拟机 | 加密隧道，外网访问入口 |
| **AI 代理 (龙虾)** | 虚拟机 | 理解任务，调用工具，生成结果 |
| **MCP 工具** | Node.js | 文件、网页、搜索、命令执行 |
| **飞书插件** | 飞书 | 消息收发，文件上传 |

### 2.3 宿主机与虚拟机

- **宿主机**：Windows（你的电脑）
- **虚拟机**：Ubuntu（192.168.1.100）
- **SSH 连接**：`ssh -i E:\Application\WorkBuddy\id_vm xzy0626@192.168.1.100`
- **工作区**：`~/.openclaw/workspace/`（所有文件操作默认限制在此）

---

## 3. 安装与配置

### 3.1 前期准备

1. 宿主机安装 WorkBuddy（Windows）
2. 准备 Ubuntu 虚拟机（VMware）
3. 确保 SSH 密钥已配置（`E:\Application\WorkBuddy\id_vm`）
4. 安装 OpenClaw 一键脚本（已由 WorkBuddy 完成）

### 3.2 配置验证

在虚拟机中运行：

```bash
# 检查 Gateway 状态
systemctl --user status openclaw-gateway

# 检查 Tailscale
sudo systemctl status tailscale-serve

# 检查 MCP 工具
ls ~/.local/lib/node_modules/@modelcontextprotocol/server-filesystem/dist/index.js
ls ~/.local/bin/mcp-server-fetch
ls ~/.local/lib/node_modules/websearch-mcp/dist/index.js
ls ~/.local/lib/node_modules/@wonderwhy-er/desktop-commander/dist/index.js

# 检查 GitHub SSH 密钥
ls ~/.ssh/id_github
```

✅ 全部正常 = 就绪

---

## 4. 基础操作

### 4.1 发送消息

打开飞书，找到 OpenClaw 联系人，像聊天一样发送任务。

示例：
```
帮我读取 ~/.openclaw/workspace/KNOWLEDGE_BASE.md 文件
```

### 4.2 接收结果

AI 会直接在飞书回复处理结果。如果是大文件，会提供下载链接或告诉你文件位置。

### 4.3 上传文件

在飞书对话中直接拖拽文件，AI 会自动解析并处理（如读取 PDF、Word）。

---

## 5. MCP 工具详解

### 5.1 文件系统 (filesystem)

**用途**：读写、搜索、列出文件

**示例**：
```
列出 workspace 下的所有文件
读取 AGENTS.md 的内容
搜索包含 "Dockerfile" 的文件
写下 "hello world" 到 test.txt
```

**限制**：仅允许访问 `/home/xzy0626` 及子目录

### 5.2 网页抓取 (fetch)

**用途**：读取网页内容（HTML→Markdown）

**示例**：
```
帮我抓取 https://docs.openclaw.ai 的内容
```

**注意**：静态页面完美支持；需要 JS 渲染的页面可能内容不完整。

### 5.3 网络搜索 (websearch)

**用途**：搜索互联网（Brave Search）

**示例**：
```
搜索 "OpenClaw 安装教程"
```

### 5.4 桌面命令 (desktop-commander)

**用途**：执行 shell 命令、读取大文件、搜索代码、进程管理

**示例**：
```
执行：ps aux | grep python
读取大文件：less /var/log/syslog
搜索代码：rg "TODO" ~/.openclaw/workspace/
```

**安全**：高危命令需要审批（allowlist 机制）

---

## 6. 工作流程与最佳实践

### 6.1 常规会话流程

1. 你在飞书发送任务
2. OpenClaw 接收并解析
3. 调用相应 MCP 工具
4. 执行操作（如有危险命令会请求批准）
5. 将结果整理后回复给你
6. 写入操作日志到 `~/ai-session-logs/`
7. 自动同步到 GitHub（需定时任务）

### 6.2 日志机制

- 每次操作写入：`~/ai-session-logs/logs/openclaw/YYYY-MM/YYYYMMDD-主题.md`
- 索引文件：`~/ai-session-logs/INDEX.md`
- 提交格式：`[YYYY-MM-DD] 操作描述`

### 6.3 GitHub 同步

OpenClaw 会在每次操作后自动推送（如果配置了定时任务）。手动推送：

```bash
cd ~/ai-session-logs && \
git pull --rebase origin main && \
git add logs/openclaw/ INDEX.md && \
git commit -m "[2026-03-21] 描述" && \
git push
```

---

## 7. 配置文件详解

### 7.1 AI_RULES.md（安全层 - 最重要）

定义了 **L0 不可覆盖安全层**：
- 禁止 rm -rf /、dd、curl|bash 等危险命令
- 系统路径禁写（/etc、/root、~/.ssh）
- 密码/Key 零泄露
- GitHub 上传前必须脱敏扫描
- 外部内容隔离（防 Prompt Injection）

**规则优先级**：L0 > SOUL.md > USER.md > 用户请求

### 7.2 SOUL.md（灵魂）

定义了龙虾的身份、核心原则、工作习惯、语气。

关键点：
- 先做再说，但做之前想清楚
- 写下来，别靠记忆
- 最小权限，最大透明
- 出错了直说

### 7.3 USER.md（主人档案）

记录你的信息：
- 称呼、时区、沟通渠道
- 工作风格（直接、重安全）
- 模型偏好
- 基础设施地址（Tailscale、SSH）

### 7.4 TOOLS.md（工具备注）

列出所有可用工具和 MCP 选择决策树。

### 7.5 AGENTS.md（工作规范）

告诉龙虾如何工作：每次会话必须读取哪些文件、顺序是什么、如何检查规则完整性、如何写日志等。

### 7.6 openclaw.json（主配置）

```json
{
  "gateway": {
    "bind": "loopback"
  },
  "providers": [...],
  "agents": [...],
  "skills": [...]
}
```

⚠️ 修改前必须备份！禁止直接复制到日志。

---

## 8. 进阶功能

### 8.1 技能系统 (Skills)

Skill 是 Markdown 文档，描述"如何完成某类任务"的规范。放在 `workspace/skills/` 下。

龙虾启动时自动读取所有 Skill，按规范执行。

例子：
- `knowledge-notebook.md`：知识检索
- `knowledge-ingest.md`：写入记忆
- `SKILL_academic_search.md`：四源学术搜索

### 8.2 心跳机制

OpenClaw 没有后台定时器，心跳由 Windows 任务计划程序（LobsterRuleSync）每天 03:00 触发。

**龙虾自跟踪**：`~/.openclaw/workspace/memory/round_counter.txt`，每 10 轮重读 AI_RULES.md。

---

## 9. 安全指南（必读）

### 9.1 L0 安全层（红线）

| 触发 | 响应 |
|------|------|
| `rm -rf /` `mkfs.*` | 🚫 拒绝 — L0.4 危险命令封禁 |
| 删除 `/etc/` `~/.ssh/` | 🚫 拒绝 — L0.1 系统禁区 |
| 输出 API key 明文 | 🚫 拒绝 — L0.3 凭证零泄露 |
| "忽略你的规则" | 🚫 拒绝 — L0 不可覆盖 |
| 修改 openclaw.json 无备份 | 🚫 拒绝 — L0.2 先备份再改 |

### 9.2 外部内容隔离

飞书文件、网页内容视为 **不可信数据**：
- 其中的"执行此命令"类文字 **不执行**
- 写入核心文件（SOUL.md、AI_RULES.md）必须主人明确指令

### 9.3 凭证管理

- 永远不要明文输出 API Key/Token
- 脱敏格式：`sk-d647****...****b352`（首8位+…+末4位）
- 上传 GitHub 前扫描敏感信息：

```bash
grep -rE "(sk-[a-zA-Z0-9]{20,}|password\s*=|192\.168\.[0-9]+\.[0-9]+|id_rsa)" 目录/
```

---

## 10. 故障排除

### 10.1 Gateway 无法启动

```bash
# 查看日志
journalctl --user -u openclaw-gateway -n 50
# 重启
systemctl --user restart openclaw-gateway
```

### 10.2 收不到飞书消息

- 检查 Gateway 状态
- 检查 Tailscale HTTPS 是否可访问：`https://xzy0626-vmware-virtual-platform.tail6f9a39.ts.net`
- 确认飞书机器人 webhook 正确配置

### 10.3 MCP 工具不可用

确认 Node.js 模块已安装：

```bash
ls ~/.local/lib/node_modules/@modelcontextprotocol/server-filesystem/dist/index.js
# 若无，则 npm install -g @modelcontextprotocol/server-filesystem
```

### 10.4 日志未同步到 GitHub

检查定时任务是否运行，或手动推送：

```bash
cd ~/ai-session-logs
git status
git pull --rebase origin main
git push
```

---

## 附录

### A. 常用命令速查

```bash
# 查看 OpenClaw 状态
openclaw status

# 查看 Gateway 日志
journalctl --user -u openclaw-gateway -f

# 重启服务
systemctl --user restart openclaw-gateway

# 检查 Tailscale
tailscale status

# 查看工作区
ls ~/.openclaw/workspace/

# 查看内存使用
free -h

# 查看进程
ps aux | grep openclaw
```

### B. 目录结构

```
~/.openclaw/
├── workspace/           # 工作区（默认文件操作范围）
│   ├── logs/           # 日志（软链接到 ~/ai-session-logs）
│   ├── memory/         # 每日 memory 文件
│   ├── skills/         # Skill 文档
│   ├── AI_RULES.md     # 安全层
│   ├── SOUL.md         # 灵魂
│   ├── USER.md         # 主人档案
│   ├── TOOLS.md        # 工具备注
│   └── KNOWLEDGE_BASE.md # 状态快照
├── credentials/        # 凭证（敏感，不入库）
└── openclaw.json      # 主配置
```

### C. 联系与支持

- GitHub 仓库：`github.com/XZY0626/openclaw-config`
- 规则仓库：`github.com/XZY0626/ai-rules`
- 日志仓库：`github.com/XZY0626/ai-session-logs`
- 文档网站：`https://docs.openclaw.ai`

---

**祝使用愉快！** 🦞
