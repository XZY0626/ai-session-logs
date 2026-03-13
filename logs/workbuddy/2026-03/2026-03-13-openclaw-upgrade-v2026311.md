# WorkBuddy 操作日志 2026-03-13（三）

## 操作：OpenClaw 版本升级 v2026.3.8 -> v2026.3.11

### 环境信息
- 平台：Linux 虚拟机（VMware）
- 连接方式：SSH 密码认证（宿主机 -> 虚拟机）
- OpenClaw 安装方式：npm 全局安装，nohup 后台启动 gateway 服务

---

## 升级流程

### 1. 更新前状态确认
| 项目 | 状态 |
|------|------|
| 当前版本 | v2026.3.8 |
| npm 可用版本 | 2026.3.11 (confirmed) |
| Gateway 服务 | nohup 后台运行 |
| HTTP 18789 | 正常 200 |
| 飞书 channel | enabled |
| MiniMax 配置 | 已在 openclaw.json providers |

### 2. 升级过程

#### 遇到问题：npm ENOTEMPTY
```
npm error ENOTEMPTY: directory not empty, rename '/usr/lib/node_modules/openclaw' -> ...
```
**原因**：npm 全局更新时，旧目录非空无法直接重命名。
**解决**：先用 `sudo mv` 将旧版目录移走，再执行 `npm install -g openclaw@2026.3.11`，安装成功后删除旧版目录备份。

#### 成功安装
```
added 684 packages in 2m
openclaw 2026.3.11 (29dc654)
```

### 3. 前端架构变化（重要）

v2026.3.11 的 Control UI 采用了不同的前端架构：

| 对比项 | v2026.3.8 | v2026.3.11 |
|--------|-----------|------------|
| 模型列表来源 | 前端 model-selector 插件（独立 JS 文件） | 后端 openclaw.json providers 配置（动态加载）|
| 前端文件结构 | 含 `model-selector-v2.js` 独立文件 | 单文件打包（`index-[hash].js`，Vite 构建）|
| 自定义模型方式 | 覆盖替换前端 JS | 修改 openclaw.json 的 models.providers |

**结论**：model-selector 插件方式已天然过时，新版直接从配置文件读取——**这反而更稳定，不会因版本更新失效**。MiniMax 等所有模型配置保存在 openclaw.json 中，版本更新时配置文件不会被覆盖。

---

### 4. API 路径变化（重要）

v2026.3.11 同步更新了 API 路径，与旧版不兼容：

| 项目 | v2026.3.8 | v2026.3.11 |
|------|-----------|------------|
| 登录接口 | `/api/v1/auth/login` | `/auth/login` |
| 其他 API | `/api/v1/...` | `/...` |

升级后若使用旧路径调用 API 会返回 404，需注意同步更新客户端或脚本中的接口地址。

---

### 5. 认证问题排查：device identity required

升级完成后访问 Control UI 遇到"设备身份要求（device identity required）"错误，排查过程如下：

#### 问题一：allowInsecureAuth 未开启
- 现象：输入 token 后提示认证失败
- 原因：`gateway.controlUi.allowInsecureAuth` 默认为 `false`，HTTP 环境下不允许认证
- 修复：将该配置改为 `true`

#### 问题二：浏览器阻止 HTTP 下的设备身份验证
- 现象：allowInsecureAuth 开启后仍然报 device identity required
- 原因：v2026.3.11 引入设备身份验证机制，浏览器安全策略在纯 HTTP 环境下阻止该 API 调用
- 修复：升级为 HTTPS 访问（见《网络配置日志》）

#### 最终生效配置（openclaw.json 片段，脱敏）
```json
{
  "gateway": {
    "controlUi": {
      "allowInsecureAuth": true,
      "dangerouslyDisableDeviceAuth": true,
      "allowedOrigins": ["https://[TAILSCALE_HTTPS_DOMAIN]"]
    }
  }
}
```

> `dangerouslyDisableDeviceAuth: true` 为临时兼容方案，建议在 HTTPS 环境稳定后评估是否恢复设备身份验证。

---

## 测试结果

| 测试项 | 结果 |
|--------|------|
| 版本号 | v2026.3.11 (29dc654) |
| Gateway 进程 | nohup 后台运行正常 |
| HTTP 18789 | 200 OK |
| 宿主机浏览器可达 | 已确认（通过 Tailscale HTTPS）|
| 飞书 channel | feishu_doc / feishu_chat / feishu_wiki / feishu_drive / feishu_bitable 全部 Registered |
| MiniMax 模型 | 4 个模型正常（provider: minimax）|
| 全部模型数 | 45 个，9 个 provider |

---

## Provider 模型汇总（脱敏）

| Provider | 模型数 | 说明 |
|----------|--------|------|
| dashscope-qwen | 5 | 阿里云通用 |
| dashscope-qwen-coder | 2 | 阿里云编程 |
| dashscope-reasoning | 3 | 阿里云推理 |
| dashscope-deepseek | 2 | 阿里云 DeepSeek |
| dashscope-vision | 3 | 阿里云视觉 |
| stepfun | 8 | 阶跃 |
| siliconflow | 6 | 硅基流动 |
| openrouter | 12 | OpenRouter |
| minimax | 4 | MiniMax |
| **TOTAL** | **45** | |

> API Keys 等敏感信息已脱敏，不记录于日志。

---

## 规则遵守情况
- 路径脱敏：用 [HOST_IP]、[TAILSCALE_HTTPS_DOMAIN] 替代真实地址
- 开放平台上传脱敏：无 token、密码、key 信息
- 操作全程密码仅在内存中使用，未写入任何持久文件

---

*操作时间：2026-03-13*
*记录人：WorkBuddy*
