# WorkBuddy 操作日志 2026-03-13（三）

## 操作：OpenClaw 网络配置 - Tailscale HTTPS 接入

### 环境信息
- 平台：Linux 虚拟机（VMware）
- 宿主机系统：Windows
- 目标：为 OpenClaw Control UI 提供可信 HTTPS 访问，解决浏览器设备身份验证限制

---

## 背景与问题

OpenClaw v2026.3.11 引入设备身份验证机制，浏览器在纯 HTTP 环境下会阻止相关 API 调用，导致登录时持续出现"device identity required"错误。

根本原因：浏览器安全策略（Secure Context）限制：
- `navigator.credentials` / WebAuthn / 部分安全 API 仅在 HTTPS 或 localhost 下可用
- 通过局域网 IP（如 `http://[VM_LAN_IP]:18789`）访问属于不安全上下文，设备身份验证 API 被拒绝

---

## 解决方案：Tailscale Serve

使用 Tailscale 的 Serve 功能，将虚拟机的 OpenClaw 端口通过 Tailscale 网络以 HTTPS 暴露，无需自签证书，无需公网 IP。

### 方案优势

| 对比 | 局域网 HTTP | Tailscale Serve HTTPS |
|------|------------|----------------------|
| 是否需要公网 IP | 否 | 否 |
| 是否需要域名 | 否 | 自动分配 |
| 是否需要证书 | 否 | Tailscale 自动管理 |
| 浏览器安全上下文 | 不满足 | 满足 |
| 跨网络访问 | 仅局域网 | 任意 Tailscale 设备 |

---

## 配置过程

### 1. 虚拟机安装 Tailscale
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```
认证：终端输出登录链接，在浏览器打开完成账号认证。

### 2. 开启 Tailscale Serve（HTTPS 转发）

在 Tailscale 管理后台开启 Serve 功能后，执行：
```bash
sudo tailscale serve --bg 18789
```

验证：
```
Available within your tailnet:
https://[TAILSCALE_HOSTNAME].[TAILNET_DOMAIN] (Funnel off)
|-- / proxy http://127.0.0.1:18789
```

HTTPS 访问地址：`https://[TAILSCALE_HOSTNAME].[TAILNET_DOMAIN]`

### 3. 修改 OpenClaw Gateway 配置

修改 `~/.openclaw/openclaw.json`：

```json
{
  "gateway": {
    "bind": "loopback",
    "tailscale": {
      "mode": "serve"
    },
    "controlUi": {
      "allowInsecureAuth": true,
      "dangerouslyDisableDeviceAuth": true,
      "allowedOrigins": ["https://[TAILSCALE_HOSTNAME].[TAILNET_DOMAIN]"]
    }
  }
}
```

**关键配置说明：**

| 配置项 | 值 | 说明 |
|--------|----|------|
| `gateway.bind` | `loopback` | Gateway 仅监听本地回环，不暴露到局域网，通过 Tailscale 转发 |
| `gateway.tailscale.mode` | `serve` | 启用 Tailscale Serve 集成 |
| `controlUi.allowedOrigins` | Tailscale HTTPS 域名 | 解决跨域（CORS）访问限制 |
| `controlUi.allowInsecureAuth` | `true` | 允许非标准安全上下文认证 |
| `controlUi.dangerouslyDisableDeviceAuth` | `true` | 禁用设备身份验证（临时方案） |

### 4. 宿主机安装 Tailscale

Windows 宿主机通过官网下载安装：`https://tailscale.com/download`

登录同一 Tailscale 账号后，自动加入同一 tailnet，即可通过 HTTPS 域名访问虚拟机的 OpenClaw。

---

## CORS 问题修复

通过 Tailscale HTTPS 访问时遇到跨域错误：
```
不允许 origin（从网关主机打开控制界面，或在 gateway.controlUi.allowedOrigins 中允许）
```

**原因**：OpenClaw 默认不允许来自非本机的 Origin。

**修复**：在 `controlUi.allowedOrigins` 中添加 Tailscale HTTPS 域名，重启 gateway 进程生效。

---

## 重启 Gateway 进程

OpenClaw gateway 以 nohup 方式后台运行，配置修改后需手动重启：

```bash
# 注意：进程名超过 15 字符，需要用 -f 参数
kill $(pgrep -f openclaw-gateway)
sleep 2
nohup openclaw gateway start > ~/openclaw.log 2>&1 &
```

---

## 最终验证

| 验证项 | 结果 |
|--------|------|
| 宿主机 HTTPS 访问 | 正常，页面加载 |
| token 登录 | 正常，认证通过 |
| 飞书 OpenClaw 插件 | 正常使用 |
| 虚拟机本地访问 | `http://localhost:18789`（无需 Tailscale）|

---

## 后续建议

1. **评估是否恢复设备身份验证**：当前 `dangerouslyDisableDeviceAuth: true` 为临时方案，HTTPS 环境稳定后可尝试改为 `false` 测试是否正常
2. **Tailscale Serve 持久化**：确认 `sudo tailscale serve --bg 18789` 在系统重启后是否自动恢复（若否，需加入开机启动脚本）

---

## 脱敏说明
- 虚拟机 LAN IP：已脱敏为 `[VM_LAN_IP]`
- Tailscale 域名：已脱敏为 `[TAILSCALE_HOSTNAME].[TAILNET_DOMAIN]`
- Token、API Key、账号密码：未记录

---

*操作时间：2026-03-13*
*记录人：WorkBuddy*
