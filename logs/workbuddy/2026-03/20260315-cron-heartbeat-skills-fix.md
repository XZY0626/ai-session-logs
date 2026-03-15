# 2026-03-15 四项可靠性修复：cron频率+心跳写入+skill识别

## 问题来源

Openclaw 自检报告了以下 5 个未解决问题：
1. 心跳 cron 频率仍为 `0 */2 * * *`（每2小时），应为每小时
2. `last_heartbeat.txt` 停在 06:22:25，cron 跑过但未写入
3. `round_counter.txt` 仍为 0，多轮对话后未递增
4. `rule_sync_time.txt` 仍为初始值，从未同步
5. knowledge-ingest/knowledge-notebook 未出现在可用技能列表

## 根因分析

### cron 频率问题
- `jobs.json` 中 schedule.expr 是 `0 */2 * * *`，之前修改未生效（被后续覆盖），本次重新确认修复

### 心跳未写入文件
- 之前的 payload message 里步骤5用了 `$HOME` shell 变量 + 复杂引号，在 JSON 字符串里不会被正确执行
- 根本原因：心跳消息是 agentTurn，龙虾读到后需要自己执行 exec 命令；原来的格式不是明确的 exec 指令

### skills 未被识别
- 通过分析 `openclaw.json` 发现：`commands.nativeSkills: "auto"` 会自动扫描 `workspace/skills/`
- 但要被识别，skill 文件必须有 **YAML frontmatter**（`name:` + `description:` + `activation:`）
- 对比发现：
  - `knowledge-ingest.md` ✅ 已有标准 frontmatter
  - `knowledge-notebook.md` ❌ 缺少 frontmatter（只有 `# title`）
  - `SKILL_academic_search.md` ❌ 缺少 frontmatter

## 修复内容

### 1. jobs.json — cron 频率 + 心跳消息重写

**频率**：`0 */2 * * *` → `0 * * * *`（每小时整点触发）

**心跳消息步骤5改写**（用明确 exec 指令替代 shell 变量）：
```
exec: echo "$(date '+%Y-%m-%d %H:%M:%S')" > /home/xzy0626/.openclaw/workspace/memory/last_heartbeat.txt
exec: python3 -c "p='/home/xzy0626/.openclaw/workspace/memory/round_counter.txt'; n=int(open(p).read().strip() or '0')+1; open(p,'w').write(str(n))"
exec: echo "$(date '+%Y-%m-%d %H:%M:%S') (heartbeat-sync)" > /home/xzy0626/.openclaw/workspace/memory/rule_sync_time.txt
```

### 2. knowledge-notebook.md — 新增 YAML frontmatter
```yaml
---
name: knowledge-notebook
description: >
  Intelligent knowledge notebook for OpenClaw. Activate when the user wants to
  query past work logs, search historical memory, ask questions about previous sessions...
  Trigger phrases: "查一下之前", "历史记录", "recall", "search my notes"...
activation: explicit
version: "1.0"
---
```

### 3. SKILL_academic_search.md — 新增 YAML frontmatter
```yaml
---
name: academic-search
description: >
  Four-source academic paper search (arXiv + OpenAlex + Crossref + Semantic Scholar)...
  Trigger phrases: "搜索论文", "找文献", "find papers", "literature search"...
activation: explicit
version: "2.0"
---
```

### 4. memory 文件初始化（当前值）
| 文件 | 修复前 | 修复后 |
|------|--------|--------|
| `last_heartbeat.txt` | 2026-03-15 06:22:25 | 2026-03-15 20:52:50 |
| `round_counter.txt` | 0 (保持) | 0（下次心跳递增） |
| `rule_sync_time.txt` | 00:33:17 (initial) | 2026-03-15 20:52:50 (WorkBuddy-sync) |

## 推送记录

- openclaw-config: commit 32d58e1
  - 新增 `cron/jobs.json`（首次加入 git 追踪）
  - 修改 `workspace/skills/SKILL_academic_search.md`（+frontmatter）
  - 修改 `workspace/skills/knowledge-notebook.md`（+frontmatter）
- 3 files changed, 66 insertions(+)

## 预期效果

| 问题 | 修复前 | 修复后 |
|------|--------|--------|
| cron 频率 | 每2小时 | **每小时** ✅ |
| last_heartbeat 更新 | 不写入 | **每次心跳写入** ✅ |
| round_counter 递增 | 不递增 | **每次心跳+1** ✅ |
| rule_sync_time 更新 | 固定初始值 | **每次心跳更新** ✅ |
| skills 识别 | knowledge-notebook/academic-search 不可见 | **3个skills均有frontmatter** ✅ |

## 备注

- feishu-file-reader/github-sync/rules-loader/task-planner/workbuddy-dna 这5个 skill 没有 frontmatter，属于手动调用型，不需要 nativeSkills 自动识别，保持原样
- openclaw 内置 heartbeat `{"every": "1h"}` 与 cron jobs 是两套独立机制，互不干扰
- 下次 Openclaw 自检应可验证所有5项问题均已解决
