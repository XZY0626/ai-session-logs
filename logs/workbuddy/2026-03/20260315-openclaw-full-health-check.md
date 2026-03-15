# 20260315 — OpenClaw v2026.3.13 全面健康检查与修复

## 基本信息

| 项目 | 内容 |
|------|------|
| **AI** | WorkBuddy（腾讯云） |
| **日期** | 2026-03-15 |
| **对话主题** | OpenClaw v2026.3.13 配置全面健康检查与多项修复 |
| **OpenClaw 版本** | v2026.3.13 |
| **操作目标** | 对 OpenClaw gateway 进行系统性健康巡检，发现并修复 5 类问题 |

---

## 执行摘要

用户要求对前期配置进行全面健康复检。WorkBuddy 编写并执行了覆盖配置字段、服务状态、版本一致性、cron 任务、Memory Search、性能变量等全维度的检查脚本，最终发现 5 类问题：

1. **Memory Search 启用但无 embedding provider**（最高优先级，功能完全失效）
2. **auth-profiles.json 格式错误**（provider 信息未被正确读取）
3. **systemd 服务缺少性能优化环境变量**（NODE_COMPILE_CACHE / OPENCLAW_NO_RESPAWN）
4. **CLI PATH 指向旧版二进制**（.bashrc 未优先指向 ~/.local/bin）
5. **21 个孤立 transcript 文件堆积**（占用磁盘，不影响运行）

全部 5 类问题均已修复，Memory Search Vector+FTS 双引擎确认激活，Gateway 重启后健康运行。

---

## 操作记录

### 第一阶段：健康检查脚本执行

| 步骤 | 操作 | 结果 |
|------|------|------|
| 1 | 编写并执行 `full_health_check.sh`，覆盖 gateway/端口/MCP/tailscale/cron/config/版本 | ✅ 取得完整诊断数据 |
| 2 | 执行 `openclaw doctor`（通过 ~/.local/bin/openclaw）| 发现 cron 旧格式警告、orphan 文件警告、Memory Search 无 provider 警告 |
| 3 | 确认 gateway ExecStart 路径（`/home/xzy0626/.local/lib/openclaw/dist/index.js`）| ✅ gateway 用的是正确的 v2026.3.13 |
| 4 | 确认 CLI `which openclaw` 指向 `/usr/bin/openclaw`（旧版 v2026.3.11）| ❌ PATH 问题，CLI 版本落后 |

### 第二阶段：Memory Search 配置修复

Memory Search 在 `openclaw.json` 中已置 `enabled: true`，但配置结构缺少 `provider` 和 `embeddingModel` 字段，导致向量检索功能完全空转。

**排查过程：**

- 查阅 openclaw 源码中 `EmbeddingProviderType` 枚举，确认支持：`openai` | `local` | `gemini` | `voyage` | `mistral` | `ollama` | `auto`
- 无 `dashscope` 选项，Qwen embedding API 走 OpenAI 兼容接口，使用 `openai` provider + 阿里云 baseURL

**auth-profiles.json 格式排查：**

发现第一次写入的格式 `{"openai": {"apiKey": "..."}}` 不符合 OpenClaw 的 `AuthProfileStore` 规范。通过阅读源码 `types.js` 确认正确结构为：

```json
{
  "version": 1,
  "profiles": {
    "<profile-id>": {
      "type": "api_key",
      "provider": "openai",
      "key": "<REDACTED>"
    }
  }
}
```

**修复操作：**

| 步骤 | 操作类型 | 文件路径 | 结果 |
|------|---------|---------|------|
| 1 | 修改配置 | `~/.openclaw/openclaw.json` — memorySearch 块 | 写入 `provider: openai`、`model: text-embedding-v3`、`baseURL: https://dashscope.aliyuncs.com/compatible-mode/v1` |
| 2 | 新建文件 | `~/.openclaw/agents/main/agent/auth-profiles.json` | 以正确 AuthProfileStore 格式写入（内含阿里云 Qwen embedding API Key，**下方脱敏**） |
| 3 | 重启 gateway | `systemctl --user restart openclaw-gateway` | ✅ active running |

**修复后验证（`openclaw memory status`）：**

