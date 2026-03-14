# 日志：OpenClaw 升级 2026.3.11 → 2026.3.13

**日期：** 2026-03-14  
**类型：** 版本升级 / 服务修复  
**环境：** 虚拟机（Linux）

---

## 版本信息

| 项目 | 版本 |
|------|------|
| 升级前 | 2026.3.11 (29dc654) |
| 升级后 | 2026.3.13 (61d171a) |
| Node.js | v22.22.0 |

---

## 2026.3.12 / 2026.3.13 主要变更（影响本环境的部分）

- **3.12**：Dashboard v2、快速模式、安全收紧（设备配对短生命周期）、Cron 投递路径 breaking change（需 `doctor --fix`）
- **3.13**：安全加固（设备配对码一次性、执行批准强化）、配置验证更严格
- **两版本均无配置文件字段变更**

---

## 升级过程与遭遇的问题

### 问题一：npm 全局安装权限不足（EACCES）

- **原因**：openclaw 安装在 `/usr/lib/node_modules`（需要 root），`sudo npm` 触发 root 的 GitHub SSH key 认证失败（子依赖 `libsignal-node` 需要从 GitHub SSH 拉取）
- **修复**：将 npm 全局前缀切换到用户目录，以普通用户身份安装

```bash
npm config set prefix /home/[用户名]/.local
npm install -g openclaw@latest
# 安装到 /home/[用户名]/.local/lib/node_modules/openclaw
```

### 问题二：新版本启动时检测到危险配置标志并拒绝启动

- **现象**：服务状态 `failed`，日志：`security warning: dangerous config flags enabled: gateway.controlUi.allowInsecureAuth=true, gateway.controlUi.dangerouslyDisableDeviceAuth=true`
- **原因**：3.12 版本安全收紧，`dangerouslyDisableDeviceAuth=true` 会阻止 gateway 启动
- **修复**：通过 Python3 从 `openclaw.json` 中移除这两个字段（依赖默认安全配置）

```python
ctrl.pop('allowInsecureAuth', None)
ctrl.pop('dangerouslyDisableDeviceAuth', None)
```

### 问题三：service 文件 ExecStart 路径未更新（新版装在用户目录）

- **原因**：旧 service 文件仍指向 `/usr/bin/openclaw`，而新版安装在 `/home/[用户名]/.local/bin/openclaw`
- **修复**：更新 ExecStart 路径，并添加 `DBUS_SESSION_BUS_ADDRESS` 环境变量

### 问题四：插件循环注册 + 端口不开（新版 self-restart 机制）

- **现象**：服务状态 `active`，但插件反复注册，端口 18789 始终不监听
- **原因**：新版 openclaw 在启动完成后会调用 `systemctl --user restart` 进行热更新确认，手写的 service 文件缺少 `OPENCLAW_SYSTEMD_UNIT` 等关键环境变量，导致 openclaw 找不到自己的服务名，陷入重启循环
- **正确修复**：使用 openclaw 官方命令重新生成 service 文件

```bash
openclaw gateway install --force
openclaw gateway start
```

官方生成的 service 文件包含所有必要环境变量：
```
OPENCLAW_GATEWAY_PORT=18789
OPENCLAW_SYSTEMD_UNIT=openclaw-gateway.service
OPENCLAW_SERVICE_MARKER=openclaw
OPENCLAW_SERVICE_KIND=gateway
OPENCLAW_SERVICE_VERSION=2026.3.13
```

---

## 最终状态验证

| 项目 | 状态 |
|------|------|
| openclaw 版本 | 2026.3.13 ✅ |
| openclaw-gateway 服务 | active + enabled ✅ |
| tailscale-serve 服务 | active + enabled ✅ |
| 端口 18789 监听 | 127.0.0.1:18789 ✅ |
| Tailscale 网络 | 两端在线 ✅ |
| loginctl linger | Linger=yes ✅ |
| 危险配置标志 | 已清除 ✅ |

---

## 经验总结

