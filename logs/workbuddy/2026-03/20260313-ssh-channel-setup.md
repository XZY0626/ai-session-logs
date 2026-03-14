# 日志：SSH 免密登录建立过程

**日期：** 2026-03-13 ~ 2026-03-14  
**类型：** 安全通道配置  
**环境：** 宿主机（Windows）→ 虚拟机（Linux 192.168.1.100）

---

## 背景

用户希望 AI 能完全自主操作虚拟机，不需要用户手动输入命令处理网络问题。

---

## 经历的故障链

### 阶段一：公钥手工写入出错

- **尝试：** 手动将公钥内容粘贴写入虚拟机 `~/.ssh/authorized_keys`
- **现象：** SSH 登录时报 `is not a public key file`
- **原因：** 手工输入公钥内容时格式错误（多空格、换行、字符错误）

### 阶段二：原始私钥文件损坏

- **尝试：** 使用原有私钥文件 `vm_key` 进行认证
- **现象：** 私钥无法完成签名操作，认证始终失败
- **原因：** 原始私钥文件内容已损坏，无法修复

### 阶段三：路径中文问题

- **现象：** 私钥存放在 `C:\Users\须\...` 路径下，部分工具读取失败
- **原因：** 宿主机 Windows 用户名含中文"须"，部分命令行工具（Git Bash 路径解析等）无法正确处理含中文的路径
- **修复：** 所有文件改用纯英文路径 `E:\Application\WorkBuddy\`

### 阶段四：SSH_ASKPASS 方案（方案B原型）

- **背景：** 由于公钥认证失败，需要通过密码登录来覆写 `authorized_keys`
- **方案：** 利用 Git Bash 的 `SSH_ASKPASS` 机制实现密码自动输入
  - 写 `askpass.sh` 脚本，内容只有一行：输出密码
  - 设置环境变量 `SSH_ASKPASS` 指向该脚本，`SSH_ASKPASS_REQUIRE=force`
  - Git Bash 的 ssh 会调用该脚本获取密码，**无需用户手动输入**
- **注意：** 此方案依赖用户事先提供密码，密码明文存储在脚本中，属于临时过渡方案，不作为长期方案使用

> **关于方案B的说明：** 方案B（Git Bash SSH）本质上确实依赖用户提供的虚拟机密码，通过 SSH_ASKPASS 自动填入。这是一个临时引导手段，目的是在密钥认证失败时完成 `authorized_keys` 的写入，从而建立正式的密钥认证通道。方案B本身不是最终的运行方式。

### 阶段五：重新生成密钥对（最终方案）

- **操作：** 在 `E:\Application\WorkBuddy\` 下重新生成 ed25519 密钥对
  - 私钥：`id_vm`
  - 公钥：`id_vm.pub`
- **写入：** 通过 SSH_ASKPASS + 密码登录，将公钥覆写入虚拟机 `~/.ssh/authorized_keys`
- **结果：** 密钥认证成功建立

### 阶段六：Windows OpenSSH 权限问题（本次）

- **现象：** `ssh -i E:\Application\WorkBuddy\id_vm` 报 "Identity file not accessible"
- **原因：** Windows OpenSSH 对私钥文件有严格权限要求，私钥不能被其他用户读取
- **修复：** 用 PowerShell 移除文件继承权限，只保留当前用户的读写权限（`icacls` 等价操作）

```powershell
$acl.SetAccessRuleProtection($true, $false)  # 断开继承
# 只保留当前用户 Read,Write
```

---

## 最终可用连接命令

```
# 执行远程命令
ssh -i E:\Application\WorkBuddy\id_vm -o StrictHostKeyChecking=no -o BatchMode=yes xzy0626@[VM_IP] <命令>

