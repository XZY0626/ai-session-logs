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
