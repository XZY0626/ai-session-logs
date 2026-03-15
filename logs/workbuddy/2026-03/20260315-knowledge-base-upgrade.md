# [2026-03-15] OpenClaw 知识库架构升级 + 国外方案评估

**时间**：2026-03-15（Asia/Shanghai）
**执行者**：WorkBuddy（宿主机侧）
**相关 commit**：workspace `f2c724e`

---

## 任务描述

评估国外 OpenClaw 培养方案的三个方向，决定采纳/不采纳，并实施采纳的方案：
1. 持久化记忆/知识库（RAG）
2. 工作流编排层（n8n/Webhook）
3. n8n 重复项（同上）

---

## 评估结论

### 方向1：持久化记忆/知识库（RAG）
**决策：采纳轻量替代方案**

原因：
- 虚拟机 4GB 内存（今天刚 OOM），无法运行 embedding 服务
- 现有 memory/ 目录只有按日期存的流水账，无法快速检索
- 用结构化 YAML/Markdown 知识库文件完全满足当前需求

**实施**：新建 `KNOWLEDGE_BASE.md`（7个 Section，141行），涵盖：
- 系统架构快照（宿主机/虚拟机/Tailscale）
- 项目状态（VoxBridge 当前进度）
- 规则合规状态
- 已知问题与待办（含 n8n 作为"未来选项"）
- 操作历史（最近10条）
- 常用命令速查
- Memory 写入规范

### 方向2&3：n8n 工作流编排
**决策：暂不实施，记录为未来选项**

原因：
- 虚拟机内存不足以运行 n8n 容器
- 现有 exec allowlist 已提供安全隔离
- VoxBridge Day 3 阶段无实际 webhook 需求
- 已在 KNOWLEDGE_BASE.md Section 4 记录为 P3 待办

---

## 执行过程

### 1. 创建 KNOWLEDGE_BASE.md
- 路径：`workspace/KNOWLEDGE_BASE.md`
- 内容：7个结构化 Section，状态快照（非流水账）
- 用途：开话前快速定位当前状态，替代翻日志

### 2. 更新 AGENTS.md → v4.1
- Startup Checklist 第5步新增：读取 KNOWLEDGE_BASE.md（优先于读日志）
- 末尾新增"📚 KNOWLEDGE BASE 维护规则"Section
- 说明何时更新、写什么、不写什么

### 3. 三端验证（8项全绿）
| 检查项 | 结果 |
|--------|------|
| AGENTS.md v4 版本标记 | ✅ |
| KNOWLEDGE_BASE 规则注入 | ✅ 4处引用 |
| KNOWLEDGE_BASE.md 7个Section | ✅ |
| L0 硬性拒绝规则 | ✅ |
| AI_RULES.md L0 层 | ✅ |
| github-sync pull-rebase 规则 | ✅ 4处 |
| workspace 软链接（logs/INDEX.md） | ✅ |
| openclaw-gateway + tailscale | ✅ 均运行中 |

### 4. 脱敏扫描
- KNOWLEDGE_BASE.md：✅ 无敏感信息（IP 脱敏为 192.168.x.x）
- AGENTS.md：✅ 无敏感信息
- github-sync.md：⚠️ 误报（`192.168.x.x` 是规则描述文本，非真实 IP）→ 确认为误报，无需处理

---

## 结果

- workspace commit `f2c724e`
- KNOWLEDGE_BASE.md（新文件）：141行，7个 Section
- AGENTS.md v4.1：startup checklist 更新，新增 KB 维护规则
- 三端验证：8/8 全绿
- 安全检测：通过

---

## 遗留问题

- scrapling 技能（龙虾自装）未加入 workspace git，待评估是否需要纳入版本控制
- n8n 部署已记录为 P3 未来选项，等 VoxBridge 有实际 webhook 需求时启动