# 上传文件（多行命令通过上传脚本到 /tmp/ 执行）
scp -i E:\Application\WorkBuddy\id_vm -o StrictHostKeyChecking=no <本地文件> xzy0626@[VM_IP]:<远程路径>
```

---

---

## 备选方案B：Git Bash SSH（方案A失效时使用）

### 适用场景

当 **Windows 系统 OpenSSH 私钥权限修复失效**（如系统更新后权限被重置、PowerShell 无法执行等）时，可使用此方案作为备用连接手段。

### 方案B原理

Git Bash 自带的 `ssh` 工具（位于 `E:\Application\Git\Git\bin\ssh.exe` 或类似路径）**不强制检查私钥文件的 Windows ACL 权限**，因此即使私钥权限不满足 Windows OpenSSH 的要求，Git Bash ssh 仍然可以读取并使用。

但 Git Bash ssh 有一个限制：**在非交互式环境（如 AI 自动化执行）下，无法弹出密码输入框**。这时需要借助 `SSH_ASKPASS` 机制。

### SSH_ASKPASS 机制说明

`SSH_ASKPASS` 是 Unix/Linux ssh 客户端的标准机制：当 ssh 需要输入密码且没有终端时，它会调用 `SSH_ASKPASS` 环境变量指向的程序来获取密码，该程序只需将密码输出到 stdout 即可。

Git Bash 的 ssh 支持此机制。

### 方案B实施步骤

**前提：** 需要知道虚拟机的密码（用于初次引导或密钥失效后的恢复）。

**步骤一：创建 askpass 脚本**

```bash
# E:\Application\WorkBuddy\askpass_temp.sh（执行完毕后立即删除）
#!/bin/bash
echo "虚拟机登录密码"
```

**步骤二：设置环境变量并连接**

```bash
# 在 Git Bash 中执行
export SSH_ASKPASS="/e/Application/WorkBuddy/askpass_temp.sh"
export SSH_ASKPASS_REQUIRE=force
export DISPLAY=":0"

# 连接并执行命令
ssh -o StrictHostKeyChecking=no xzy0626@192.168.1.100 "执行的命令"
```

**步骤三：完成后立即删除含密码的脚本**

```bash
rm -f /e/Application/WorkBuddy/askpass_temp.sh
```

### 在 Windows cmd / PowerShell 中调用 Git Bash ssh

由于 Windows shell 的引号传递问题，**多行命令需要先写成脚本文件，再通过 Git Bash 执行**：

```batch
:: 写脚本到临时文件
:: 用 Git Bash 执行脚本
"E:\Application\Git\Git\bin\bash.exe" E:\Application\WorkBuddy\your_script.sh
```

> **注意：** 直接在 `bash.exe -c "..."` 中嵌套复杂命令会有引号解析问题，始终优先使用脚本文件方式。

### 方案B的局限性

| 项目 | 说明 |
|------|------|
| 密码依赖 | 需要提前知道虚拟机密码，密码变更后需更新 |
| 安全风险 | 密码明文存储于脚本文件，使用完毕必须立即删除 |
| 不适合长期 | 仅作为密钥认证失效时的应急恢复手段 |
| Git Bash 路径 | 依赖 Git for Windows 已安装，路径可能因安装位置不同而变化 |

### 方案B使用后的恢复流程

方案B成功连接后，应立即执行：

1. 重新生成 SSH 密钥对（或修复现有私钥权限）
2. 将新公钥写入虚拟机 `~/.ssh/authorized_keys`
3. 验证密钥认证可用
4. 删除含密码的 askpass 脚本

---

## 经验总结

1. 公钥内容不能手工输入，必须通过脚本/管道写入，避免格式错误
2. Windows 用户名含中文会导致部分 CLI 工具路径解析异常，密钥文件应存放在纯英文路径
3. Windows OpenSSH 私钥权限要求严格，需主动移除继承权限（首选方案A）
4. 方案A失效时可用方案B（Git Bash SSH_ASKPASS），但需要虚拟机密码且用后立即清理
5. 多行命令通过 scp 上传脚本到 /tmp/ 再执行的方式，可避免 Windows shell 引号传递问题
6. 建立密钥认证的引导过程中，密码只作为临时手段，完成后务必删除明文密码脚本
