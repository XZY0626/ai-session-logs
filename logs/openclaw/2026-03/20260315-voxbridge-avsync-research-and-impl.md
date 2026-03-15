# 2026-03-15 VoxBridge 音画同步前沿研究 + 实现

**时间**: 22:44 - 23:30 (Asia/Shanghai)
**发起人**: 潘神（飞书）
**模型**: hunter-alpha

---

## 1. 前沿论文研究

### 核心论文
- **AlignDiT** (ACM MM 2025, arXiv:2504.20629): 多模态对齐扩散Transformer，无需外部forced aligner
- **Voicebox** (NeurIPS 2023, Meta): 50K+小时flow-matching TTS，zero-shot跨语言
- **HPMDubbing/StyleDubber** (2023-2024): 分层韵律建模，两级风格学习
- **Dubbing in Practice** (TACL 2023): 人类配音时长/节奏规律分析
- **Seamless** (Meta 2023): 多语言流式语音翻译

### 创新点提取
1. 韵律感知时长预测（不只看文字长度，分析pitch/energy轮廓）
2. 动态语速平滑（相邻句变化≤15%，避免机械感）
3. 静音边界感知重排（按音频自然停顿而非标点切分）

---

## 2. 代码实现

### 新增文件
- `src/duration.py` (11KB): 韵律感知时长预测模块
  - DurationPredictor类：提取energy/pitch轮廓，计算韵律权重
  - analyze_silence(): 静音边界检测
  - 支持简单比例回退模式（无音频时）

- `src/boundary.py` (17KB): 静音边界感知重排模块
  - BoundaryReshaper类：对齐、合并、分割三步处理
  - VideoTypeAdapter类：6种视频类型预设
  - DOMAIN_GLOSSARY：课程/游戏/电影/音乐专用术语库

### 更新文件
- `src/tts.py`: 新增
  - smooth_rate_adjustment(): 动态语速平滑函数
  - synthesize_segments_with_prosody(): 增强版合成流程

---

## 3. 视频类型研究

6种视频类型的处理差异分析：
- 课程/教程：慢速、长停顿、高术语密度
- 电影/电视剧：正常语速、情感丰富、背景噪声
- 游戏：快速、高能量、游戏术语
- Vlog：自然口语、过滤填充词
- 音乐：节奏跟随、需要音源分离

---

## 4. 未来优化方向（用户提出）
1. 上下文感知翻译（英语→中文语序调整）
2. 语音克隆 + 情感控制（复刻原始说话人）
3. 更多视频类型测试
