# 2026-03-14 虚拟机 SSH 通道修复 + 服务开机自启修复

## 问题描述

1. Windows OpenSSH 无法读取私钥 `E:\Application\WorkBuddy\id_vm`，报 "not accessible"
2. `openclaw-gateway.service`（system 级）不断重启失败（已重启 190+ 次）
3. 用户希望 AI 能完全自主操作虚拟机，无需手动输入命令

## 修复过程

### 问题一：SSH 私钥权限修复（方案A）

**原因：** Windows OpenSSH 要求私钥文件只有当前用户可读，否则拒绝使用。  
**修复：** 用 PowerShell 移除继承权限，只保留 `DESKTOP-VKTBOP8\须` 的读写权限。

```powershell
# 脚本：E:\Application\WorkBuddy\fix_key_perm.ps1
$keyPath = 'E:\Application\WorkBuddy\id_vm'
$acl = Get-Acl $keyPath
$acl.SetAccessRuleProtection($true, $false)
$currentUser = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule($currentUser, 'Read,Write', 'Allow')
$acl.SetAccessRule($rule)
Set-Acl -Path $keyPath -AclObject $acl
```

**验证：**
```
ssh -i E:\Application\WorkBuddy\id_vm -o StrictHostKeyChecking=no -o BatchMode=yes xzy0626@192.168.1.100 whoami
# 输出：xzy0626 ✅
```

### 问题二：openclaw-gateway 服务持续崩溃

**原因：** openclaw 内部启动时会调用 `systemctl is-enabled`，在 system 级 systemd 服务环境里没有 D-Bus session，报 "Failed to connect to bus: 找不到介质" 然后退出码=1。

**修复：** 改用**用户级 systemd 服务**（`systemctl --user`），用户级服务有完整的 D-Bus session。同时启用 `loginctl linger`，确保即使用户未登录也能自动启动。

```ini
# /home/xzy0626/.config/systemd/user/openclaw-gateway.service
[Unit]
Description=OpenClaw Gateway (User)
After=default.target

[Service]
Type=simple
WorkingDirectory=/home/xzy0626
Environment="HOME=/home/xzy0626"
ExecStart=/usr/bin/openclaw gateway start
Restart=on-failure
RestartSec=10
StartLimitBurst=3
StandardOutput=append:/home/xzy0626/openclaw.log
StandardError=append:/home/xzy0626/openclaw.log

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable openclaw-gateway
systemctl --user start openclaw-gateway
sudo loginctl enable-linger xzy0626
```

## 最终状态

| 项目 | 状态 |
|------|------|
| SSH 免密连接 | ✅ `ssh -i E:\Application\WorkBuddy\id_vm xzy0626@192.168.1.100` |
| openclaw-gateway（用户级） | ✅ active + enabled |
| tailscale-serve（系统级） | ✅ active + enabled |
| loginctl linger | ✅ Linger=yes（开机无需登录自启）|
| OpenClaw 监听端口 | ✅ 127.0.0.1:18789 |
| Tailscale HTTPS 网页访问 | ✅ 用户确认正常 |

## 关于改 Windows 用户名

**不建议。** 改名后注册表中大量 `C:\Users\须\` 路径不会自动更新，风险高。  
私钥安全性取决于权限设置，与盘符无关，现在 E 盘的私钥已经只有当前用户可读，无需移动。

## SSH 连接命令（供参考）

```bash
# 宿主机 → 虚拟机（Windows cmd/PowerShell）
ssh -i E:\Application\WorkBuddy\id_vm -o StrictHostKeyChecking=no -o BatchMode=yes xzy0626@192.168.1.100 <命令>

# 上传文件到虚拟机
scp -i E:\Application\WorkBuddy\id_vm -o StrictHostKeyChecking=no <本地文件> xzy0626@192.168.1.100:<远程路径>
```
