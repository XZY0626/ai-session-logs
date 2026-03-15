# 2026-03-15 | SELF_KNOWLEDGE 能力加固 + 序列号异常排查

**会话类型**：跟进修复 + 问题排查  
**涉及仓库**：openclaw-config  
**操作人**：WorkBuddy（代主人执行）

---

## 一、背景

上次会话完成了「学术搜索工具部署 + 能力激活可靠性审查」后，发现两个遗留项：
- knowledge-notebook / knowledge-ingest 仅在 skills/ 目录，SELF_KNOWLEDGE.md 未提及，激活可靠性 ⭐⭐
- 心跳 cron 未验证是否运行
  
同时主人报告 OpenClaw UI 弹出："检测到事件间隔异常：预期序列号为68，实际得到的序列号为69"

---

## 二、执行内容

### 1. 心跳 cron 验证

**发现**：cron 已经在运行，`~/.openclaw/cron/jobs.json` 中存在名为"每小时心跳"的 job（实际 schedule 为 `0 */2 * * *`，即每 2 小时）。

运行记录（cron/runs/ 下的 jsonl 文件）显示最近几次均正常执行，状态 `ok`，已通过飞书 deliver。

**结论**：心跳 cron ✅ 正常运行，无需修复。

---

### 2. 序列号异常（69→68）排查

**根因**：WebSocket 断线重连导致。

日志记录：
```
19:24:28 [ws] webchat disconnected code=1006 reason=n/a
19:03:11 [ws] webchat disconnected code=1012 reason=service restart
```

- `code=1006`：网络层异常断开（非正常关闭握手），通常是网络抖动、浏览器页面切到后台、或 Tailscale 隧道短暂中断
- `code=1012`：服务重启（gateway reload），正常现象

**序列号跳号机制**：OpenClaw 使用单调递增序列号跟踪事件流，断线期间服务端继续推进序列号，重连后客户端收到比预期更大的序列号，触发"间隔异常"提示。

**处理方法**：刷新页面（F5）即可，重连后序列号恢复一致。

**是否需要修复**：不需要。这是 WebSocket 长连接的正常行为，不影响龙虾功能。若频繁出现可检查 Tailscale 连接稳定性。

---

### 3. SELF_KNOWLEDGE.md 补充 knowledge-notebook/ingest

**修改内容**：
- 技能文件数量：5 → 7 
- 新增两行技能表条目：
  - `knowledge-notebook.md`：语义搜索历史记忆和工作日志，支持模糊问答和知识回溯
  - `knowledge-ingest.md`：向记忆库写入新知识，支持主人随时投喂片段内容
- 在"我能做的事"章节补充两个能力条目

**验证**：grep 确认4处关键位置均写入（第 36、46、47、150、151 行）

**激活可靠性变化**：knowledge-notebook/ingest ⭐⭐ → ⭐⭐⭐

---

## 三、GitHub 推送

| 仓库 | commit | 内容 |
|------|--------|------|
| openclaw-config | `639f5c9` | fix: SELF_KNOWLEDGE.md - add knowledge-notebook and knowledge-ingest |

---

## 四、当前能力激活可靠性（更新版）

| 能力 | 可靠性 | 状态 |
|------|--------|------|
| 安全规则 L0 | ⭐⭐⭐⭐⭐ | AGENTS.md 启动强制检查 |
| MCP 工具选择 | ⭐⭐⭐⭐ | TOOLS.md 决策树 |
| 学术搜索 | ⭐⭐⭐ | SELF_KNOWLEDGE.md 末尾章节 |
| knowledge-notebook | ⭐⭐⭐ | **已修复：写入 SELF_KNOWLEDGE.md** |
| knowledge-ingest | ⭐⭐⭐ | **已修复：写入 SELF_KNOWLEDGE.md** |
| 飞书文件读取 | ⭐⭐⭐ | TOOLS.md 决策树 |
| 心跳 cron | ⭐⭐⭐⭐ | **已验证：jobs.json 有效，每2小时运行** |
| desktop-commander | ⭐⭐⭐ | TOOLS.md 有优先级说明 |

---

_记录人：WorkBuddy | 已脱敏（无 API Key / Token / 真实密码）_
