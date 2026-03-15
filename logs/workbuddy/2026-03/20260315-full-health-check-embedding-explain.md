# 2026-03-15 全面安全检查 + Embedding 机制说明 + Hunter/Healer 验证

## 操作摘要

本次会话主要完成：
1. 澄清 Qwen Embedding 与对话模型的协作机制
2. 确认 OpenRouter Hunter Alpha / Healer Alpha 配置正确并实测可用
3. 对 OpenClaw 所有配置和能力做一次全面安全检查

---

## 全面检查结果（2026-03-15 14:00）

### ✅ 正常项

| 检查项 | 状态 | 详情 |
|-------|------|------|
| openclaw-gateway 服务 | ✅ ACTIVE | 用户级 systemd，开机自启 |
| tailscale-serve | ✅ 正常（已退出型服务） | ExecStart 执行后正常退出，serve 后台运行中 |
| tailscaled 守护进程 | ✅ ACTIVE | 系统级服务 |
| Tailscale HTTPS 外部访问 | ✅ HTTP 200 | https://[脱敏].ts.net 可访问 |
| Memory Search | ✅ 65文件/141 chunks/Vector ready | Qwen API 正常 |
| Hunter Alpha API | ✅ HTTP 200 | OpenRouter 返回正常 |
| Healer Alpha API | ✅ HTTP 200 | OpenRouter 返回正常 |
| Qwen Embedding API | ✅ HTTP 200 | Dashscope 正常 |
| AGENTS.md | ✅ 存在 (201行) | v4 版本，含 L0 规则 |
| NODE_COMPILE_CACHE | ✅ 已配置 | systemd 环境变量 /var/tmp/openclaw-compile-cache |
| OPENCLAW_NO_RESPAWN | ✅ =1 | 已配置 |
| loginctl Linger | ✅ yes | 无需登录开机自启 |

### ⚠️ 说明项（非问题）

| 项目 | 说明 |
|------|------|
| tailscale-serve systemctl status 显示 inactive | 这是正常现象：该服务类型为 oneshot（执行后退出），启动时执行 `tailscale serve --bg 18789` 后进程退出，但 serve 配置持久生效，HTTPS 代理继续运行 |
| openclaw doctor 提示 NODE_COMPILE_CACHE not set | doctor 检查的是当前 shell 环境变量，而非 systemd 服务环境变量，实际 systemd 里已配置，无需处理 |
| 1 session lock file | 龙虾当前有活跃会话（pid 17840），属正常运行状态 |

### 已配置模型清单（22个）

- **阿里云 Qwen 系列**（8个）：qwen-max / qwen-plus / qwen-turbo / qwen-long / qwen-coder / qwq / qwen3 / qwen-vl
- **DeepSeek**（4个）：dashscope-r1 / dashscope-v3 / siliconflow-r1 / siliconflow-v3
- **StepFun**（2个）：step-2 / step-1
- **MiniMax**（4个）：MiniMax-M2.5 / M2.5-highspeed / M1 / Text-01
- **OpenRouter 免费**（2个）：hunter-alpha（1M上下文）/ healer-alpha（256k，多模态）
- **自定义**（2个）：aliyun-qwen-turbo / coding-qwen-turbo

### Skills 状态（8/55 ready）

- feishu-doc / feishu-drive / feishu-wiki / feishu-bitable / feishu-chat ✅
- knowledge-notebook ✅（本次新增）
- knowledge-ingest ✅（本次新增）
- weather ✅

---

## 技术说明：Embedding 与对话模型协作机制

### 时序图（简化）

```
用户发送消息
  ↓
[Step 1] Memory Search 触发
  - 用 Qwen text-embedding-v3 将用户消息转成 1024维向量
  - 在 SQLite 向量库（main.sqlite）中做 ANN（近似最近邻）查找
  - 返回 top-N 相关文档片段（score > 0.4）
  ↓
[Step 2] 上下文组装
  - 把检索到的片段作为 system context 注入
  - 格式：<memory>...相关内容...</memory>
  ↓
[Step 3] 对话模型推理
  - StepFun step-3.5-flash 收到 [system context + 用户消息]
  - 基于检索到的内容生成回答
  - 不依赖模型自身权重里的"记忆"，而是依赖检索到的原文
```

### 两者关系

- Embedding 模型只做"向量转换"，不生成文字
- 对话模型只做"理解+生成"，不做检索
- 完全解耦：换任意一方不影响另一方

---

## 本次新增/变更内容

| 类型 | 文件/配置 | 内容 |
|------|----------|------|
| 知识笔记 | memory/2026-03-15-embedding-provider-strategy.md | Embedding 通用方案（4种备选） |
| Skill | skills/knowledge-ingest.md | 知识投喂操作指南 |
| 验证 | Hunter/Healer Alpha API 实测 HTTP 200 | 两个免费模型均可用 |