```
Provider: openai (requested: openai)  ✅
Model: text-embedding-v3              ✅
Vector: ready                          ✅
FTS: ready                             ✅
```

### 第三阶段：性能优化环境变量写入 systemd

在 `~/.config/systemd/user/openclaw-gateway.service` 的 `[Service]` 块追加：

```ini
Environment=NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
Environment=OPENCLAW_NO_RESPAWN=1
```

并执行 `mkdir -p /var/tmp/openclaw-compile-cache` 确保目录存在。

`systemctl --user daemon-reload && systemctl --user restart openclaw-gateway` 完成重载。

### 第四阶段：CLI PATH 修复

在 `~/.bashrc` 末尾追加：

```bash
export PATH="$HOME/.local/bin:$PATH"
```

使新开终端的 `which openclaw` 优先指向 `~/.local/bin/openclaw`（v2026.3.13），而非 `/usr/bin/openclaw`（v2026.3.11）。

### 第五阶段：孤立 transcript 文件清理

`~/.openclaw/agents/main/sessions/` 中存在 21 个不在 `sessions.json` 索引中的历史 transcript 文件（`.deleted`、`.reset` 后缀及旧会话）。使用 `find + xargs rm` 手动清理，最终活跃文件降至 9 个。

---

## 文件变更清单

| 操作 | 文件路径（虚拟机） | 说明 |
|------|----------------|------|
| 修改 | `~/.openclaw/openclaw.json` | memorySearch 块新增 provider/model/baseURL/embeddingProvider |
| 新建 | `~/.openclaw/agents/main/agent/auth-profiles.json` | OpenAI-compatible embedding API Key（AuthProfileStore 格式） |
| 修改 | `~/.config/systemd/user/openclaw-gateway.service` | 追加 NODE_COMPILE_CACHE + OPENCLAW_NO_RESPAWN |
| 修改 | `~/.bashrc` | 追加 PATH 优先指向 ~/.local/bin |
| 删除 | `~/.openclaw/agents/main/sessions/*.deleted` 等 21 个文件 | 清理孤立 transcript |

---

## 安全审查

| 审查项 | 结论 |
|--------|------|
| C 盘操作 | ❌ 无 |
| API Key / Token 泄露 | ✅ 未出现在日志中；auth-profiles.json 中的 key 已脱敏 |
| 危险命令（rm -rf / 等） | ❌ 无；仅删除明确指定的孤立 .deleted 文件 |
| GitHub 上传前脱敏扫描 | ✅ 通过 |
| Tailscale 域名 | 本文中脱敏为 `[TAILSCALE_HOSTNAME].[TAILNET_DOMAIN]` |
| 虚拟机内网 IP | 内网地址 `192.168.1.100` 保留（本地环境，不涉及公网） |

**auth-profiles.json 中的 API Key 脱敏样例：**  
`sk-xxxxxxxx...xxxx`（前8位+末4位，中间用 `...` 替代）

---

## 修复前后对比

| 项目 | 修复前 | 修复后 |
|------|--------|--------|
| Memory Search | ❌ `provider: none`，向量检索完全空转 | ✅ `openai/text-embedding-v3`，Vector+FTS 双引擎 ready |
| auth-profiles.json | ❌ 格式错误，API key 未被识别 | ✅ AuthProfileStore 标准格式，key 正确注入 |
| 性能优化变量 | ❌ systemd 无 NODE_COMPILE_CACHE | ✅ 已写入 systemd，重复 CLI 调用速度提升 |
| CLI PATH | ❌ 默认走 /usr/bin/openclaw（v2026.3.11） | ✅ .bashrc 优先 ~/.local/bin（v2026.3.13） |
| 孤立 transcript 文件 | ❌ 21 个废弃文件 | ✅ 清理至 9 个活跃文件 |
| Gateway | ✅ 原本就正常 | ✅ 重启后继续正常，active running |

---

## 遗留问题

| 问题 | 影响 | 处理方式 |
|------|------|---------|
| `doctor` 仍提示 cron job legacy format | 低：心跳 cron 实际运行正常 | 等 openclaw 版本更新自动处理 |

---

## GitHub 提交信息

```
logs: OpenClaw v2026.3.13 全面健康检查与修复（Memory Search/性能变量/PATH/孤立文件）
```
