---
date: 2026-05-25
type: topic-search
topic: video × agent
period: 2026-04-25 ~ 2026-05-25 (last 30 days)
source: arxiv API (cs.CV / cs.CL / cs.AI / cs.LG)
total_hits: 30
tags: [video-agent, video-understanding, long-video, tool-use, memory, topic-search]
---

# 🎯 Video × Agent 专题搜索（过去 30 天）

## 元信息

- **搜索范围**: arxiv cs.CV / cs.CL / cs.AI / cs.LG
- **关键词组合（13 组查询）**:
  - `abs:"video agent"` / `abs:"video understanding agent"` / `abs:"video reasoning agent"`
  - `abs:"agentic video"` / `abs:"video tool use"`
  - `abs:"agent" AND abs:"long video"`
  - `abs:"agent" AND abs:"video reasoning"` / `abs:"video question answering"` / `abs:"video understanding"` / `abs:"video llm"`
  - `ti:"video" AND ti:"agent"`
  - `abs:"multimodal agent" AND abs:"video"`
  - `abs:"vlm agent" AND abs:"video"`
- **时间窗口**: 2026-04-25 ~ 2026-05-25
- **去重后命中**: 30 篇
- **打分规则**: 命中 `video` / `agent` / `reasoning` / `long video` / `tool` / `planning` / `memory` / `streaming` / `temporal` 加分；`code agent` / `gui agent` / `web agent` / `browser agent` / `computer use` 减分

---

## 🔪 本月 Video × Agent 趋势锐评

**这个方向正处于热点起爆点**。一个月就出了 30+ 篇相关工作，核心模式正在快速收敛：

1. **Video Agent = Video LLM + Tool Use + Memory**：从被动 QA 转向主动取证（active evidence seeking）。模型不再一次性吞下整段视频，而是会"搜索 → 看 → 想 → 再搜索"。
2. **Long video 是主战场**：context window 已经被证明不够用，要靠 **memory + indexing + retrieval** 才能扛分钟到小时级视频。
3. **Tool use 范式分化**：单工具（裁剪关键帧）→ 并行工具（[[ParaVT]]）→ 递归工具（[[ReTool-Video]]）→ 元工具（meta-augmented grounding）。
4. **训练方法**：纯 SFT 已经不够，**RL（GRPO/反事实/工具奖励）** 是标配，多步轨迹蒸馏开始流行。
5. **Egocentric / 长程**: 多篇专攻"smart glasses / 日记式视频"——一整天/一周的连续视频流。

---

## 🏆 必读 Top 5

