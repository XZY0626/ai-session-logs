# 2026-03-15 可靠性加固：AGENTS.md v5 升级

## 任务背景

用户关注四个能力的激活可靠性，并要求在不影响三端正常运行的前提下改进：
- knowledge-notebook/ingest（⭐⭐ → 目标 ⭐⭐⭐）
- 心跳 cron（⭐⭐ → 目标 ⭐⭐⭐）
- 学术搜索（⭐⭐⭐ → 目标 ⭐⭐⭐⭐）
- MCP 工具选择（⭐⭐⭐⭐ → 维持/提升）

## 分析结论

通过读取 AGENTS.md、TOOLS.md、SOUL.md、jobs.json 完整内容，发现以下激活缺口：

| 缺口 | 位置 | 影响 |
|------|------|------|
| AGENTS.md Skills 表缺少 knowledge-notebook/ingest/academic_search | 启动时不会提示读这三个 skill | 三技能仅依赖 SELF_KNOWLEDGE.md，若不仔细读则激活失败 |
| AGENTS.md 启动清单未要求读 TOOLS.md | 每次启动不一定知道 MCP 工具决策树 | MCP 工具选择依赖记忆，易遗忘 |
| 心跳 cron consecutiveErrors 无检查 | job 失败了不会主动告警 | 心跳失效无感知 |

## 实施内容

### AGENTS.md v4 → v5 变更（commit a7dab70）

**1. Skills 表补全（3条新增）**
```markdown
| `skills/knowledge-notebook.md` | Query historical memory and work logs with semantic search; fuzzy Q&A |
| `skills/knowledge-ingest.md`   | Write new knowledge into memory store; owner can feed fragments anytime |
| `skills/SKILL_academic_search.md` | Four-source academic search (arXiv+OpenAlex+Crossref+S2) |
```

**2. 启动清单 step 4 加注释**
```
4. [PARALLEL] Read SELF_KNOWLEDGE.md (skills list: knowledge-notebook, knowledge-ingest, academic-search)
```

**3. 启动清单新增 step 9+10**
```
9.  [PARALLEL] Read TOOLS.md — know your MCP tools and decision tree before starting any task
10. Check heartbeat cron health: python3 one-liner checks consecutiveErrors — if any errors, notify owner
```

**4. 版本号升级**
- Header: `MODIFIED v4` → `MODIFIED v5`
- CHECKSUM: `v4-2026-03-15` → `v5-2026-03-15`
- version self-check grep 字符串同步更新

### 安全性确认

- ✅ 只增加内容，未删除任何现有规则
- ✅ L0 HARD REJECT 表格完整保留
- ✅ openclaw.json 相关保护规则未触动
- ✅ 三端运行无影响（gateway/mobile/web 均不依赖 AGENTS.md 行号）
- ✅ 心跳 cron 检查是只读操作，不会修改 jobs.json

## 可靠性变化

| 能力 | 变更前 | 变更后 | 加固方式 |
|------|--------|--------|---------|
| knowledge-notebook/ingest | ⭐⭐ | ⭐⭐⭐ | 写入 AGENTS.md Skills 表 + step 4 注释 |
| 心跳 cron | ⭐⭐ | ⭐⭐⭐ | step 10 主动检查 consecutiveErrors |
| 学术搜索 | ⭐⭐⭐ | ⭐⭐⭐⭐ | 写入 AGENTS.md Skills 表（明确 when to use） |
| MCP 工具选择 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | step 9 强制每次启动读 TOOLS.md |

## 推送记录

- openclaw-config: commit a7dab70，1 file changed, 9 insertions(+), 5 deletions(-)
- 分支: main，remote: github.com/XZY0626/openclaw-config.git
