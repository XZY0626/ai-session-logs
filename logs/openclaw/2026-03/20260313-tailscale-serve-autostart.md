# 日志：Tailscale Serve + 开机自启配置

**日期：** 2026-03-13 ~ 2026-03-14  
**类型：** 网络服务配置  
**环境：** 虚拟机（Linux）

---

## 背景

需要通过 Tailscale Serve 将虚拟机本地的 OpenClaw Gateway（监听 127.0.0.1:18789）暴露为 Tailscale 网络内的 HTTPS 服务，使宿主机可通过 Tailscale HTTPS 地址访问网页控制台。

---

## Tailscale Serve 配置

**启动命令：**
```bash
sudo tailscale serve --bg 18789
```

此命令将本机 18789 端口通过 Tailscale Serve 以 HTTPS 暴露。  
`--bg` 参数使其在后台持久运行（不依赖当前 shell session）。

**访问地址：** `https://[tailscale-hostname].[tailnet].ts.net`

---

## systemd 服务配置（system 级）

与 openclaw-gateway 不同，tailscale-serve 使用 **system 级 systemd 服务**，因为 `tailscale` 命令本身不依赖用户 D-Bus session。

**服务文件：** `/etc/systemd/system/tailscale-serve.service`

```ini
[Unit]
Description=Tailscale Serve
After=network.target tailscaled.service
Requires=tailscaled.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/tailscale serve --bg 18789
ExecStop=/usr/bin/tailscale serve --bg off

[Install]
WantedBy=multi-user.target
```

**启用：**
```bash
sudo systemctl enable tailscale-serve
sudo systemctl start tailscale-serve
```

---

## sudo 免密配置

为支持 AI 自主执行需要 root 权限的命令，配置了 sudo 免密：

**配置位置：** `/etc/sudoers.d/[用户名]` 或通过 `visudo` 写入

```
[用户名] ALL=(ALL) NOPASSWD:ALL
```

> **安全说明：** 此配置只在虚拟机内生效，且虚拟机仅通过 SSH 密钥认证访问（无密码登录入口），综合风险可接受。

---

## 开关机验证

| 项目 | 状态 | 说明 |
|------|------|------|
| tailscale-serve | active + enabled | system 级服务，开机自启 |
| openclaw-gateway | active + enabled | user 级服务，linger 开机自启 |
| Tailscale 网络连接 | 正常 | 两端均在线 |
| 宿主机网页访问 | ✅ 用户确认正常 | HTTPS 访问 OpenClaw 控制台 |

---

## 经验总结

1. `tailscale serve --bg` 的 `--bg` 参数很关键，缺少它则 serve 配置在进程退出后丢失
2. tailscale-serve 适合用 system 级服务（Type=oneshot + RemainAfterExit），因为实际上是配置命令而非持久进程
3. 与 openclaw-gateway 的 user 级服务搭配使用时，启动顺序依赖 tailscaled.service，需在 After/Requires 中声明
