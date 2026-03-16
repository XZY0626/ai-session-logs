# HF Token 更换为只读 Token 操作日志

**日期：** 2026-03-16  
**操作：** 更换 HF token 为最小权限只读 token  
**状态：** ⚠️ 需配置网络代理

---

## 操作步骤

### 1. 更新 VM 上的 Token
```bash
# 存储新 token
echo "hf_uDTxhLJ...zAHs" > ~/.huggingface/token
chmod 600 ~/.huggingface/token
```

**结果：** ✅ 文件已更新，权限正确（600）

### 2. 测试新 Token
```bash
python3 -c "from huggingface_hub import whoami; print(whoami(token=...))"
```

**结果：** ❌ 网络不可达 `[Errno 101] Network is unreachable`

### 3. 问题分析
Hugging Face 无法直接访问，需要配置：
- 代理（推荐）
- 镜像源（hf-mirror.com）
- Tailscale 内网穿透

### 4. 文档更新
- 创建 `docs/huggingface-token-update-log.md`
- 记录更换详情、安全问题、后续行动
- 推送至 GitHub

---

## 新旧 Token 对比

| 项目 | 旧 Token | 新 Token |
|------|---------|---------|
| 类型 | 读写 token | **只读 token** |
| 权限 | 过大（含写入） | **最小化（仅读取）** |
| 风险 | 高（泄露后可修改仓库） | **低（仅可下载）** |
| 状态 | 已删除 | ✅ 已启用 |

---

## 安全确认

✅ Token 已更换为只读权限  
✅ Token 文件权限 600  
✅ Token 未在任何日志/配置中明文暴露  
⚠️ 网络访问需要配置代理/镜像源

---

## 后续行动

1. **立即：** 配置 HF 网络访问（代理或镜像）
2. **测试：** 验证 token 可正常下载模型
3. **长期：** 考虑部署 HF 镜像缓存服务

---

**相关文档：**
- `docs/huggingface-token-setup-guide.md` - Token 配置指南
- `docs/huggingface-token-update-log.md` - Token 更换记录
