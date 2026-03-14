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
