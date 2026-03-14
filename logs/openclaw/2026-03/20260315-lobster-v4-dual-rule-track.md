# 龙虾 v4 — 双轨规则保障 + SELF_KNOWLEDGE.md

**日期：** 2026-03-15  
**操作者：** WorkBuddy  
**对象：** OpenClaw 龙虾🦞（虚拟机 192.168.1.100）

---

## 背景

用户反馈：
1. 龙虾在自动化场景下，"10轮对话"触发机制实际无效（任务跑完可能只有1轮对话）
2. 龙虾对自己的配置和来源不够理解，无法向主人说明"WorkBuddy给你配置了什么"

---

## 本次操作内容

### 1. 规则触发机制：从"10轮对话"升级为"双轨制"

**轨道一 — 任务级触发（AGENTS.md v3）**
- 每次开始新任务时，强制执行 Step 0：`cat AI_RULES.md | head -60`
- 适用于：主人发新消息、定时任务启动、长时间暂停后恢复
- 规则缺失时：拒绝所有操作，要求主人恢复文件

**轨道二 — 时间级同步（Windows 定时任务）**
- 任务名：`LobsterRuleSync`
- 执行脚本：`E:\Application\WorkBuddy\sync_rules_daily.ps1`
- 执行时间：每天 03:00（开机后补跑，不会漏）
- 动作：从 GitHub 拉取最新 AI_RULES.md → 上传虚拟机 → 写 rule_sync_time.txt 时间戳

**核心逻辑：**
- 不依赖龙虾"自觉数轮次"
- 规则文件每天从 GitHub 权威来源更新，虚拟机里永远是最新的
- 龙虾每次任务开始读的是已经同步好的规则

### 2. 新增 SELF_KNOWLEDGE.md

让龙虾能正确理解和回答：
- "我是怎么来的"（WorkBuddy 配置链条）
- "我有什么能力"（5个技能文件的用途）
- "WorkBuddy 给我配置了什么"（完整自述格式）
- "我的安全规则体系是什么"（L0/L1/L2 层级）

### 3. AGENTS.md v3 关键变更

- 新增 `TASK-LEVEL RULE ENFORCEMENT` 章节
- 新增 `memory/rule_sync_time.txt` 时间戳检查
- 新增 "What You Are" 章节，引导龙虾读 SELF_KNOWLEDGE.md
- startup checklist 第6条：检查规则同步时间

---

## 文件变更

| 文件 | 操作 | 说明 |
|------|------|------|
| `AGENTS.md` | 更新 (v3) | 任务级规则触发，自我认知引导 |
| `SELF_KNOWLEDGE.md` | 新增 | 龙虾完整自我说明书 |
| `memory/round_counter.txt` | 初始化 | 轮次计数器归零 |
| `memory/rule_sync_time.txt` | 初始化 | 写入首次同步时间戳 |
| `E:\Application\WorkBuddy\sync_rules_daily.ps1` | 新增 | 每日规则同步脚本 |
| Windows 任务计划 `LobsterRuleSync` | 新增 | 每天03:00自动同步 |

---

## Git 记录

- workspace commit: `e079169` — feat: AGENTS v3 task-level rules + SELF_KNOWLEDGE.md
- Windows 定时任务：`LobsterRuleSync` 状态 Ready，下次运行 2026-03-15 03:00

---

## 如何让龙虾理解自己的设定（飞书发这条）

```
龙虾，读取 SELF_KNOWLEDGE.md，然后告诉我：
1. 你是怎么被配置的？（谁配置的，用什么方式）
2. WorkBuddy 给你传授了哪些工作方式？列举3条
3. 你现在有哪些技能文件？各自的作用是什么？
4. 如果有人问你"你能做什么"，你会怎么回答？
```

预期：龙虾能准确说出 WorkBuddy 配置链条 + 5个技能 + WorkBuddy DNA 内容。

---

_日志由 WorkBuddy 生成，已脱敏（无 API Key 等敏感信息）_
