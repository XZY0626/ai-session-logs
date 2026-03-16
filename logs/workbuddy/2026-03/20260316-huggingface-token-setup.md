# Hugging Face Token 配置与模型下载

**日期：** 2026-03-16  
**操作：** 配置 HF token 并测试下载 pyannote 模型  
**状态：** ✅ 完成

---

## 操作记录

### 1. Token 存储（安全）
- **存储路径：** `/home/xzy0626/.huggingface/token`
- **权限设置：** `chmod 600`（仅所有者可读写）
- **Token 格式：** `hf_ZHTxLAV...TaKS`（已脱敏）

### 2. OpenClaw 配置更新
在 `openclaw.json` 中添加环境变量引用：
```json
{
  "env": {
    "HF_TOKEN": "@file:/home/xzy0626/.huggingface/token"
  }
}
```

**优点：** Token 不会出现在配置文件中，OpenClaw 运行时自动读取文件内容。

### 3. 依赖安装
- **包：** `huggingface_hub`
- **安装方式：** `pip3 install huggingface_hub --break-system-packages`
- **原因：** 系统使用 PEP 668，需要特殊参数安装

### 4. 模型下载测试
- **模型：** `pyannote/speaker-diarization-3.1`
- **状态：** ✅ 下载成功
- **缓存路径：** `/home/xzy0626/.cache/huggingface/hub/models--pyannote--speaker-diarization-3.1/snapshots/84fd25912480287da0247647c3d2b4853cb3ee5d`
- **Token 验证：** 有效，可访问 gated 模型

### 5. 服务重启
- **服务：** openclaw-gateway
- **操作：** `systemctl --user restart openclaw-gateway`
- **状态：** active (running) ✅

### 6. GitHub 同步
- **提交：** `feat: add HF_TOKEN env reference for Hugging Face model downloads`
- **文件：** `openclaw-config/openclaw.json`（脱敏版）
- **变更：** 新增 `env.HF_TOKEN` 配置

---

## 安全说明

1. **Token 权限：** 最小权限配置（仅读取 gated 模型和公共仓库）
2. **存储安全：** 文件权限 600，不在 GitHub 中暴露
3. **访问控制：** 仅 VM 本地可访问，通过文件引用方式传递给 OpenClaw
4. **脱敏处理：** 所有提交到 GitHub 的文件均不含明文 token

---

## 后续使用

在 OpenClaw 的 skill 或 workspace 中，可通过环境变量使用 token：

```python
import os
from huggingface_hub import snapshot_download

token = os.getenv('HF_TOKEN')
model_path = snapshot_download('pyannote/speaker-diarization-3.1', token=token)
```

---

**相关文档：**
- `docs/huggingface-token-setup-guide.md` - HF token 安全配置指南
- `docs/project-cloud-storage-options.md` - 项目云端存储方案

**下次需要：**
- 定期轮换 token（建议每 3-6 个月）
- 如需更多权限，重新创建 token 并更新文件
