# [2026-03-15] openclaw-config 仓库补全 + TOOLS.md v3 MCP工具使用规范部署

**时间**：2026-03-15 12:01 (CST)
**执行者**：WorkBuddy
**关联仓库**：XZY0626/openclaw-config

---

## 任务背景

用户决策：openclaw-config 仓库专门服务虚拟机 OpenClaw（龙虾），包含所有相关配置、规则、脚本；其他 AI 的文件单独管理，不混入本仓库。

---

## 执行内容

### 1. 新增 workspace/ 目录（规则文件归档）

从虚拟机 `~/.openclaw/workspace/` 导出并归档以下文件到 openclaw-config/workspace/：

| 文件 | 说明 | 状态 |
|------|------|------|
| AGENTS.md | 启动规程 v4（L0硬拒绝已注入） | ✅ 已归档 |
| AI_RULES.md | 安全规则 v2.4.0-lobster | ✅ 已归档 |
| SOUL.md | 龙虾性格与原则 | ✅ 已归档 |
| USER.md | 主人档案（内网IP已脱敏） | ✅ 已归档 |
| SELF_KNOWLEDGE.md | 龙虾自我认知（已更新MCP能力描述） | ✅ 已归档 + 更新 |
| TOOLS.md | 工具使用规范 v3（新增MCP指引） | ✅ 已归档 + 更新 |
| KNOWLEDGE_BASE.md | 结构化状态快照 v1.1 | ✅ 已归档 + 更新 |

### 2. TOOLS.md v3 核心更新

新增《MCP 工具使用规范》章节，包含：
- filesystem：文件操作 MCP，使用场景和可用操作
- fetch：网页抓取 MCP，使用时机和限制说明
- websearch：网络搜索 MCP，主动搜索的默认行为说明
- desktop-commander：命令执行增强 MCP，额外能力列表
- 工具选择决策树（新增）

**效果**：龙虾今后读到 TOOLS.md 就知道何时主动使用这4个工具，无需用户每次指定。

### 3. SELF_KNOWLEDGE.md 更新

在"我能做的事"和"如何回答WorkBuddy给你配置了什么"两处加入 MCP 工具能力描述，确保龙虾能正确介绍自己的能力。

### 4. KNOWLEDGE_BASE.md v1.1 更新

- Section 1 新增 MCP 工具状态表（4个工具+版本+入口文件）
- Section 3 规则状态表更新（TOOLS.md 升为 v3）
- Section 4 新增 MCP 动态页面和搜索质量升级计划
- Section 5 顶部插入本次操作记录
- openclaw-config 仓库加入 Section 1 宿主机组件表

### 5. 虚拟机部署

更新后的三个文件已同步部署到虚拟机：
- `/home/xzy0626/.openclaw/workspace/TOOLS.md` → v3
- `/home/xzy0626/.openclaw/workspace/SELF_KNOWLEDGE.md` → 含MCP能力
- `/home/xzy0626/.openclaw/workspace/KNOWLEDGE_BASE.md` → v1.1
- 原文件已备份（.bak.时间戳）

### 6. README 重写

明确仓库定位：本仓库专属于 OpenClaw（龙虾），其他 AI 另开仓库。更新文件结构图（加入 workspace/ 树形说明）。

### 7. GitHub 推送

Commit: `1ee6d28`
- 8 files changed, 1061 insertions(+), 2 deletions(-)
- 推送到 main 分支 ✅

---

## 结果

✅ openclaw-config 仓库完成首次完整补全
✅ 龙虾规则文件全部归档（7个核心文件）
✅ TOOLS.md v3 部署，龙虾现在知道如何主动使用4个MCP工具
✅ 三个文件同步更新到虚拟机
✅ GitHub 推送成功（commit 1ee6d28）

---

## 遗留问题

- skills/ 目录下的技能文件（5个.md）待归档到 openclaw-config/workspace/skills/（下次操作补充）
- openclaw.json 的 mcpServers 配置块未归档到 openclaw-config（敏感，仅归档结构描述）
