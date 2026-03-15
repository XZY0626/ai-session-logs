# 2026-03-15 Openclaw 全面健康检查与修复

## 背景

用户反映龙虾版本已升级至 v2026.3.13，WorkBuddy 对整个 OpenClaw 架构进行了一次系统性检查和修复。

## 发现的问题

| # | 问题 | 严重度 | 状态 |
|---|------|--------|------|
| 1 | 二进制版本不一致（CLI 指向 /usr/bin/openclaw v2026.3.11，而非 ~/.local/bin v2026.3.13） | 高 | ✅ 修复 |
| 2 | Memory Search 启用但无 embedding provider，语义检索完全不可用 | 高 | ✅ 修复 |
| 3 | auth-profiles.json 格式错误（写了 `{"openai": {"apiKey": ...}}` 而非正确的 `{"version":1,"profiles":{...}}` 格式） | 高 | ✅ 修复 |
| 4 | 孤立 transcript 文件堆积（15个 .deleted/.reset + 6个 test- 文件） | 中 | ✅ 修复 |
| 5 | 缺少 VM 性能优化环境变量（NODE_COMPILE_CACHE、OPENCLAW_NO_RESPAWN） | 低 | ✅ 修复 |
| 6 | ~/.bashrc 未将 ~/.local/bin 置于 PATH 优先位 | 低 | ✅ 修复 |

## 修复操作

### 1. 性能优化变量写入 systemd service
在 `~/.config/systemd/user/openclaw-gateway.service` 添加：
```
Environment=NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
Environment=OPENCLAW_NO_RESPAWN=1
```
- 备份保存到原文件名 + `.bak.<timestamp>`

### 2. Memory Search 配置
使用 `openclaw config set` 写入：
- `agents.defaults.memorySearch.enabled = true`
- `agents.defaults.memorySearch.provider = openai`
- `agents.defaults.memorySearch.model = text-embedding-v3`

### 3. auth-profiles.json 正确格式写入
路径：`~/.openclaw/agents/main/agent/auth-profiles.json`
格式为 `AuthProfileStore` 标准结构：
```json
{
  "version": 1,
  "profiles": {
    "openai-default": {
      "type": "api_key",
      "provider": "openai",
      "key": "<Qwen API key (OpenAI 兼容接口)>"
    }
  }
}
```
Embedding 使用阿里云 Qwen text-embedding-v3，通过 OpenAI 兼容接口调用。

### 4. 孤立文件清理
使用 `find -delete` 清理 sessions 目录下的 `.deleted` / `.reset` / `test-*` 文件，从 25 个清到 10 个活跃文件。

### 5. PATH 修复
在 `~/.bashrc` 追加 `export PATH="$HOME/.local/bin:$PATH"`，确保 CLI 直接使用 v2026.3.13。

## 修复后验证状态

```
Gateway:              ✅ active (running) PID 14174
Memory Search:
  Provider:           ✅ openai (requested: openai)
  Model:              ✅ text-embedding-v3
  Vector index:       ✅ ready
  FTS index:          ✅ ready
性能变量:             ✅ NODE_COMPILE_CACHE, OPENCLAW_NO_RESPAWN 已写入 service
Feishu WebSocket:     ✅ ok
心跳 cron:            ✅ 每2小时运行，状态 ok
```

## 遗留事项

- `openclaw doctor` 的 cron legacy format 警告仍然存在（需要 gateway IPC 触发 `--fix`，纯 CLI 无法修复）——不影响功能，等下次 openclaw 版本更新自动处理
- Memory Search 当前 indexed 0/1 files（首次启用，待 gateway 后台建立索引）
