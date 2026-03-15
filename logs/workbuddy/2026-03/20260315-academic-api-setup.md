# 20260315-academic-api-setup.md

**日期**: 2026-03-15  
**会话主题**: 为 VoxBridge 配置学术论文 API，部署三源搜索工具到 openclaw

---

## 操作摘要

### 测试结果
| API | 状态 | 说明 |
|-----|------|------|
| arXiv | ✅ 正常 | 无 key，加 0.4s 延迟，稳定可用 |
| OpenAlex | ✅ 正常 | 无需注册，10000 req/天，当前余量 9990 |
| Semantic Scholar（无 key）| ❌ 429 | 公网共享池限流，需申请免费 key |

### 部署内容
1. **脚本**: `/home/xzy0626/.openclaw/scripts/academic_search.py`（16KB，三源合并搜索）
2. **知识库**: `KNOWLEDGE_BASE.md` 新增 Section 5（学术论文 API 配置）
3. **工具文档**: `TOOLS.md` 新增学术论文搜索工具说明

### openclaw 运行状态
- gateway 服务: ✅ active (running)，运行 3h 40min
- 内存占用: 1.7G（peak 2.0G）

---

## academic_search.py 功能

```
用法: python3 ~/.openclaw/scripts/academic_search.py <关键词> [参数]
参数:
  --sources arxiv,openalex,s2  （默认三源，无 S2 key 时用 arxiv,openalex）
  --limit N                    每源返回 N 篇（默认 8）
  --year-from YYYY             年份下限
  --json                       JSON 输出
  --doi <DOI>                  按 DOI 精确查找
  --s2-key <KEY>               Semantic Scholar API key
```

---

## 待办

- [ ] 申请 Semantic Scholar API key（Outlook 邮箱，地址: https://www.semanticscholar.org/product/api）
- [ ] 获得 key 后：`export S2_API_KEY="<key>"` 加入 `~/.bashrc`，三源全部启用