1. **升级 openclaw 时，永远先用 `openclaw update` 或 `npm install -g openclaw@latest`，不要用 `sudo npm`**（会触发 root SSH key 问题）；若全局目录无权限，改用 `npm config set prefix ~/.local`
2. **升级后必须用 `openclaw gateway install --force` 重新生成 service 文件**，手写的 service 文件缺少关键环境变量会导致新版自重启机制失效
3. **每次升级前看 CHANGELOG**，安全相关的配置收紧（如 `dangerouslyDisableDeviceAuth`）会静默阻止启动，不一定有明显报错
4. **`openclaw doctor --fix` 是升级后的标准操作**，可迁移 Cron 等数据格式
5. openclaw 新版日志从 `~/openclaw.log` 迁移到 `/tmp/openclaw/openclaw-YYYY-MM-DD.log`，排查问题时注意查新路径

---

## 附录：升级后首次登录问题排查（2026-03-14）

### 背景

升级到 3.13 后，虚拟机本地浏览器可以正常登录 Dashboard，但宿主机通过 Tailscale 访问时一直显示"需要配对"，无法进入。

### Dashboard v2 登录方式变更

| 项目 | 旧版（3.11） | 新版（3.12/3.13） |
|------|------------|-----------------|
| 进入方式 | 打开带 token 的 URL 直接进入 | 需在界面手动填入 WebSocket URL 和网关令牌 |
| 令牌获取 | URL 内嵌 | `openclaw dashboard --no-open` 输出带令牌的完整 URL，从中提取令牌 |
| 本地/远程认证 | 统一 | 区分 local（需配对）和 remote 客户端 |

### Dashboard 密码字段说明

登录界面有三个输入项：WebSocket URL、网关令牌、密码（可选）。

**密码字段不是账户密码**，它对应的是 `gateway.auth.mode` 的两种认证模式：

| 认证模式 | 密码字段 | 适用场景 |
|---------|---------|---------|
| `token`（当前配置） | 留空，完全忽略 | Tailscale serve（私有网络），推荐默认 |
| `password` | 必填，连接时验证 | Tailscale Funnel（公网暴露），token 不适合公网明文传输 |

openclaw 源码中有明确提示：`"Tailscale funnel requires password auth"`，说明 `password` 模式专为 Funnel 公网场景设计。

**当前配置为 `token` 模式，密码字段留空即可。**

### 问题一：`allowedOrigins` 配置误将本地也屏蔽

- **排查过程**：查看 gateway 日志发现 `code=4008 connect failed`，继续查详细日志发现 `cause: origin-mismatch`
- **根因**：在修复宿主机访问时新增了 `allowedOrigins` 只允许 Tailscale 域名，导致虚拟机本地 `http://127.0.0.1:18789` 的 origin 也被拒绝
- **修复**：将 `http://127.0.0.1:18789` 和 `http://localhost:18789` 一同加入 `allowedOrigins`

### 问题二：宿主机显示"需要配对"，无配对按钮

- **根因**：新版 `local` 模式下，gateway 将所有非 loopback 来源的客户端视为"远程设备"，必须完成设备配对才能使用。通过 Tailscale 代理转发的宿主机请求属于此类
- **尝试过但无效的方案**：
  - `gateway.mode=network`：该值不存在（仅 `local`/`remote`）
  - `gateway.mode=remote`：需额外 remote server 配置，不适用本地自托管
  - `gateway.auth.mode=trusted-proxy`：需要代理在请求头中注入用户信息，Tailscale serve 不具备此能力
- **正确方案：走设备配对流程**
  1. 宿主机浏览器打开 Dashboard，填入令牌后点击连接（触发配对请求）
  2. 在虚拟机执行 `openclaw devices list` 查看待处理的配对请求
  3. 执行 `openclaw devices approve <requestId>` 批准
  4. 配对仅需做一次，此后宿主机作为已信任设备直接用令牌登录

```bash
# 查看配对请求
openclaw devices list

# 批准（request ID 从 list 输出中获取）
openclaw devices approve <request-id>
```

