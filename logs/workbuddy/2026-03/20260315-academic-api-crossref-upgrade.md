# 20260315-academic-api-crossref-upgrade.md

**日期**: 2026-03-15  
**会话主题**: 升级学术搜索工具，加入 Crossref，部署龙虾技能感知文件

---

## 背景

Semantic Scholar 申请表不接受 Outlook 邮箱，中国用户申请门槛持续提高。  
决策：放弃等待 S2 key，改用 Crossref 作为第三源，三源全部无需 key。

---

## 变更内容

### academic_search.py v2.0（四源）
- **新增**: Crossref API 搜索函数 `search_crossref()`
  - 端点: `https://api.crossref.org/works`
  - 字段: DOI、标题、摘要、作者、年份、引用数、期刊名、PDF 链接
  - polite pool: User-Agent + mailto 参数
- **变更**: 默认 `--sources` 从 `arxiv,openalex,s2` 改为 `arxiv,openalex,crossref`
- **变更**: `search_by_doi()` 新增 Crossref DOI 精确查找，S2 仅在有 key 时查
- **路径**: `/home/xzy0626/.openclaw/scripts/academic_search.py`（19960 bytes）

### 新增技能感知文件
- **路径**: `/home/xzy0626/.openclaw/workspace/skills/SKILL_academic_search.md`
- **内容**: 龙虾何时主动调用、如何调用、VoxBridge 专属搜索词表

### 文档更新
- `TOOLS.md`: 三源→四源，更新推荐用法，追加技能文件说明
- `KNOWLEDGE_BASE.md Section 5`: 加入 Crossref 数据源状态行

---

## 测试结果

| 测试 | 结果 |
|------|------|
| 三源默认搜索（arXiv+OpenAlex+Crossref） | ✅ 6篇，Crossref 返回 Whisper-Flamingo 等专业论文 |
| JSON 格式输出 | ✅ 无错误，来源分布 2:2:2 |
| DOI 精确查找 | ✅ 找到 Whisper 原论文 |
| 技能文件部署 | ✅ skills/ 目录可见 |

---

## 当前学术搜索能力汇总

```
arXiv      → CS/ML/Physics 预印本，无需 key ✅
OpenAlex   → 4.74亿篇，无需注册 ✅  
Crossref   → 1.4亿篇 DOI 元数据，无需 key ✅
S2         → 可选，有 key 时加 --sources arxiv,openalex,crossref,s2
```
