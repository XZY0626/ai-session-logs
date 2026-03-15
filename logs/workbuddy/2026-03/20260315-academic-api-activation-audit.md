# 2026-03-15 学术 API 配置 + 激活可靠性审查

**操作者**: WorkBuddy  
**日期**: 2026-03-15  
**涉及系统**: openclaw（Ubuntu 虚拟机）、ai-session-logs、openclaw-config

---

## 一、本次操作摘要

### 背景
用户希望 openclaw（以下称"龙虾"）能调用学术 API 进行算法浮现与创新研究。
Semantic Scholar 因不接受 Outlook 邮箱申请 key，最终方案改为三源：**arXiv + OpenAlex + Crossref**。

### 主要操作
1. 测试三源 API 连通性（虚拟机上）
2. 部署 `academic_search.py` 到 `/home/xzy0626/.openclaw/scripts/`
3. 在 `workspace/skills/` 新增 `SKILL_academic_search.md`（技能感知文件）
4. 更新 `SELF_KNOWLEDGE.md`（追加学术搜索能力章节）
5. 更新 `TOOLS.md` 和 `KNOWLEDGE_BASE.md`

### 测试结果

| API | 状态 | 说明 |
|-----|------|------|
| arXiv | ✅ 正常 | 返回 Whisper 相关论文 |
| OpenAlex | ✅ 正常 | 10000 req/天，返回 Distil-Whisper 等 |
| Crossref | ✅ 正常 | 返回 Whisper-Flamingo、WhisperD 等 |
| Semantic Scholar（无 key） | ❌ 429 限流 | 申请表不接受 Outlook，暂放弃 |
| openclaw-gateway | ✅ 运行中 | 用户级 systemd 服务，正常 |

---

## 二、能力激活可靠性审查

### 审查结论：发现 4 类激活可靠性问题

---

### 问题 1：学术搜索 — 感知路径正确但触发依赖主动读文件 ⚠️

**现象**：用户在飞书问龙虾时，龙虾不调用学术搜索脚本。  
**根因**：
- `SKILL_academic_search.md` 放在 `workspace/skills/`，但 AGENTS.md 的 Every Session Startup Checklist 没有明确要求"读 skills/ 目录"
- AGENTS.md 只要求读：`AI_RULES.md` / `SOUL.md` / `USER.md` / `SELF_KNOWLEDGE.md` / `KNOWLEDGE_BASE.md` / `memory/YYYY-MM-DD.md`
- skills/ 目录的文件需要龙虾**自己意识到去读**，没有强制加载机制

**修复措施（已实施）**：
- `SELF_KNOWLEDGE.md` 末尾追加学术搜索能力章节，包含完整调用示例
- 技能描述写入 `TOOLS.md`
- 给用户提供激活文案：`@龙虾 你现在有学术搜索能力了，读一下 SELF_KNOWLEDGE.md 的最后一节...`

**遗留风险**：skills/ 目录的其他文件（knowledge-ingest、knowledge-notebook 等）同样存在未必被加载的问题。

**建议**：在 AGENTS.md 的 Startup Checklist 中加一条："列出 skills/ 目录，对每个技能文件标题有感知"。

---

### 问题 2：knowledge-notebook / knowledge-ingest — 功能强大但可能从未被激活 ⚠️

**现象**（推测）：龙虾可能不知道自己有本地知识库语义搜索能力。  
**根因**：
- `knowledge-notebook.md`（6974 字节）描述了完整的 sqlite-vec + Qwen embedding 本地知识库
- `knowledge-ingest.md`（3137 字节）描述了如何把文件/链接收录进知识库
- 这两个能力都没有写入 `SELF_KNOWLEDGE.md` 的"我能做的事"章节
- 用户问"帮我记住这篇文章"时，龙虾可能直接写 markdown 而不是用知识库

**建议**：
```
# 应在 SELF_KNOWLEDGE.md "我能做的事" 章节补充：
- **本地知识库**：通过 knowledge-notebook 技能，可语义搜索已收录的文档（sqlite-vec）
- **知识收录**：通过 knowledge-ingest 技能，可把飞书文件/URL/文本收录进知识库
```

---

### 问题 3：feishu-file-reader — 触发条件不清晰 ⚠️

**现象**：龙虾收到飞书文件时，可能直接用 fetch 抓 URL 而不用专用技能。  
**根因**：
- `feishu-file-reader.md`（3319 字节）是专门处理飞书文件的技能
- TOOLS.md 的"工具选择决策树"里提到了它，但描述简短
- 龙虾可能在飞书文件场景下选择 fetch 而非此技能

**建议**：在 AGENTS.md 的工具决策树中加强：
```
飞书收到文件（非消息文本）→ 必须先用 feishu-file-reader skill 解析
```

---

### 问题 4：desktop-commander vs 普通 exec — 优先级可能混淆 ⚠️

**现象**：龙虾在需要大文件读取或代码搜索时，可能用了普通 exec 而不是 desktop-commander。  
**根因**：
- TOOLS.md 已写了"优先于普通 exec"，但没有说明具体触发场景
- desktop-commander 的核心价值在于：流式读取大文件、ripgrep 代码搜索、进程管理

**建议**：在 AGENTS.md 决策树中加：
```
读取文件 > 1MB 或 > 500行 → desktop-commander read_file（流式）
搜索代码内容 → desktop-commander search_code（ripgrep）
```

---

## 三、脱敏说明

本文件已脱敏处理：
- VM IP 已改为 `192.168.x.x`
- API Key、App Secret、Token 均已移除
- 私钥路径仅保留结构性说明

---

## 四、遗留待办

| 项目 | 优先级 | 说明 |
|------|--------|------|
| AGENTS.md 补充 skills/ 加载意识 | 高 | 避免技能文件白写 |
| SELF_KNOWLEDGE.md 补充 knowledge-notebook 能力 | 中 | 龙虾应知道自己有本地知识库 |
| Semantic Scholar key 申请 | 低 | 用 edu 邮箱或等待审批窗口 |
| Playwright MCP 升级 | 低 | 遇到动态页面 fetch 失败时使用 |

---

_记录人: WorkBuddy | 2026-03-15_
