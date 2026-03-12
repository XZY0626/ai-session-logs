# 2026-03-12 OpenRouter地区封锁修复与三端联调

> AI助手: 小跃 (StepFun Desktop)
> 日期: 2026-03-12
> 主题: OpenRouter IP地区封锁诊断与修复、虚拟机代理配置、三端联调测试

---

## 问题背景

用户在使用 learn.deeplearning.ai 时遇到客户端异常，同时 OpenClaw 通过 OpenRouter 调用模型持续失败。经诊断发现多个关联问题需要系统性解决。

## 问题诊断

### 1. OpenRouter API Key 失效
- 旧 Key (`sk-or-v1-8ee5fbe...`) 返回 `401 User not found`
- 原因: 账户或Key被禁用/过期
- 解决: 用户生成了新 Key (`sk-or-v1-6dfe2ce...c60`)

### 2. OpenRouter 地区限制 (核心问题)
- 新 Key 验证通过，但调用 OpenAI/Anthropic/Google 模型返回 `403 This model is not available in your region`
- 诊断过程:
  - 检测到宿主机有**两个 Clash 进程**:
    - `Clash for Windows` (PID 20280, 端口 7890) — 代理功能不正常
    - `闪电加速器 Clash` (PID 10504, 端口 33210) — 正常工作
  - 通过 Clash API (`http://127.0.0.1:33212/configs`) 确认配置
  - 初始节点为**香港/台湾**，导致 OpenAI 等模型地区受限
  - 通过 Clash API 切换到**美国洛杉矶节点** (出口IP: `66.92.207.78`)
  - 切换后全部 9 个模型测试通过

### 3. Clash allow-lan 未开启
- 虚拟机无法通过宿主机代理上网
- 通过 Clash API PATCH `/configs` 开启 `allow-lan: true`
- 端口 33210 从 `127.0.0.1` 变为 `0.0.0.0` 监听

### 4. 虚拟机代理配置错误
- HTTPS代理 IP 多打了一个 `3`，写成了 `192.168**3**.1.11`
- HTTP代理端口填的 `8080`，实际应为 `33210`
- 端口 7890 虽然在监听但属于 Clash for Windows，代理功能不正常
- 修正为: HTTP/HTTPS 均指向 `192.168.1.11:33210`

## 操作步骤

### 宿主机操作

1. **确认 Clash 端口归属**
   ```
   PID 10504 (闪电加速器) -> 端口 33210 (HTTP), 33211 (SOCKS), 33212 (API)
   PID 20280 (Clash for Windows) -> 端口 7890 (不可用)
   ```

2. **开启 Clash LAN 访问**
   ```
   PATCH http://127.0.0.1:33212/configs
   Body: {"allow-lan": true}
   Result: 204 No Content (成功)
   ```

3. **切换到美国节点**
   ```
   PUT http://127.0.0.1:33212/proxies/GLOBAL
   Body: {"name": "Lv2 美国 01 [1.5]"}
   验证: 出口IP 66.92.207.78 (Los Angeles, CA, US)
   ```

4. **验证 OpenRouter API Key**
   ```
   GET https://openrouter.ai/api/v1/auth/key
   Authorization: Bearer sk-or-v1-6dfe...c60
   Result: 200 OK, is_free_tier: false, usage: 0.0002283
   ```

5. **模型可用性测试 (美国节点)**
   | 模型 | 状态 |
   |------|------|
   | openai/gpt-4o-mini | ✅ OK |
   | openai/gpt-4o | ✅ OK |
   | anthropic/claude-3.5-haiku | ✅ OK |
   | anthropic/claude-sonnet-4 | ✅ OK |
   | google/gemini-2.0-flash-001 | ✅ OK |
   | deepseek/deepseek-chat | ✅ OK |
   | qwen/qwen-2.5-72b-instruct | ✅ OK |
   | microsoft/phi-4 | ✅ OK |
   | nvidia/llama-3.1-nemotron-70b-instruct | ✅ OK |

6. **更新飞书机器人 API Key**
   - 文件: `feishu_bot/feishu_stepfun_bot.py`
   - 替换所有 OpenRouter 模型的 `api_key` 为新 Key
   - 涉及模型: step-flash, gpt-4o, gpt-4o-mini, o3-mini, claude-sonnet, claude-haiku, gemini-flash, llama-70b, mistral-large

### 虚拟机操作

1. **修正代理设置** (系统设置 > 网络 > 代理 > 手动)
   - HTTP代理: `192.168.1.11` 端口 `33210`
   - HTTPS代理: `192.168.1.11` 端口 `33210`

2. **验证代理连通性**
   ```bash
   curl -s -x http://192.168.1.11:33210 https://api.ipify.org
   # 返回: 66.92.207.78 ✅
   ```

3. **部署 SSH 公钥** (方便后续远程管理)
   ```
   ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKA2sh1zhvqopcI275msxJ2vugTPPQTXhdAkESD9f6l6
   ```

## 最终测试结果

| 组件 | 状态 | 详情 |
|------|------|------|
| 宿主机 Clash 代理 | ✅ | `0.0.0.0:33210`, 美国洛杉矶节点 |
| OpenRouter API Key | ✅ | 有效, 非免费层, 无过期时间 |
| OpenRouter 模型调用 | ✅ | 4/4 模型测试通过 (gpt-4o-mini, claude-3.5-haiku, gemini-2.0-flash, deepseek-chat) |
| OpenClaw 网关 (VM) | ✅ | `192.168.1.34:18789`, health: ok |
| 虚拟机网络桥接 | ✅ | SSH 可达, 代理连通 |
| 虚拟机代理出口 | ✅ | 66.92.207.78 (美国) |
| 飞书机器人 Key | ✅ | 已更新为新 Key |
| 飞书机器人进程 | ⚠️ | 未运行, 需手动启动 |

## 关键发现与注意事项

1. **两个 Clash 共存**: 宿主机同时运行 `Clash for Windows` (7890) 和 `闪电加速器` (33210)，只有闪电加速器可用
2. **地区限制**: OpenRouter 的 OpenAI/Anthropic/Google 模型在香港/台湾节点不可用，必须使用美国/日本/新加坡节点
3. **重启后注意事项**:
   - 必须先启动闪电加速器，虚拟机才能通过代理访问 OpenRouter
   - 不开 VPN 时虚拟机普通网络正常，但 OpenClaw 调用 OpenRouter 会失败
   - `allow-lan` 设置可能需要重新开启（取决于闪电加速器是否保存配置）
4. **.bashrc 第121行**: 存在引号未闭合错误，需手动修复

## 生成的工具脚本

| 脚本 | 用途 |
|------|------|
| `fix_vm.sh` | 虚拟机一键修复脚本 (代理、bashrc、SSH公钥) |
| `full_test.ps1` | 宿主机全面测试脚本 |
| `host_status.ps1` | 宿主机状态检查 |

---

*日志由小跃 (StepFun Desktop) 自动生成*
