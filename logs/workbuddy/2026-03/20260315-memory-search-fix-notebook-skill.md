# 20260315 — Memory Search 修复 + NotebookLM 风格知识库 Skill 部署

## 基本信息

| 项目 | 内容 |
|------|------|
| **AI** | WorkBuddy（腾讯云） |
| **日期** | 2026-03-15 |
| **对话主题** | Memory Search 配置路径修复 + 通用 Embedding 方案设计 + knowledge-notebook Skill 创建 |
| **OpenClaw 版本** | v2026.3.13 |
| **核心成果** | Memory Search 从"配置了但空转"变成真正激活，62个文件/137个chunks全量建立语义索引 |

---

## 执行摘要

用户提问 Memory Search 技术原理、是否有备选 embedding 方案，以及希望仿照 NotebookLM 设计一个适配 OpenClaw 的知识库技能。

WorkBuddy 系统排查了 Memory Search 失败的真正原因（`fetch failed`）：`remote.baseUrl` 和 `remote.apiKey` 应写在 `agents.defaults.memorySearch.remote` 子块，之前错误地配在了 `auth-profiles.json` 里，导致 OpenClaw 始终向 `api.openai.com` 发请求。

修复后，Memory Search 完全激活（Vector + FTS 双引擎，1024维向量），并创建了仿 NotebookLM 的 `knowledge-notebook` Skill，已部署到 workspace。

---

## 操作记录

### 第一阶段：根因诊断

| 步骤 | 发现 |
|------|------|
| 查 `openclaw.json` memorySearch 块 | 顶层 `memorySearch` 是空 `{}`，真正配置在 `agents.defaults.memorySearch` |
| 查 `agents.defaults.memorySearch` | 有 `provider: openai`、`model: text-embedding-v3`，但**缺少 `remote` 子块** |
| 直接 curl Dashscope API | HTTP 200，API key 有效，服务器侧完全正常 |
| 查 OpenClaw 源码 schema | 确认 API key 应写在 `agents.defaults.memorySearch.remote.apiKey`，不在 `auth-profiles.json` |
| 确认 `memory index` 报 `fetch failed` | 因为 remote.baseUrl 未设置，请求默认打到 `api.openai.com` |

**Schema 查阅结果**（MemorySearchSchema）：
```
agents.defaults.memorySearch:
  enabled: boolean
  provider: "openai" | "local" | "gemini" | "voyage" | "mistral" | "ollama"
  model: string
  remote:
    baseUrl: string (optional, 不设则用 provider 默认地址)
    apiKey: SecretInput (关键！API key 放这里)
    headers: object (optional)
    batch: object (optional)
  extraPaths: string[] (额外索引路径)
  sources: ["memory", "sessions"] (索引来源)
```

### 第二阶段：修复 memory search

| 操作 | 结果 |
|------|------|
| 备份 `openclaw.json` → `openclaw.json.bak.20260315_130646` | ✅ |
| 写入 `agents.defaults.memorySearch.remote.baseUrl` = 阿里云地址 | ✅ |
| 写入 `agents.defaults.memorySearch.remote.apiKey` = Qwen API Key | ✅ |
| 创建 `~/.openclaw/workspace/memory/` 目录 | ✅ |
| 重启 gateway | ✅ active |
| 运行 `memory index` | ✅ `Memory index updated (main)` |
| 验证 `memory status` | ✅ `Embeddings: ready`, `Indexed: 1/1 · 3 chunks` |

**API Key 脱敏**：`sk-d6475...b352`（写入 openclaw.json，not in auth-profiles.json）

### 第三阶段：扩展索引范围

发现 `sources: memory` 只索引 `memory/` 子目录，workspace 根目录的 AGENTS.md、AI_RULES.md 等文件未被索引。

写入 `agents.defaults.memorySearch.extraPaths = ["/home/xzy0626/.openclaw/workspace"]`，重建索引后：

```
Indexed: 62/62 files · 137 chunks
Vector dims: 1024
Embedding cache: 120 entries
```

### 第四阶段：创建 knowledge-notebook Skill

**设计原则**（仿 NotebookLM）：
1. 文档收录：任何来源 → 存入 `memory/` → 自动建立向量索引
2. 语义检索：`openclaw memory search` → 不是关键词匹配，是语义相似度
3. 带引用回答：搜索到相关内容才回答，标注 `📚 来源: [文件名]`
4. 摘要生成：多文档聚合 → 结构化要点提取
5. 私有部署：数据不出虚拟机

**Skill 文件**：`~/.openclaw/workspace/skills/knowledge-notebook.md`（205 行）

内容包含：
- 功能说明和 NotebookLM 对比表
- 文档收录流程（含标准 frontmatter 格式）
- 检索查询 SOP（先搜再答，无结果时明确告知）
- Embedding Provider 切换指南（Qwen/OpenAI/Ollama/Gemini 四种方案）
- 故障排查表
- 安全规则（知识库内容不外传、API Key 不显示等）

---

## 通用 Embedding 方案总结

| Provider | 模型 | 费用 | 推荐场景 | 切换命令 |
|----------|------|------|---------|---------|
| **Qwen（当前）** | text-embedding-v3 | ¥0.0007/千token | 中文主力 | 已配置 |
| OpenAI 原生 | text-embedding-3-small | $0.02/百万token | 英文文档为主时 | 改 baseUrl + apiKey |
| Ollama 本地 | nomic-embed-text | 免费 | 完全离线 | provider 改 ollama |
| Gemini | text-embedding-004 | 免费额度大 | 备用 | provider 改 gemini |

---

## 测试验收结果

| 测试项 | 结果 |
|--------|------|
| Gateway active | ✅ PASS |
| Memory provider: openai, model: text-embedding-v3 | ✅ PASS |
| 62/62 files · 137 chunks indexed | ✅ PASS |
| 英文搜索命中 embedding 配置文档（score: 0.722） | ✅ PASS |
| 中文内容正确向量化和检索（SELF_KNOWLEDGE.md 等） | ✅ PASS |
| knowledge-notebook.md skill 文件就位（205行） | ✅ PASS |
| Vector + FTS 双引擎 ready | ✅ PASS |

---

## 文件变更清单

| 操作 | 路径（虚拟机） | 说明 |
|------|-------------|------|
| 修改 | `~/.openclaw/openclaw.json` | `agents.defaults.memorySearch.remote` 写入 baseUrl + apiKey；`extraPaths` 写入 workspace 路径 |
| 新建 | `~/.openclaw/workspace/memory/` | 知识库文档目录 |
| 新建 | `~/.openclaw/workspace/memory/2026-03-15-note-openclaw-embedding-setup.md` | 示例知识文档（embedding 配置说明，UTF-8） |
| 新建 | `~/.openclaw/workspace/skills/knowledge-notebook.md` | NotebookLM 风格知识库 Skill（205行） |

---

## 安全审查

| 审查项 | 结论 |
|--------|------|
| C 盘操作 | ❌ 无 |
| API Key 泄露 | ✅ 已脱敏：`sk-d6475...b352` |
| 知识库内容外传 | ✅ memory/ 目录不纳入任何 GitHub 推送 |
| 危险命令 | ❌ 无 |

---

## GitHub 提交信息

```
logs: memory-search fix (remote.baseUrl) + knowledge-notebook skill + extraPaths index expansion
```
