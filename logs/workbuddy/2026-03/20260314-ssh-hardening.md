# 虚拟机 SSH 安全加固日志（2026-03-14）

## 背景

完成 openclaw 升级与登录问题排查后，对虚拟机 SSH 认证方式进行加固：关闭密码认证，仅保留密钥认证。

---

## 环境信息

| 项目 | 值 |
|------|-----|
| 虚拟机系统 | Ubuntu 24.04.4 LTS (Noble Numbat) |
| SSH 服务 | `ssh.service`（OpenBSD Secure Shell server） |
| 密钥类型 | ed25519 |
| 私钥存储 | 宿主机 `E:\Application\WorkBuddy\id_vm`（仅当前 Windows 用户可读） |

---

## 加固前状态

通过 `sudo sshd -T` 查询实际运行时配置：

```
passwordauthentication yes   ← 密码登录开启（需关闭）
pubkeyauthentication yes     ← 密钥认证开启
challengeresponseauthentication no
```

`/etc/ssh/sshd_config` 中仅有 `UsePAM yes`，`PasswordAuthentication` 未显式配置，使用 Ubuntu 默认值（yes）。

---

## 加固操作

### 步骤一：备份原始配置

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.20260314
```

### 步骤二：写入加固配置

Ubuntu 24.04 推荐通过 `sshd_config.d/` 覆盖，不直接修改主配置文件，便于管理和回滚：

```bash
# /etc/ssh/sshd_config.d/99-hardening.conf
PasswordAuthentication no
ChallengeResponseAuthentication no
PermitRootLogin no
```

### 步骤三：验证配置语法

```bash
sudo sshd -t
# 输出：（无报错，语法检查通过）
```

### 步骤四：重载 sshd

```bash
sudo systemctl reload ssh
# 使用 reload 而非 restart，避免中断现有连接
```

---

## 加固后验证

### 运行时配置确认

```
permitrootlogin no           ✅
pubkeyauthentication yes     ✅
passwordauthentication no    ✅ （已关闭）
```

### 密钥登录验证

加固后重新建立 SSH 连接，确认密钥认证仍然有效：

```bash
ssh -i [私钥路径] -o BatchMode=yes xzy0626@[VM_IP] "whoami && echo KEY_LOGIN_OK"
# 输出：xzy0626 KEY_LOGIN_OK ✅
```

---

## 加固后安全状态

| 维度 | 状态 | 说明 |
|------|------|------|
| 密码登录 | 🚫 已关闭 | 无法通过密码暴力破解 |
| 密钥登录 | ✅ 正常 | ed25519 密钥，宿主机 AI 自主操作不受影响 |
| root 登录 | 🚫 已禁止 | `PermitRootLogin no` |
| 网络层隔离 | ✅ 内网 | SSH 只在局域网可达，未暴露公网 |
| 私钥安全 | ✅ 受保护 | Windows ACL 只允许当前用户读写 |

---

## 兜底方案

**私钥丢失时的恢复方式：**

VMware 控制台（物理级访问）完全不经过 SSH，任何时候都可以从 VMware 界面直接登录虚拟机控制台重置 authorized_keys，不存在被锁死的风险。

---

## 配置文件位置备忘

| 文件 | 路径 |
|------|------|
| SSH 主配置 | `/etc/ssh/sshd_config` |
| 加固覆盖配置 | `/etc/ssh/sshd_config.d/99-hardening.conf` |
| 加固前备份 | `/etc/ssh/sshd_config.bak.20260314` |
| authorized_keys | `~/.ssh/authorized_keys` |
| 宿主机私钥 | `E:\Application\WorkBuddy\id_vm` |

---

## 附录：重启验证与 tailscale-serve 启动顺序修复（2026-03-14）

### 第一次重启验证结果

加固完成后立即重启虚拟机进行验证，发现 `tailscale-serve` 服务启动失败：

```
tailscale-serve.service: Failed with result 'exit-code'
tailscale[1574]: unexpected state: NoState
```

**根因：** `tailscale serve --bg 18789` 在 systemd 启动时执行太早，此时 tailscaled 守护进程虽已启动但尚未完成与 Tailscale 网络的握手（处于 `NoState`），导致 serve 命令失败退出。

注：tailscale serve 的规则配置本身没有丢失，仅是本次启动时命令执行失败。

### 修复方案

在 `tailscale-serve.service` 中添加 `ExecStartPre`，在执行 serve 之前轮询等待 tailscale 网络就绪（最多等 30 秒）：

```ini
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStartPre=/bin/bash -c 'for i in $(seq 1 30); do tailscale status 2>/dev/null | grep -q "^[0-9]" && exit 0; sleep 1; done; exit 1'
ExecStart=/usr/bin/tailscale serve --bg 18789
ExecStop=/usr/bin/tailscale serve --https=443 off
```

备份原文件：`/etc/systemd/system/tailscale-serve.service.bak.20260314`

### 第二次重启验证结果（全链路）

| 检查项 | 状态 |
|--------|------|
| SSH 密钥连接（20秒内） | ✅ |
| `passwordauthentication no` 持久化 | ✅ |
| `pubkeyauthentication yes` | ✅ |
| `permitrootlogin no` | ✅ |
| openclaw-gateway | ✅ active |
| tailscale-serve | ✅ active（修复有效） |
| tailscaled | ✅ active |
| port 18789 监听 | ✅ |
| loginctl linger | ✅ Linger=yes |

**结论：所有服务开机自启正常，SSH 安全加固持久化生效，全链路验证通过。**