### 关于 `gateway.trustedProxies` 配置

在排查过程中加入了 `gateway.trustedProxies: ["127.0.0.1"]`。该配置作用是让 gateway 信任反向代理（Tailscale serve）传来的 `X-Forwarded-For` 头，正确识别真实客户端 IP。这与配对要求是两个独立问题：
- `trustedProxies` 解决的是 IP 识别问题（日志中的 `Proxy headers detected from untrusted address` 警告）
- 配对要求来自 `gateway.mode=local` 的安全策略，和 IP 识别无关

两个配置均需要正确设置，缺一不可。

### 最终登录方式备忘

1. 访问 `https://[tailscale-hostname]`（Tailscale HTTPS 地址）
2. WebSocket URL 字段自动填入，无需修改
3. 网关令牌：在虚拟机执行 `openclaw dashboard --no-open`，从输出 URL 的 `#token=` 后面提取（服务重启后令牌会变）
4. 密码字段：留空（当前为 token 认证模式）
5. 点击连接，首次需要在虚拟机端批准配对请求

---

## 附录：开机自启验证与方案 A 安全性分析（2026-03-14）

### 开机自启链路验证

关机重启后 openclaw 能否自动恢复，取决于以下三个服务/配置全部就绪：

| 组件 | 类型 | 状态 | 说明 |
|------|------|------|------|
| `tailscaled` | system 级 systemd | enabled ✅ | Tailscale 守护进程，开机自启 |
| `tailscale-serve` | system 级 systemd | enabled ✅ | HTTPS 反向代理，开机自启 |
| `openclaw-gateway` | 用户级 systemd | enabled ✅ | openclaw 网关，用户级自启 |
| loginctl linger | xzy0626 用户 | Linger=yes ✅ | 无需登录即可启动用户级服务 |

端口监听：`127.0.0.1:18789` 已由 openclaw-gateway 持有 ✅

**结论：全链路开机自启配置正确，无需手动干预。**

> 注意：`openclaw-gateway` 是用户级（`--user`）systemd 服务，`loginctl linger=yes` 是让它在无人登录时也能随系统启动。如果 linger 被意外关闭（`loginctl disable-linger xzy0626`），openclaw 只会在你 SSH 登录后才启动。

---

### 方案 A 安全性分析（`gateway.mode=remote`）

在排查宿主机配对问题时，曾考虑过将 `gateway.mode` 改为 `remote` 来跳过配对要求，但最终放弃了。这里记录为什么放弃以及该模式的安全含义。

**方案 A 是什么：** 将 `gateway.mode` 从 `local` 改为 `remote`，openclaw gateway 设计为"被远程服务端管理"的模式，不强制要求设备配对。

**安全性评估：**

| 维度 | local 模式（当前） | remote 模式 |
|------|-----------------|------------|
| 攻击面 | 只监听 loopback，网络层隔离 | 同样只监听 loopback，但认证逻辑不同 |
| 认证强度 | token + 设备配对双重验证 | 依赖 remote server 的身份管理，本地配对可绕过 |
| 适用场景 | 个人/团队自托管 | 企业 SaaS 托管（openclaw cloud） |
| 配对要求 | 有，首次需要 | 无 |
| 是否适合当前环境 | ✅ 是 | ❌ 否（需要配套 remote server，单机部署无法启动） |

**结论：方案 A 不适合当前自托管场景**，并非安全性低，而是设计上就不支持无 remote server 的独立部署（实测 `gateway.mode=remote` 时服务直接拒绝启动）。

**当前方案（local + 设备配对）安全性分析：**
- gateway 只绑定 loopback（`127.0.0.1`），外部无法直接访问
- Tailscale serve 做 HTTPS 终止，流量在 Tailscale 网络内加密
- 宿主机首次访问需配对批准，之后作为受信任设备
- 令牌不存储在浏览器中（"不存储"为 Dashboard 界面原文）

这套方案在私有 Tailscale 网络下安全性是足够的。
