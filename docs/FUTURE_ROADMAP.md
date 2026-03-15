# FUTURE_ROADMAP.md — 未来技术方案储备库

> **存放位置**：`ai-session-logs/docs/FUTURE_ROADMAP.md`
> **GitHub 地址**：https://github.com/XZY0626/ai-session-logs/blob/main/docs/FUTURE_ROADMAP.md
> **维护规则**：由 WorkBuddy 维护，每次评估后更新状态；龙虾可读取但不主动修改。
> **最后更新**：2026-03-15

---

## 说明

这个文件记录所有**现阶段无法或不必要实施、但未来有价值**的技术方案。
每个方案标注触发条件，到达触发条件时再实施，避免过早引入不必要的复杂度。

状态标签：
- `📋 待评估` — 已记录，尚未详细分析
- `🔍 已评估·暂不实施` — 分析过，不满足触发条件
- `🟡 条件满足·可实施` — 触发条件已满足，可以开始部署
- `✅ 已实施` — 已完成部署

---

## 方案一：n8n 工作流编排层

| 字段 | 内容 |
|------|------|
| **状态** | 🔍 已评估·暂不实施 |
| **评估日期** | 2026-03-15 |
| **评估结论** | 现阶段不需要，原因见下方 |

### 是什么

n8n 是一个开源的可视化工作流编排工具，支持 Webhook / HTTP 请求 / 定时触发，可以作为 AI Agent 与外部 API 之间的"安全隔离层"。

国外跑通的典型模式：
```
OpenClaw → 触发 Webhook → n8n → 执行第三方 API 操作（GitHub / Slack / 数据库）
```

好处：
- 避免在 Agent 里硬编码 API Key
- 可视化操作历史，出错后容易溯源
- 可以对 Agent 的操作范围做细粒度限制（白名单 endpoint）

### 当前不实施的原因

1. **内存不够**：虚拟机只有 4GB，n8n（Node.js）至少占用 400-600MB，今天刚 OOM 过
2. **已有替代方案**：龙虾的 `exec allowlist` 已做了安全隔离，当前场景足够
3. **没有实际触发需求**：VoxBridge 还在本地开发阶段，没有需要编排的外部 API 调用

### 触发条件（满足任一即可考虑实施）

- [ ] 虚拟机内存升级到 8GB+
- [ ] VoxBridge 需要对接第三方 API（如云存储、推送通知、监控报警）
- [ ] 需要多个 Agent 协作触发同一工作流

### 部署方案（到时候直接用）

```bash
# Docker 方式（推荐，内存约 400MB）
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  --restart unless-stopped \
  n8nio/n8n

# 配置 tailscale serve 暴露
sudo tailscale serve --bg 5678

# 访问地址
# https://xzy0626-vmware-virtual-platform.tail6f9a39.ts.net:5678
```

参考资料：
- 官方文档：https://docs.n8n.io/
- 国外 Claude + n8n 集成案例：https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.anthropic/

---

## 方案二：RAG 向量检索知识库

| 字段 | 内容 |
|------|------|
| **状态** | 🔍 已评估·暂不实施 |
| **评估日期** | 2026-03-15 |
| **评估结论** | 现阶段用 KNOWLEDGE_BASE.md 轻量替代，效果足够 |

### 是什么

RAG（Retrieval-Augmented Generation）是把大量文档向量化存入数据库，Agent 提问时先语义检索最相关片段，再拼入 prompt。适合：
- 个人知识库（拖拽 URL/文章→可搜索）
- 语义记忆搜索（基于向量检索历史对话）
- 个人 CRM（自动追踪联系人上下文）

国外典型方案：`Chroma / Qdrant + LlamaIndex / LangChain + OpenClaw`

### 当前不实施的原因

1. **内存和 CPU 成本高**：embedding 服务（如 `nomic-embed-text`）即使轻量版也要 1GB+，虚拟机不够
2. **内容量还没到"记不住"的临界点**：目前日志总量 <50 个文件，结构化 KNOWLEDGE_BASE.md 完全够用
3. **冷启动成本高**：需要把历史日志全部向量化入库，工程量不小

### 触发条件（满足任一即可考虑实施）

- [ ] 虚拟机内存升级到 8GB+，或有独立服务器可用
- [ ] ai-session-logs 日志文件超过 200 个，手动检索变得困难
- [ ] 需要跨项目语义搜索（"我之前在哪个项目里解决过 XXX 问题？"）
- [ ] VoxBridge 有大量技术文档需要 Agent 随时检索

### 推荐部署方案（到时候直接用）

**轻量路线（内存 <2GB）：**
```bash
# 使用 Ollama 本地 embedding + ChromaDB
pip install chromadb langchain-community sentence-transformers

# 向量化脚本示例
python3 index_logs.py --source ~/ai-session-logs/logs --db ~/chroma_db
```

**更轻量的替代（0 内存开销）：**
```bash
# 用 ripgrep 做语义近似检索（不是真 RAG，但实用）
rg -l "关键词" ~/ai-session-logs/logs/
```

参考资料：
- ChromaDB：https://docs.trychroma.com/
- LlamaIndex 本地部署：https://docs.llamaindex.ai/
- Ollama embedding 模型：`nomic-embed-text`（最轻量，512MB）

---

## 方案三：个人 CRM（联系人追踪）

| 字段 | 内容 |
|------|------|
| **状态** | 📋 待评估 |
| **评估日期** | — |
| **评估结论** | 尚未评估 |

### 是什么

国外"第二大脑"方向的热门实践：Agent 自动从对话中提取联系人信息（姓名、公司、上次沟通内容、待跟进事项），存入结构化数据库，下次提到这个人时自动带入上下文。

### 触发条件

- [ ] 有明确的"需要追踪联系人"的业务需求
- [ ] VoxBridge 进入商业化阶段（需要追踪客户/合作伙伴）

---

## 方案四：多 Agent 协作框架

| 字段 | 内容 |
|------|------|
| **状态** | 📋 待评估 |
| **评估日期** | — |
| **评估结论** | 尚未评估 |

### 是什么

让多个 AI Agent 分工协作：
- 一个 Agent 专门做代码（龙虾）
- 一个 Agent 专门做文档/写作（WorkBuddy）
- 一个 Agent 专门做测试/审查
- 协调 Agent（Orchestrator）分配任务、汇总结果

国外典型方案：`AutoGen / CrewAI + OpenClaw`

### 触发条件

- [ ] VoxBridge 进入多人协作开发阶段
- [ ] 单个 Agent 的上下文窗口不够用（项目复杂度大幅提升）

---

## 方案五：语音交互入口

| 字段 | 内容 |
|------|------|
| **状态** | 📋 待评估 |
| **评估日期** | — |
| **评估结论** | 尚未评估 |

### 是什么

给 OpenClaw 加一个语音输入/输出层：
- STT（语音→文字）：faster-whisper（本地，已在 VoxBridge 里用过）
- TTS（文字→语音）：Edge-TTS（本地，已在 VoxBridge 里用过）
- 对接方式：OpenClaw 的 custom input 接口 / 飞书语音消息

好处：解放双手，走路/开车时也能跟 Agent 交互。

### 触发条件

- [ ] VoxBridge STT/TTS pipeline 稳定可用
- [ ] 有实际的"解放双手"使用场景需求

---

## 更新记录

| 日期 | 操作 | 操作者 |
|------|------|--------|
| 2026-03-15 | 初始创建，整理5个未来方案 | WorkBuddy |
