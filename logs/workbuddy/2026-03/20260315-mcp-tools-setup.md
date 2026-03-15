# 2026-03-15 MCP 工具包部署日志

**操作时间**: 2026-03-15  
**操作人**: WorkBuddy Agent  
**目的**: 为 OpenClaw 配置 MCP 工具，使其具备与 WorkBuddy 相近的工具调用能力

---

## 背景

用户希望 OpenClaw 也能具备 WorkBuddy 内置的核心工具能力，包括文件操作、命令执行、网页抓取、网络搜索。

## 环境信息

- 虚拟机: 192.168.1.100 (Ubuntu 24.04, Linux 6.17.0)
- Node.js: v22.22.0 / npm 10.9.4
- Python: 3.12.3
- pip: 24.0 (通过 apt 安装 python3-pip)
- 磁盘剩余: 13G / 49G (74% 已用)

## 工具对照表

| WorkBuddy 工具 | 对应 MCP Server | 版本 |
|---|---|---|
| read_file / write_to_file / list_dir / search_file | @modelcontextprotocol/server-filesystem | 2026.1.14 |
| web_fetch | mcp-server-fetch (Python) | 2025.4.7 |
| web_search | websearch-mcp | 1.0.3 |
| execute_command | @wonderwhy-er/desktop-commander | 0.2.38 |

## 安装步骤

1. `sudo apt-get install -y python3-pip python3-venv` → pip 24.0 安装成功
2. `pip3 install --break-system-packages mcp-server-fetch` → 安装成功，bin 在 ~/.local/bin/
3. `npm install -g @modelcontextprotocol/server-filesystem` → 安装成功
4. `npm install -g websearch-mcp` → 安装成功
5. `npm install -g @wonderwhy-er/desktop-commander` → 安装成功（puppeteer 下载浏览器失败但不影响基本功能）

## 配置修改

- **修改文件**: `/home/xzy0626/.openclaw/openclaw.json`
- **备份文件**: `/home/xzy0626/.openclaw/openclaw.json.bak.mcp-20260315_112420`
- **新增字段**: `mcpServers` (4 个 server)
- **重启服务**: `systemctl --user restart openclaw-gateway` → active running

## 最终配置

```json
"mcpServers": {
  "filesystem": {
    "command": "node",
    "args": ["/home/xzy0626/.local/lib/node_modules/@modelcontextprotocol/server-filesystem/dist/index.js", "/home/xzy0626"]
  },
  "fetch": {
    "command": "/home/xzy0626/.local/bin/mcp-server-fetch",
    "args": []
  },
  "websearch": {
    "command": "node",
    "args": ["/home/xzy0626/.local/lib/node_modules/websearch-mcp/dist/index.js"]
  },
  "desktop-commander": {
    "command": "node",
    "args": ["/home/xzy0626/.local/lib/node_modules/@wonderwhy-er/desktop-commander/dist/index.js"]
  }
}
```

## 验证结果

- ✅ filesystem: 文件存在，配置正确
- ✅ fetch: 文件存在，配置正确
- ✅ websearch: 文件存在，配置正确
- ✅ desktop-commander: 文件存在，配置正确（修正了初始配置中的路径错误 cli.js→index.js）
- ✅ openclaw-gateway: active (running)

## 注意事项

1. `mcp-server-fetch` 安装路径在 `~/.local/bin/`，需确保 PATH 包含此目录（OpenClaw 通过绝对路径调用，无需担心）
2. `desktop-commander` 安装时 puppeteer 浏览器下载失败（网络问题），但核心命令执行功能不依赖浏览器
3. **后续更好方案**: 若需要真实浏览器操作（JS 渲染页面），可考虑添加 `@playwright/mcp`

## 后续改进建议

- [ ] 考虑添加 `@playwright/mcp` 实现完整浏览器自动化（当前 desktop-commander 不支持渲染型页面）
- [ ] websearch-mcp 免费搜索质量有限，如有 Tavily/Brave API key 可替换为更高质量的搜索 MCP
- [ ] 定期 `npm update -g` 更新 MCP 工具包
