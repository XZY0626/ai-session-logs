# AI 会话日志 - 2026-03-13 - OpenClaw 前端 MiniMax 模型接入

## 会话概要
- 日期：2026-03-13
- 主题：OpenClaw 前端模型选择器升级，接入 MiniMax 模型组

## 问题背景
OpenClaw 控制台前端的可视化模型选择列表（model-selector）中缺少 MiniMax 分组。
后端 openclaw.json 中已配置 4 个 MiniMax 模型，但前端列表未反映。

## 操作过程

### 1. 诊断
- 确认前端文件位于：`/usr/lib/node_modules/openclaw/dist/control-ui/assets/model-selector-v2.js`
- 当前版本：v4（无 MiniMax 分组）
- 文件由 npm 全局包提供，为只读状态

### 2. 制作 v5
在宿主机 `E:/agent/openclaw-config/` 中，基于 v4 新增 MiniMax 分组：
- `minimax/MiniMax-M1` — 思考型，alias: minimax-m1
- `minimax/MiniMax-M2` — 推理型，alias: minimax-m2
- `minimax/MiniMax-M2.5` — 旗舰版，alias: minimax
- `minimax/MiniMax-Text-01` — 长文本，alias: minimax-text

### 3. 部署
- 通过 SFTP 上传 v5 到虚拟机 `~/.openclaw/model-selector-v2.js`（持久备份）
- 通过 `sudo cp` 覆盖 npm 包内文件
- 创建恢复脚本 `~/.openclaw/apply-custom-ui.sh`，版本更新后一键恢复

### 4. 验证
- npm 包文件内 MiniMax 行数：6（正常）
- HTTP 服务 `/assets/model-selector-v2.js` MiniMax 行数：6（正常）
- 版本标识：v5

## 防丢设计
| 位置 | 说明 |
|------|------|
| `~/.openclaw/model-selector-v2.js` | 虚拟机持久备份，不受 npm 更新影响 |
| `~/.openclaw/apply-custom-ui.sh` | 一键恢复脚本 |
| GitHub openclaw-config 仓库 | 云端备份 |

版本更新后恢复命令：
```bash
~/.openclaw/apply-custom-ui.sh
```

## 结果
- 前端模型选择列表新增 MiniMax 分组，刷新页面即可见
- 代码已提交 GitHub：`feat: model-selector v5 - add MiniMax group (4 models)`

## 脱敏说明
- 所有 API Key、IP 地址、用户名已脱敏处理
- 虚拟机 IP：已脱敏（内网地址）
- MiniMax API Key：存于虚拟机环境变量，未入库
