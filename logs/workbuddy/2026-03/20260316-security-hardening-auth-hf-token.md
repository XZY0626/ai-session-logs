# 对话日志

- **AI**: WorkBuddy
- **对话窗口ID**: 5d41f0f03ccc4ccfa2ea9e250679db96
- **日期**: 2026-03-16
- **用户**: XZY0626
- **主题**: OpenClaw 安全加固 — 删除 .pre-disable-auth + HF Token 最小权限化

## 执行摘要

本次对话完成了两项安全加固操作：
1. **删除 `.pre-disable-auth` 标志文件**：清除了 gateway 启动时可绕过认证的遗留标志文件，彻底消除认证旁路风险。
2. **HuggingFace Token 最小权限化**：将 HF Token 从读写 Token 降级为只读 Token，降低泄露后损失半径。
3. **AGENTS.md 技能注册表补全**：新增 `scrapling` 技能条目，使技能列表与实际安装状态保持一致。

## 操作记录

| 序号 | 操作类型 | 目标路径/资源 | 操作描述 | 结果 |
|------|---------|-------------|---------|------|
| 1 | 删除文件 | `~/.openclaw/.pre-disable-auth`（VM） | 删除绕过认证的遗留标志文件 | ✅ 成功 |
| 2 | 更新凭证 | `/home/xzy0626/.huggingface/token` | 替换 HF Token 为只读权限 Token | ✅ 成功（见20260316-hf-token-update-readonly.md）|
| 3 | 修改文件 | `workspace/AGENTS.md` | 新增 scrapling 技能条目 | ✅ 成功 |

## 文件变更清单

- 修改：`workspace/AGENTS.md` — Skills 表新增 scrapling 技能条目
- 删除：`~/.openclaw/.pre-disable-auth`（VM 端，标志文件）
- 更新：`/home/xzy0626/.huggingface/token`（VM 端，HF Token 内容）

## 安全审查

- 凭证泄露检查：✅ 通过（Token 已脱敏，未明文记录）
- 危险操作检查：✅ 无危险操作
- C盘操作：✅ 未涉及
- 敏感路径：⚠️ 涉及 VM 上的 `~/.huggingface/token`（仅更新，权限 600）

## `.pre-disable-auth` 影响说明

**该文件删除前的安全风险：**
- 该文件是 OpenClaw 遗留的"标志文件"（flag file）机制
- 当 `~/.openclaw/.pre-disable-auth` 存在时，gateway 启动时会跳过认证检查
- 任何能访问 gateway 端口的人（或进程）都可以在**无需任何凭证**的情况下直接调用所有 API
- 这正是 2026 年 2 月 OpenClaw 安全危机（CVE-2026-25253 等）的核心诱因之一
- 即使配置了 `gateway.bind=loopback`，在 VM 本机内部的进程（如恶意 Skill、被注入的会话）仍可利用此漏洞

**删除后的效果：**
- Gateway 在启动时强制执行认证流程
- 任何连接请求都必须通过 `gateway.auth.token` 或 `gateway.auth.mode` 规定的凭证
- 与 `gateway.bind=loopback` + Tailscale HTTPS 双重防护形成完整的纵深防御体系

## 待办/遗留问题

- 建议定期检查 `~/.openclaw/` 目录下是否有异常标志文件（如 `.pre-disable-auth` 重新出现）
- HF Token 轮换建议每 3-6 个月执行一次
- 网络不可达问题仍需配置代理/镜像源

## GitHub提交

- 仓库：ai-session-logs（日志）、openclaw-config（AGENTS.md）
- Commit：`security: 删除 .pre-disable-auth 认证旁路文件 + HF Token 最小权限化 + 补全技能注册表`