### 1. Visual Agentic Memory (VAM)
- **arXiv**: [2605.16481](https://arxiv.org/abs/2605.16481) · 2026-05-15 · [Score 13]
- **标题**: Visual Agentic Memory: Enabling Online Long Video Understanding via Online Indexing
- **摘要**: Long video understanding requires more than large context windows. It also needs a memory mechanism that decides what visual evidence to retain, keeps it searchable over long horizons...
- **必读理由**: 把"memory 系统"和"agent 行为"显式拆开——visual memory 不是大 context window，而是带 indexing 的 store，agent 主动决定何时存、何时取。直接命中 `long-term video memory` + `video agent`。🔥

### 2. ReTool-Video
- **arXiv**: [2605.13228](https://arxiv.org/abs/2605.13228) · 2026-05-13 · [Score 11]
- **标题**: ReTool-Video: Recursive Tool-Using Video Agents with Meta-Augmented Tool Grounding
- **摘要**: Video understanding requires active evidence seeking, motivating tool-augmented video agents for temporal reasoning, cross-modal understanding, and complex question answering...
- **必读理由**: 递归工具调用 + meta-augmented tool grounding。"什么时候该调哪个工具"本身做成可学习——video agent 工具范式的下一代。🔥

### 3. VideoSEAL
- **arXiv**: [2605.12571](https://arxiv.org/abs/2605.12571) · 2026-05-12 · [Score 11]
- **标题**: VideoSEAL: Mitigating Evidence Misalignment in Agentic Long Video Understanding by Decoupling Evidence Localization and Reasoning
- **摘要**: Long video question answering requires locating sparse, time-scattered visual evidence within highly redundant content. Although current MLLMs perform well on short videos, long videos...
- **必读理由**: 直击 video agent 的核心痛点——evidence 找错位置就 reasoning 失败。把"定位证据"和"推理"解耦是清晰设计。👀

### 4. EgoMemReason
- **arXiv**: [2605.09874](https://arxiv.org/abs/2605.09874) · 2026-05-11 · [Score 11]
- **标题**: EgoMemReason: A Memory-Driven Reasoning Benchmark for Long-Horizon Egocentric Video Understanding
- **摘要**: Next-generation visual assistants, such as smart glasses, embodied agents, and always-on life-logging systems, must reason over an entire day or more of continuous visual experience...
- **必读理由**: 不是模型而是 benchmark——专门评测"一整天的连续视频"的 memory + reasoning。如果你做长程视频理解的评测，这是必备 benchmark。📏

### 5. Efficient Streaming Video Understanding Framework with Agentic Control
- **arXiv**: [2605.17921](https://arxiv.org/abs/2605.17921) · 2026-05-18 · [Score 7]
- **摘要**: Streaming video requires handling dynamic information density under strict latency budgets. Yet, existing methods typically employ static strategies, such as fixed memory compression...
- **必读理由**: Streaming + Agent 罕见的精准命中。动态决定 memory 压缩策略，不再固定。完美对应 `streaming video understanding`。🔥

---

## 📚 全部 25 篇分主题速览

### A. Long Video + Memory + Agent（最强子方向）

| Score | 论文 | arXiv | 日期 | 一句话 |
|-------|------|-------|------|--------|
| 13 | **Visual Agentic Memory** | [2605.16481](https://arxiv.org/abs/2605.16481) | 05-15 | Online indexing + memory store 支撑长视频 |
| 11 | **EgoMemReason** | [2605.09874](https://arxiv.org/abs/2605.09874) | 05-11 | 长程 egocentric 视频的 memory benchmark |
| 10 | **Bridging Modalities, Spanning Time** | [2605.08271](https://arxiv.org/abs/2605.08271) | 05-08 | 结构化 memory 处理超长 agentic 视频 |
| 8 | **Scaling via Multi-Agent Collab** | [2605.00444](https://arxiv.org/abs/2605.00444) | 05-01 | 多 agent 协作压缩 latent，突破 context budget |

### B. Video Agent + Tool Use

| Score | 论文 | arXiv | 日期 | 一句话 |
|-------|------|-------|------|--------|
| 11 | **ReTool-Video** | [2605.13228](https://arxiv.org/abs/2605.13228) | 05-13 | 递归工具 + meta-augmented grounding |
| 9 | **VideoSeeker** | [2605.16079](https://arxiv.org/abs/2605.16079) | 05-15 | Native agentic tool use 做 instance-level 理解 |
| 9 | **Aurora** | [2605.18748](https://arxiv.org/abs/2605.18748) | 05-18 | Tool-using agent 做统一视频编辑 |
| 8 | **VTAgent** | [2605.04870](https://arxiv.org/abs/2605.04870) | 05-06 | Agentic keyframe anchoring for Video TextVQA |
| 8 | **ParaVT** | [2605.20342](https://arxiv.org/abs/2605.20342) | 05-19 | 并行工具调用的 video RL 训练 |

### C. Video Agent + Reasoning / Evidence

| Score | 论文 | arXiv | 日期 | 一句话 |
|-------|------|-------|------|--------|
| 11 | **VideoSEAL** | [2605.12571](https://arxiv.org/abs/2605.12571) | 05-12 | 解耦 evidence localization 和 reasoning |
| 10 | **MARS** | [2605.18176](https://arxiv.org/abs/2605.18176) | 05-18 | EgoVis 2026 Challenge 系统，多模态 agentic reasoning + source selection |
| 10 | **AffectSeek** | [2605.05640](https://arxiv.org/abs/2605.05640) | 05-07 | Agentic 情感理解 (vague query 场景) |
| 6 | **ViSRA** | [2605.10106](https://arxiv.org/abs/2605.10106) | 05-11 | Video-based 空间推理 agent |

### D. Streaming + Agent

| Score | 论文 | arXiv | 日期 | 一句话 |
|-------|------|-------|------|--------|
| 7 | **Streaming Video w/ Agentic Control** | [2605.17921](https://arxiv.org/abs/2605.17921) | 05-18 | 动态信息密度 + 严格延迟 = agentic control |

### E. 训练数据 / Pipeline / Self-Improvement

| Score | 论文 | arXiv | 日期 | 一句话 |
|-------|------|-------|------|--------|
| 8 | **MAVEN (annotation)** | [2605.21917](https://arxiv.org/abs/2605.21917) | 05-21 | 多阶段 agentic 标注 pipeline for video reasoning |
| 8 | **Long-Context VLM Training** | [2605.13831](https://arxiv.org/abs/2605.13831) | 05-13 | 长上下文 VLM 训练，覆盖视频/文档 |
| 6 | **EvoGround** | [2605.13803](https://arxiv.org/abs/2605.13803) | 05-13 | Self-evolving video agent for temporal grounding |

### F. Video Agent + Generation / Planning

| Score | 论文 | arXiv | 日期 | 一句话 |
|-------|------|-------|------|--------|
| 9 | **A²RD** | [2605.06924](https://arxiv.org/abs/2605.06924) | 05-07 | Agentic autoregressive diffusion 长视频一致性 |
| 7 | **NEWTON** | [2605.18396](https://arxiv.org/abs/2605.18396) | 05-18 | Agentic planning for 物理一致视频生成 |
| 6 | **TIE** | [2605.10543](https://arxiv.org/abs/2605.10543) | 05-11 | Time-interval encoding for 交互式 video agent |
| 6 | **MAVEN (T2V)** | [2605.16716](https://arxiv.org/abs/2605.16716) | 05-16 | 多 agent 框架做多文化 T2V |

### G. Foundation Models / 综合系统

| Score | 论文 | arXiv | 日期 | 一句话 |
|-------|------|-------|------|--------|
| 8 | **GLM-5V-Turbo** | [2604.26752](https://arxiv.org/abs/2604.26752) | 04-29 | 智谱 native 多模态 agent foundation model |

### H. Benchmark / 评测

| Score | 论文 | arXiv | 日期 | 一句话 |
|-------|------|-------|------|--------|
| 11 | **EgoMemReason** | [2605.09874](https://arxiv.org/abs/2605.09874) | 05-11 | 长程 egocentric memory-reasoning |
| 8 | **Spatial-Functional Benchmark** | [2605.02130](https://arxiv.org/abs/2605.02130) | 05-04 | "what they are for" agentic spatial intelligence |
| 7 | **Omni-DeepSearch** | [2605.08762](https://arxiv.org/abs/2605.08762) | 05-09 | Audio-driven omni-modal deep search benchmark |
| 7 | **Multimodal Controversy Detection** | [2605.02939](https://arxiv.org/abs/2605.02939) | 05-01 | Training-free 多模态争议检测，含视频 + 评论 |

---

## 🎯 推荐精读路径

如果只有 1 个晚上：
- 先读 [[Visual Agentic Memory]] → [[ReTool-Video]] → [[VideoSEAL]]
- 这三篇能让你 100% 把握当前 video agent 的核心范式（memory + tool use + evidence localization）

如果有 2-3 天：
- 在上面基础上加 [[EgoMemReason]] (benchmark) + [[Streaming Video Agentic Control]] (流式)
- 横向对比 [[ParaVT]] / [[A²RD]] / [[EvoGround]] 三篇 RL 训练相关
- 看完后基本可以对外讲 "video agent 怎么训、怎么评、怎么部署" 的全图

---

## 📎 附录：原始数据

- 完整结果（含 abstract、authors）已保存到 `/tmp/video_agent_papers.json`（30 篇）
- 搜索查询脚本：见对话记录
- 搜索时间：2026-05-25 23:55
