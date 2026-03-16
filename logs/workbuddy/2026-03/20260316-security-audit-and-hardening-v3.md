# [2026-03-16] 安全自查核实与防护加固

**时间**：2026-03-16（CST）  
**执行者**：WorkBuddy（宿主机）  
**操作类型**：安全核查 / 防护加固 / 文档整理

---

## 背景

主人让龙虾进行了安全自查，生成了《安全评估报告》和《安全改进方案》，并委托 WorkBuddy 核实问题真实性后执行改进。

---

## 核查结果汇总

### ✅ 评估报告合规项 — 全部属实

| 项目 | 核查方式 | 结论 |
|------|----------|------|
| HF Token 权限 600 | `stat -c %a ~/.huggingface/token` | ✅ 属实 |
| Gateway active（127.0.0.1 绑定） | `ss -tlnp` 实测端口 | ✅ 属实 |
| L0 硬性拒绝 7 条 | `head AGENTS.md` | ✅ 属实（已升级到 v5，8条） |
| openclaw.json 备份存在 | `ls ~/.openclaw/*.bak*` | ✅ 且有 7 份备份 + 2 套完整备份目录 |

### ❗ 改进方案 P0 问题 — 均属实并已修复

| P0 问题 | 属实？ | 修复操作 |
|---------|-------|---------|
| requirements.txt 无 sha256 hash | ✅ 属实（grep sha256 = 0） | 生成 `requirements.lock.txt`（3407条 hash） |
| AGENTS.md checksum 只是软标记 | ✅ 属实（.agents_checksum 不存在） | 建立 `verify_agents.sh` + `.agents_checksum`，验证通过 |

### ✅ Guardian Core 安全架构

**关于"类360杀毒软件架构"的位置问题**：架构已在 2026-03-16 早些时候部署到虚拟机 `workspace/skills/guardian-core.md`，但 README 未体现。本次已补充。

---

## 执行操作

### 1. AGENTS.md sha256 校验机制建立

- 文件：`~/.openclaw/workspace/.agents_checksum` + `verify_agents.sh`
- 实际 sha256：`1db79f0176c97208f1348789b75902d9d5818d25b0ff27924458746a22ff5f2a`
- 验证结果：`[OK] AGENTS.md integrity verified (sha256 match)`
- 用法：修改 AGENTS.md 后执行 `bash verify_agents.sh --update`

### 2. VoxBridge requirements hash 锁

- 文件：`~/.openclaw/workspace/VoxBridge/requirements.lock.txt`
- 使用工具：`pip-compile --generate-hashes`（在 venv_web 虚拟环境中执行）
- 生成条目数：**3407 条 sha256 hash**
- 一个注意事项：setuptools 未能固定（PEP 668 限制），pip 会发 WARNING，但不影响主要依赖的校验

### 3. README 去重整理

去除的重复内容：
- **学术搜索工具**：两节内容合并，保留详细版，"独立工具（Scripts）"改为摘要+引用
- **MCP 工具**：简化"各组件详解"中的描述，指向"能力全景"的完整版

新增内容：
- **安全防护体系（Guardian Core）**：独立章节，包含 G1-G8 架构图、配套机制表、待完成项
- 文件结构补充 `verify_agents.sh`、`.agents_checksum`、`security/` 目录、新增 skills
- 已落地能力表新增 Guardian Core、sha256 校验、requirements hash 锁条目
- 版本标记升级到安全加固 v3.0

---

## Git 记录

- Commit: `8cd41d3` — security: Guardian Core v1.0 + AGENTS.md sha256 checksum + VoxBridge requirements.lock.txt + README 去重整理
- 4 files changed, 158 insertions(+), 88 deletions(-)
- 推送到：`XZY0626/openclaw-config main`

---

_日志由 WorkBuddy 生成，已脱敏_
