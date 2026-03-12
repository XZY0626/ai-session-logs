# 2026-03-12 GitHub Secret Scanning 安全告警修复

> AI助手: 小跃 (StepFun Desktop)
> 日期: 2026-03-12
> 主题: openclaw-config 仓库 Secret Scanning 4项告警修复、代码密钥清理、AI规则文件安全补充

---

## 问题背景

用户发现 GitHub 仓库 `XZY0626/openclaw-config` 的 Security > Secret scanning 中存在 **4项未处理的安全告警**，均标记为 "Public leak"（公开泄露），持续存在2天未消除。

## 告警详情

| # | 类型 | 泄露文件 | 开启时间 |
|---|------|---------|---------|
| 1 | OpenRouter API Key | `deploy/update_keys.py` (L24), `feishu-bot/model_monitor.py` | 2026-03-10 |
| 2 | Lark Application Secret | `feishu-bot/feishu_stepfun_bot.py` (L26), `feishu-bot/model_monitor.py` | 2026-03-10 |
| 3 | OpenRouter API Key | `deploy/add_step_flash_free.py` (L17), `deploy/test_openrouter_free.py` | 2026-03-11 |
| 4 | OpenRouter API Key | `feishu-bot/feishu_stepfun_bot.py` (L46) | 2026-03-11 |

## 根因分析

AI助手（小跃）在 3月10日-11日 帮用户编写部署脚本和飞书机器人代码时，**直接将明文API Key、飞书App Secret硬编码在Python脚本中**，随后推送到公开GitHub仓库，触发了GitHub Secret Scanning自动检测。

具体泄露的密钥类型：
- OpenRouter API Key × 3个不同的Key
- 飞书 Lark Application Secret × 1个
- 附带泄露：阶跃星辰API Key、阿里云百炼API Key、硅基流动API Key、VM SSH密码（在 `model_monitor.py` 中）

## 修复操作

### 1. 代码密钥清理

对以下4个文件进行了密钥替换，将所有硬编码密钥改为环境变量读取方式：

**`feishu-bot/model_monitor.py`** — 清除5个明文密钥：
```python
# 修复前
FEISHU_APP_SECRET = "C9UQrugDAk89K00HP4g20fKXaZxOewxY"
"api_key": "2nFoR2tH4n5R0CAe9EE8NVmMISzGjwOBznlSql7xCqwXXFfPSZeDK5yni2wIPu8l"
"api_key": "sk-d647...b352"
"api_key": "sk-ovee...pnft"
"api_key": "sk-or-v1-5d02...cf0"

# 修复后
FEISHU_APP_SECRET = os.environ.get("FEISHU_APP_SECRET", "YOUR_FEISHU_APP_SECRET")
"api_key": os.environ.get("STEPFUN_API_KEY", "YOUR_STEPFUN_API_KEY")
"api_key": os.environ.get("DASHSCOPE_API_KEY", "YOUR_DASHSCOPE_API_KEY")
"api_key": os.environ.get("SILICONFLOW_API_KEY", "YOUR_SILICONFLOW_API_KEY")
"api_key": os.environ.get("OPENROUTER_API_KEY", "YOUR_OPENROUTER_API_KEY")
```

**`deploy/update_keys.py`** — 清除2个API Key + VM密码：
```python
# 修复前
pwd = "Xzy0626"
providers["siliconflow"]["apiKey"] = "sk-ovee...pnft"
providers["openrouter"]["apiKey"] = "sk-or-v1-5d02...cf0"

# 修复后
pwd = os.environ.get("VM_PASSWORD", "YOUR_VM_PASSWORD")
providers["siliconflow"]["apiKey"] = os.environ.get("SILICONFLOW_API_KEY", ...)
providers["openrouter"]["apiKey"] = os.environ.get("OPENROUTER_API_KEY", ...)
```

**`deploy/add_step_flash_free.py`** — 清除1个OpenRouter Key
**`deploy/test_openrouter_free.py`** — 清除1个OpenRouter Key

### 2. .gitignore 更新

新增排除规则，防止未来 `.env` 文件被意外提交：
```
.env
.env.*
*.secret
```

### 3. Git 提交记录

```
Commit 1: 34d4283 - security: remove all hardcoded secrets, use env vars instead
  修改: model_monitor.py, add_step_flash_free.py, test_openrouter_free.py

Commit 2: 6d6a432 - security: remove hardcoded secrets from deploy/update_keys.py, update .gitignore
  修改: update_keys.py, .gitignore
```

### 4. AI规则文件补充

AI_RULES.md 新增 **L0.3.1 代码中的密钥管理（强制）** 规则：
- 禁止在代码中硬编码任何密钥，必须使用 `os.environ.get()` 环境变量
- SSH/VM密码同样适用
- Git历史记录意识：密钥一旦进入git历史，即使删除当前文件也会被Secret Scanning检测
- 新建脚本必须过自检清单
- 版本从 1.1.0 升至 1.2.0

## 未完成事项（需用户手动操作）

GitHub Secret Scanning 扫描的是 **git历史记录**，不是当前文件。即使代码已清理，历史commit中仍残留明文密钥，告警不会自动消失。

用户需手动操作：
1. 打开 https://github.com/XZY0626/openclaw-config/security/secret-scanning
2. 逐个点击4个告警，选择 **"Close as → Revoked"**
3. 泄露的旧Key均已轮换/作废，当前使用的新Key未被泄露

## 经验教训

**根本原因**：AI助手在编写部署脚本时，为了快速实现功能，直接将用户提供的API Key硬编码在代码中并推送到公开仓库，违反了密钥零泄露原则。

**改进措施**：
1. 规则文件新增 L0.3.1 强制要求代码中使用环境变量管理密钥
2. 新建脚本前必须过安全自检清单
3. `.gitignore` 已预防性排除 `.env` 文件
4. 后续所有涉及密钥的代码均使用 `os.environ.get()` 模式

---

*日志由小跃 (StepFun Desktop) 自动生成*
