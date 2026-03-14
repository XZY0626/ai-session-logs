# OpenRouter 模型列表更新日志 v5.1（2026-03-14）

## 背景

用户反映聊天界面模型选择器中多个模型无法正常使用。对 `openclaw.json` 配置的全部 OpenRouter 模型进行了批量可用性测试，并据此更新了模型列表和 `model-selector-v5.js` 前端插件。

---

## 测试方法

通过 OpenRouter `/api/v1/chat/completions` 接口，对每个模型发送一条最小测试消息（`"Say 1"`），根据 HTTP 状态码和响应内容判断可用性：

- `200`：可用 ✅
- `400`：模型 ID 失效（Bad Request）
- `403`：区域封锁（Forbidden）
- `404`：模型已下线
- 返回非标准格式：解析异常

---

## 测试结果

| 模型 ID | 状态 | 原因 |
|---------|------|------|
| `openrouter/meta-llama/llama-3.3-70b-instruct` | ✅ 可用 | — |
| `openrouter/mistralai/mistral-large-2411` | ✅ 可用 | — |
| `openrouter/openai/gpt-4o` | 🚫 区域封锁 | HTTP 403，中国大陆不可用 |
| `openrouter/openai/gpt-4o-mini` | 🚫 区域封锁 | HTTP 403 |
| `openrouter/openai/o1` | 🚫 区域封锁 | HTTP 403 |
| `openrouter/openai/o3-mini` | 🚫 区域封锁 | HTTP 403 |
| `openrouter/anthropic/claude-3.5-sonnet` | 🚫 区域封锁 | HTTP 403 |
| `openrouter/anthropic/claude-3-opus` | 🚫 已下线 | HTTP 404，OpenRouter 已移除 |
| `openrouter/anthropic/claude-3-haiku` | 🚫 区域封锁 | HTTP 403 |
| `openrouter/google/gemini-2.0-flash-001` | 🚫 区域封锁 | HTTP 403 |
| `openrouter/google/gemini-2.0-pro-exp-02-05` | 🚫 ID 失效 | HTTP 400，模型 ID 已废弃 |
| `openrouter/stepfun/step-3.5-flash:free` | ⚠️ 解析异常 | 返回格式非标准，无法正常调用 |

> **备注**：OpenRouter 对中国大陆 IP 访问 OpenAI、Anthropic、Google 系模型均返回 403，这是 OpenRouter 的提供商级别区域限制，与自身 API Key 无关。

---

## 新增模型

本次通过推文推荐，发现 OpenRouter 上两个完全免费的高性能模型（均为 2026 年新发布）：

### Hunter Alpha

- **OpenRouter ID**：`openrouter/hunter-alpha`
- **参数量**：约 1 万亿（1T）
- **上下文长度**：1,000,000 tokens（1M）
- **特点**：专为 agentic 推理设计，长文档理解能力强
- **费用**：完全免费

### Healer Alpha

- **OpenRouter ID**：`openrouter/healer-alpha`
- **上下文长度**：262,144 tokens（262K）
- **特点**：全模态输入（文字 / 图片 / 音频 / 视频），理解多媒体内容
- **费用**：完全免费

两个模型均通过 API 测试（返回 200，内容正常），已确认可用。

---

## 变更内容

### `openclaw.json`（位于虚拟机 `~/.openclaw/openclaw.json`）

**移除**（失效/封锁模型，共 10 个）：
- `openrouter/openai/gpt-4o`
- `openrouter/openai/gpt-4o-mini`
- `openrouter/openai/o1`
- `openrouter/openai/o3-mini`
- `openrouter/anthropic/claude-3.5-sonnet`
- `openrouter/anthropic/claude-3-opus`
- `openrouter/anthropic/claude-3-haiku`
- `openrouter/google/gemini-2.0-flash-001`
- `openrouter/google/gemini-2.0-pro-exp-02-05`
- `openrouter/stepfun/step-3.5-flash:free`

**保留**（可用模型）：
- `openrouter/meta-llama/llama-3.3-70b-instruct`
- `openrouter/mistralai/mistral-large-2411`

**新增**：
- `openrouter/openrouter/hunter-alpha`
- `openrouter/openrouter/healer-alpha`

### `model-selector-v5.js` → v5.1（位于虚拟机 `~/.openclaw/plugins/`）

1. 失效模型全部标记 `status: 'unavailable'`，UI 显示为半透明 `🚫`，点击弹出提示"区域封锁或已下线"
2. OpenAI / Anthropic / Google 分组名称更新，添加 `⚠️区域封锁` 标注
3. `OpenRouter·免费` 分组顶部新增 Hunter Alpha 和 Healer Alpha，带 `★` 标记
4. `selectModel()` 函数新增 `unavailable` 状态处理分支

---

## 当前可用模型一览（2026-03-14）

| 显示名称 | 模型 ID | 分组 | 免费 |
|---------|--------|------|------|
| ★ Hunter Alpha (1T参数·1M上下文·推理) | `openrouter/openrouter/hunter-alpha` | OpenRouter·免费 | ✅ |
| ★ Healer Alpha (全模态·262K) | `openrouter/openrouter/healer-alpha` | OpenRouter·免费 | ✅ |
| Llama-3.3-70B-Instruct | `openrouter/meta-llama/llama-3.3-70b-instruct` | OpenRouter·Meta | 否 |
| Mistral-Large-2411 | `openrouter/mistralai/mistral-large-2411` | OpenRouter·Mistral | 否 |
| MiniMax-M1 / M2 / M2.5 / Text-01 | （见 MiniMax 分组） | OpenRouter·MiniMax | 部分免费 |

> MiniMax 系列模型（4个）在 v5 版本已配置，本次未重新测试，状态沿用 v5 的配置。

---

## 操作记录

```
# 确认 openclaw.json 中 OpenRouter provider 配置
# 通过脚本批量调用 OpenRouter API 测试各模型

# 修改 openclaw.json，移除失效模型，新增两个免费模型
ssh xzy0626@[vm-ip] "openclaw config reload"

# 更新 model-selector-v5.js，本地修改后 scp 上传
scp model-selector-v5.js xzy0626@[vm-ip]:~/.openclaw/plugins/

# 重启 openclaw-gateway 使配置生效
ssh xzy0626@[vm-ip] "systemctl --user restart openclaw-gateway"
# → 重启后令牌未变（gateway 做了热重载），无需重新登录
```

---

## 后续建议

1. **定期复测**：OpenRouter 区域封锁情况可能变化，可每隔 1-2 个月重新跑一次测试脚本
2. **关注 Hunter/Healer 状态**：两个模型目前免费但属于 alpha 阶段，后续可能转为付费或调整 ID
3. **MiniMax 系列补测**：本次未覆盖，建议下次一并测试
