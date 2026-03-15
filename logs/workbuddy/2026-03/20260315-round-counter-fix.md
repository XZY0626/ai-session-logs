# 2026-03-15 round_counter 修复 + knowledge-notebook embedding 说明

## 任务背景

Openclaw 自检发现 `round_counter.txt` 仍为 0，规则重读机制未激活。本次会话完成最后一项未修复项。

## 根因分析

经过详细排查，发现了**两个叠加的根因**：

### 根因 1：路径不一致（最关键）
- `AGENTS.md` 第51行读的是 `~/.openclaw/memory/round_counter.txt`（沙盒外路径）
- `jobs.json` cron 心跳写的是 `/home/xzy0626/.openclaw/workspace/memory/round_counter.txt`（workspace 内路径）
- `HEARTBEAT.md` 明确记录：`~/.openclaw/memory/` 是**错误路径**，会报 `Path escapes sandbox root`
- 结果：cron 在写一个文件，AGENTS.md 在读另一个文件，永远不会触发规则重读

### 根因 2：jobs.json 内联 Python 不可靠
- 上次修复将写入命令嵌在 `exec:` 内联 Python 里，引号传递不稳定
- 需要改为调用独立的 shell 脚本文件

## 修复内容

### 1. 创建 heartbeat_write.sh 脚本
路径：`/home/xzy0626/.openclaw/scripts/heartbeat_write.sh`

功能：
- 更新 `workspace/memory/last_heartbeat.txt` → 当前时间戳
- 递增 `workspace/memory/round_counter.txt`（读取 → +1 → 写回）
- 更新 `workspace/memory/rule_sync_time.txt` → 当前时间戳 + `(cron-heartbeat)`

已测试：`bash heartbeat_write.sh` 输出 `heartbeat_write done: counter=1`

### 2. 修复 AGENTS.md 路径（第51行）
- 修复前：`cat ~/.openclaw/memory/round_counter.txt`
- 修复后：`cat ~/.openclaw/workspace/memory/round_counter.txt`
- 同步修复 `rule_sync_time.txt` 的路径引用
- 使用 `sed -i` 直接在虚拟机修改，避免跨平台传输问题

### 3. 更新 jobs.json payload
- 移除内联 `exec: python3 -c "..."` 命令（引号不稳定）
- 替换为 `exec: bash /home/xzy0626/.openclaw/scripts/heartbeat_write.sh`

## 验证结果

| 文件 | 修复前 | 修复后 |
|------|--------|--------|
| `workspace/memory/round_counter.txt` | 0（未更新） | 1（测试写入成功） |
| `workspace/memory/last_heartbeat.txt` | 停在旧时间 | 2026-03-15 20:59:49 |
| `workspace/memory/rule_sync_time.txt` | 初始值 | 2026-03-15 20:59:49 (cron-heartbeat) |
| AGENTS.md 第51行 | 错误路径 | workspace/memory/ 正确路径 |
| jobs.json payload | 内联 Python | 调用独立脚本 |

## GitHub 推送

- 仓库：`github.com/XZY0626/openclaw-config`
- commit：`62ea0dc` — fix: round_counter path + heartbeat_write.sh + jobs.json payload
- 包含：`workspace/AGENTS.md`（路径修复）、`scripts/heartbeat_write.sh`（新增）、`cron/jobs.json`（payload更新）

## knowledge-notebook 与 Qwen text-embedding-v3 联动说明

本次会话也回答了用户对 embedding 联动的疑问：

### 联动架构
```
用户查询意图
    ↓
knowledge-notebook Skill（触发层：识别"查历史记录"意图）
    ↓
openclaw memory search 命令
    ↓
sqlite-vec 向量数据库（存储 embedding 向量）
    ↓
Qwen text-embedding-v3（将文字转为 1536 维向量）
```

### 核心特点
- **语义搜索**，不是关键词匹配——"Q2目标是留存率"和"用户增长方向"能相互召回
- **knowledge-ingest** 是存入侧，**knowledge-notebook** 是查询侧
- 共享同一个 embedding 模型和 sqlite-vec 数据库（`~/.openclaw/memory/main.sqlite`）
- 召回阈值：score > 0.5 相关，> 0.7 高度相关

## 整体修复进度（6/6）

| 问题 | 状态 |
|------|------|
| cron 频率 every 2h | ✅ 改为 every 1h |
| last_heartbeat.txt 未更新 | ✅ 已修复（独立脚本） |
| round_counter.txt 仍为 0 | ✅ 已修复（路径统一） |
| rule_sync_time.txt 初始值 | ✅ 已修复（独立脚本） |
| skills 未挂载到 OpenClaw | ✅ 补 YAML frontmatter |
| round_counter 路径错误 | ✅ sed 修复 AGENTS.md |
