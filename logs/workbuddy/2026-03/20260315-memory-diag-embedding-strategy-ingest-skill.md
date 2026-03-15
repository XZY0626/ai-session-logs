# 2026-03-15 Memory Search 诊断 + Embedding 通用方案 + 知识投喂 Skill

## 操作摘要

- 诊断确认 Memory Search 完全正常（62/64文件已索引，137 chunks，Vector: ready）
- 澄清 embedding 模型与对话模型的独立关系
- 确认 OpenRouter/StepFun 不提供 embedding API，Qwen embedding 为最优方案
- 新增知识笔记：`memory/2026-03-15-embedding-provider-strategy.md`
- 新增 Skill：`skills/knowledge-ingest.md`（知识投喂操作指南）

## 技术发现

1. **对话模型 vs Embedding 模型完全独立**
   - 对话模型：openrouter/stepfun/step-3.5-flash（负责理解和生成）
   - Embedding 模型：Qwen text-embedding-v3（负责把文字转向量）
   - 两者可以任意组合，互不影响

2. **OpenRouter 不支持 Embedding API**（测试返回 0 个 embedding 模型）

3. **当前方案是最优搭配**，无需更改

## 部署内容

| 文件 | 路径 | 说明 |
|------|------|------|
| embedding-provider-strategy.md | memory/ | Embedding 通用方案知识笔记 |
| knowledge-ingest.md | skills/ | 知识投喂操作指南 Skill |

## 验证结果

- Indexed: 64/64 files（新增2个文件后自动扩展）
- 搜索测试：命中相关文档，score 0.42-0.62
- knowledge-ingest Skill 文件已就位
