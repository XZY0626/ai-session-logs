# 日志：OpenClaw Gateway 配置修复

**日期：** 2026-03-13 ~ 2026-03-14  
**类型：** 服务配置修复  
**环境：** 虚拟机（Linux）

---

## 问题一：logging.redact 配置项导致启动失败

### 现象
openclaw-gateway 启动后报错，进程无法正常初始化。

### 原因
`~/.openclaw/openclaw.json` 中存在 `logging.redact` 配置项，该字段在当前版本的 openclaw 中已不支持（或类型不匹配），导致配置解析失败。

### 修复方式
通过 SSH 远程执行 Python3 脚本，直接解析并修改 JSON 配置文件：

```python
import json, os
p = os.path.expanduser("~/.openclaw/openclaw.json")
cfg = json.load(open(p))
if "logging" in cfg:
    cfg["logging"].pop("redact", None)          # 移除 redact 字段
    if not cfg["logging"]:
        del cfg["logging"]                        # logging 为空则整块删除
json.dump(cfg, open(p, "w"), indent=2)
```

### 验证
修复后 `logging` 字段不再存在于配置文件中，openclaw 可正常启动。

---

## 问题二：system 级 systemd 服务持续崩溃（190+ 次重启）

### 现象
`openclaw-gateway.service`（system 级）持续以退出码 1 失败，服务状态为 `activating (auto-restart)`，日志中不断出现：

```
Gateway service check failed: Error: systemctl is-enabled unavailable: Failed to connect to bus: 找不到介质
```

### 原因分析
openclaw 程序在启动时会调用 `systemctl is-enabled` 检查自身服务状态。在 **system 级 systemd 服务**环境中运行时，进程没有用户的 D-Bus session（`DBUS_SESSION_BUS_ADDRESS` 无效），导致 systemctl 命令报 "找不到介质" 并返回错误，openclaw 随即以退出码 1 退出。

这是 openclaw 程序设计上未考虑在无 D-Bus 环境下运行的问题，与 service 文件配置无关。

### 无效的尝试
- 设置 `Environment="DBUS_SESSION_BUS_ADDRESS=/dev/null"` → 无效，openclaw 内部检测逻辑仍报错
- 设置 `Environment="OPENCLAW_SKIP_SYSTEMD_CHECK=1"` → openclaw 不支持此变量
- 使用 `--no-daemon` 参数 → openclaw 不支持此参数

### 最终修复：改用用户级 systemd 服务

用户级服务（`systemctl --user`）在用户 session 下运行，天然拥有完整的 D-Bus 环境，openclaw 的内部检测可以正常通过。

**服务文件位置：** `/home/[用户名]/.config/systemd/user/openclaw-gateway.service`

```ini
[Unit]
Description=OpenClaw Gateway (User)
After=default.target

[Service]
Type=simple
WorkingDirectory=/home/[用户名]
Environment="HOME=/home/[用户名]"
ExecStart=/usr/bin/openclaw gateway start
Restart=on-failure
RestartSec=10
StartLimitBurst=3
StandardOutput=append:/home/[用户名]/openclaw.log
StandardError=append:/home/[用户名]/openclaw.log

[Install]
WantedBy=default.target
```

**关键补充配置：启用 linger**

```bash
sudo loginctl enable-linger [用户名]
```

`linger` 确保即使用户未登录，systemd 也会在开机后为该用户创建 session 并启动用户级服务，实现真正的开机自启。

### 验证结果

```
systemctl --user is-active openclaw-gateway   → active
systemctl --user is-enabled openclaw-gateway  → enabled
loginctl show-user [用户名] | grep Linger     → Linger=yes
ss -tlnp | grep 18789                         → openclaw 监听 127.0.0.1:18789
```

---

## 经验总结

1. openclaw-gateway 必须在用户级 systemd 服务下运行（需要 D-Bus session）
2. system 级服务无 D-Bus，openclaw 内部的 systemctl 调用会失败导致进程退出
3. `loginctl enable-linger` 是用户级服务实现"开机自启"的关键，缺少它则用户未登录时服务不会启动
4. openclaw.json 中出现不支持的字段（如旧版 `logging.redact`）会导致启动失败，需用 Python 直接修改 JSON 而非文本编辑（避免格式损坏）
