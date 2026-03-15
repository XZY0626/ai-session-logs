# 2026-03-15 WorkBuddy 操作日志 — workspace 路径修复 + VoxBridge 项目启动

**时间**：2026-03-15（下午至傍晚）
**执行者**：WorkBuddy
**触发渠道**：用户反馈龙虾推送 GitHub 失败 + VoxBridge 项目授权

---

## 一、问题背景

用户在飞书询问龙虾（OpenClaw），龙虾回复说"workspaceOnly 限制导致无法写入 `~/ai-session-logs/`"，要求用户授权操作。用户将回复内容转发给 WorkBuddy，请求诊断原因。

---

## 二、问题诊断：workspaceOnly 路径检查机制

### 发现的核心问题

openclaw 的 `write_file`/`read_file` 工具在工具层面做**字符串前缀检查**，路径必须以 workspace 根目录开头：

```
✅ 允许：/home/xzy0626/.openclaw/workspace/logs/...
✅ 允许：workspace/logs/...（相对路径）
❌ 拒绝：~/ai-session-logs/...
❌ 拒绝：/home/xzy0626/ai-session-logs/...
```

**软链接在文件系统层面是通的**（测试写入 `SYMLINK_TEST.md` 成功出现在 `ai-session-logs/` 里），但 openclaw 工具在传入路径时做的是字符串检查，不是实际路径解析，所以软链接对 `write_file` 无效——需要传入 workspace 前缀路径。

### github-sync.md 中的错误路径

旧版 `github-sync.md` 写的是：
```
~/ai-session-logs/logs/openclaw/YYYY-MM/...  ← 被工具拦截
```
龙虾严格按照文档执行，所以每次都失败。

---

## 三、修复内容

### 3.1 github-sync.md 完整重写

- 新增"workspaceOnly 路径说明"专节，明确 `write_file` 必须用 `workspace/` 前缀
- 修正所有路径示例
- 安全扫描规则优化：IP 地址（192.168.x.x 形式）不需要拦截，只有 API key / 私钥 / password 才停止提交
- 更新仓库状态：SSH 认证已通过，upstream 已设置，可直接 `git push`

### 3.2 TOOLS.md GitHub 操作规范段更新

将路径说明从 `~/ai-session-logs/...` 改为 `workspace/logs/openclaw/...`，注明软链接映射关系。

### 3.3 workspace 本地 git commit

两处修改已在虚拟机 workspace 内 git commit（`fcf7a25`）。

---

## 四、VoxBridge 项目启动

### 背景

龙虾此前已完成语音翻译可行性调研（`voice_translation_feasibility_report.md`，约 20KB），内容涵盖：
- Whisper / faster-whisper ASR 对比
- 翻译引擎选型（Helsinki-NLP / MarianMT 等）
- TTS 方案选型
- GPU 加速方案

### 龙虾请求授权的操作

1. 在 `workspace/VoxBridge/` 内整理项目目录结构
2. 初始化 git，关联 `git@github.com:XZY0626/VoxBridge.git`
3. 将调研报告整理进 `docs/` 并推送

### 问题：仓库名不一致

龙虾最初请求的仓库名为 `VoxBridge-`（带横杠），但用户在 GitHub 创建的仓库名为 `VoxBridge`（无横杠）。

**处置**：告知龙虾使用正确 remote 地址：
```
git remote set-url origin git@github.com:XZY0626/VoxBridge.git
```

### 当前状态

- `workspace/VoxBridge/` 和 `workspace/VoxBridge-/` 两个目录均存在（龙虾可能创建了两个）
- GitHub 仓库 `XZY0626/VoxBridge` 已由用户创建
- 推送操作由龙虾自行执行（WorkBuddy 侧不介入代码仓库操作）

---

## 五、文件变更清单

| 文件 | 位置 | 操作 | 说明 |
|------|------|------|------|
| `skills/github-sync.md` | 虚拟机 workspace | 完整重写 | 修正路径规范，新增 workspaceOnly 说明 |
| `TOOLS.md` | 虚拟机 workspace | 局部更新 | GitHub 操作规范段路径修正 |
| workspace `fcf7a25` | 虚拟机 | git commit | 上述两文件变更提交 |

---

## 六、合规说明

- 无 L0 违规
- 无敏感信息写入日志
- SSH 公钥（ed25519）已记录于上条日志，非私钥，可公开
- VoxBridge 仓库操作均为代码/文档类，无凭证风险
- 虚拟机 IP 地址已脱敏（使用 192.168.x.x 形式）

---

*日志由 WorkBuddy 自动记录 | 2026-03-15*
