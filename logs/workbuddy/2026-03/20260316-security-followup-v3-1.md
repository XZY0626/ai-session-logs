# 2026-03-16 安全自查跟进处理：三项待确认修复 + 安全卫士升级 v1.1.0

## 任务描述

Openclaw 执行安全自查后汇报了三个「待确认」项目，WorkBuddy 逐一核查并全部修复，
同时对安全事件库和 guardian-core.md 安全卫士进行了升级。

## 核查结论摘要

| 问题 | 是否属实 | 严重程度 |
|------|---------|---------|
| healthcheck.sh 未找到 | 部分属实（guardian-core 未定义但功能合理缺失） | P2-低 |
| scrapling 技能未注册 | 属实（安装时跳过了 SKILL_LIFECYCLE Step 5） | P2-低 |
| AI_RULES.md 缺认证旁路禁止条款 | 属实（规则覆盖盲区，物理删文件后未补规则） | P1-中 |

## 执行过程

### Fix 1：scrapling 注册到 AGENTS.md

- 在 AGENTS.md 技能注册表末尾追加 scrapling 条目
- 版本标记：v5 → v6（`MODIFIED v6: scrapling Registered + Auth Bypass Clause + Healthcheck Script`）
- 更新 AGENTS.md sha256 checksum（新哈希：`f479e8f...`）

### Fix 2：AI_RULES.md 新增 L0.10 认证旁路禁止条款

新增 L0.10 章节，明确禁止：
1. 创建 `~/.openclaw/.pre-disable-auth`（Gateway 认证旁路文件）
2. 修改 `openclaw.json` 中 `gateway.auth` / `gateway.bind` 为不安全值
3. 任何绕过 Tailscale HTTPS 直接暴露 Gateway 端口的操作

版本标记：v3 → v4（`SIGNED v4: + Auth Bypass Prohibition (L0.10)`）

### Fix 3：创建 healthcheck.sh

新建 `~/.openclaw/scripts/healthcheck.sh`（v1.0），包含 7 项检查：
1. AGENTS.md sha256 校验
2. `.pre-disable-auth` 不存在验证
3. Gateway 进程运行状态
4. Gateway 端口绑定（127.0.0.1:18789）
5. 供应链 denylist（clawhub/* 封锁验证）
6. HF Token 文件权限（600）
7. requirements hash 锁存在性

首次执行结果：全部 PASS（7/7）

### 安全事件库录入

新增 `security/events/audit-2026-03-16.md`，记录本次内部审计发现的 3 个问题：
- 完整描述根因、影响、修复措施和经验教训
- 属于「内部审计 / 配置缺陷 / 规则盲区」分类

### guardian-core.md 升级 v1.0.0 → v1.1.0

变更内容：
- G1 节：补充 G1.0「启动健康检查」子节，说明 healthcheck.sh 的使用方式和检查项
- G4 节：补充 G4.1b「技能注册完整性」检查，明确每次安装 Skill 必须注册
- 版本记录表：新增 v1.1.0 条目

## 推送记录

| 仓库 | commit | 变更内容 |
|------|--------|---------|
| openclaw-config | `88a4330` | 6 文件，686 行新增，46 行修改 |
| ai-session-logs | 本日志 | 安全跟进记录 |

新增文件：
- `scripts/healthcheck.sh`（新增）
- `workspace/security/events/audit-2026-03-16.md`（新增）
- `workspace/skills/guardian-core.md`（新增，之前仅在虚拟机，未同步到 config 仓库）

修改文件：
- `workspace/AGENTS.md`（v5→v6，scrapling 注册）
- `workspace/AI_RULES.md`（v3→v4，L0.10 条款）
- `workspace/.agents_checksum`（更新哈希）

## 结果

修复后执行 healthcheck.sh 验证全部通过：
```
[PASS] AGENTS.md checksum: 校验通过
[PASS] Gateway 认证: .pre-disable-auth 不存在
...（7/7 PASS）
```

安全评分（参考龙虾自评体系）：9.0 → 9.5/10

## 遗留问题

- guardian-core.md 的脱敏扫描误报：文件内含正则模式字符串 `-----BEGIN.*PRIVATE KEY-----`，
  触发了私钥检测规则（非真实私钥，脱敏替换为占位符，对内容无影响）
- HF Token 90 天轮换提醒仍未配置（低优先级，已在 README 待改进项记录）
