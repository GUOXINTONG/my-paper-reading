---
title: "An Efficient Streaming Video Understanding Framework with Agentic Control"
method_name: "R3-Streaming"
authors: [Jinming Liu, Jianguo Huang, Zhaoyang Jia, Jiahao Li, Xiaoyi Zhang, Zongyu Guo, Bin Li, Wenjun Zeng, Yan Lu, Xin Jin]
year: 2026
venue: arXiv
tags: [streaming-video-understanding, video-llm, agentic-control, memory-compression, adaptive-routing, grpo]
zotero_collection: _待整理
image_source: online
arxiv_html: https://arxiv.org/html/2605.17921v1
created: 2026-05-26
---

# 论文笔记：An Efficient Streaming Video Understanding Framework with Agentic Control

## 元信息

| 项目 | 内容 |
|------|------|
| 机构 | SJTU、Eastern Institute of Technology (Ningbo)、Microsoft Research Asia |
| 日期 | May 2026 |
| 项目主页 | 未公布 |
| 对比基线 | [[TimeChat-Online]]、[[Dispider]]、[[StreamingVLM]]、[[StreamAgent]]、[[Streamo]] |
| 链接 | [arXiv](https://arxiv.org/abs/2605.17921) / [PDF](https://arxiv.org/pdf/2605.17921) |

---

## 一句话总结

> 把流式视频理解建模为「记忆压缩 → 回答时机 → 算力路由」的三级级联决策，靠 age-aware forgetting + TB-GRPO 在裁掉 95% 视觉 token 的同时刷到 SOTA。

---

## 核心贡献

1. **两条经验观察**: ① 历史 token 拿走大部分 attention 却几乎不影响输出分布（删了反而涨点），重要信号集中在最近帧；② 模型规模在流式任务上**非单调**，3B 在 Realtime/Forward 任务上可以打败 7B-Thinking。
2. **R3-Streaming 框架**: 把每一个流式时间步显式拆成 **Remember / Respond / Reason** 三段决策——age-aware 内存压缩 + readiness head 决定是否回答 + 路由策略决定是否调慢模型。
3. **TB-GRPO**: 一个 target-balanced 的 RL 目标，用 piecewise penalty 把 escalation ratio $\rho$ 钳制在目标带 $[\eta-\gamma, \eta+\gamma]$ 内，解决了 vanilla [[GRPO]] 在二元路由上塌缩到「全 escalate」的问题。
4. **结果**: OVO-Bench 57.92、StreamingBench 76.36（streaming MLLM SOTA），同时视觉 token 减 **95%~96%**；在 MLVU/Video-MME 离线长视频上也能涨点。

---

## 问题背景

### 要解决的问题
流式视频理解需要在**严格延迟预算**下处理动态变化的信息密度。每来一帧、每接一个 query，系统都要决定：留多少历史、什么时候答、用多大模型答。

### 现有方法的局限
- **静态策略**：要么固定一套 memory 压缩规则（TimeChat-Online、AdaReTaKe），要么固定调用一个 reactive 模型（Dispider、StreamAgent）。
- **trade-off 死结**：fast model 跑得快但答不对复杂问题；"always-on" 的大模型既违反实时约束，又对简单 query 过度算力。
- **关键能力没耦合**：内存保留、回答时机、推理深度三件事在已有工作里各管各的，**不会动态协同**。

### 本文的动机
作者把它**重新框定为 agentic control 问题**：流式 agent 应该自己管理自己的状态、动态选自己的计算路径，每一步都是「先压缩记忆 → 再判断准不准备好答 → 最后再决定要不要叫大模型」。这是一个级联控制结构，下游决策能复用上游精炼过的信息状态。

---

## 方法详解

### 模型架构

R3-Streaming 是一个 **cascaded 三阶段决策框架**，每个流式 step $t$ 都串行做三件事：

- **输入**: 流式历史 $x_{1:t}$ + 当前 query $q_t$
- **Remember**: 维护一个紧凑 memory state $M_t$（[[Active Forgetting]]）
- **Respond**: 一个 readiness head $h$ 判断当前能否给出 grounded answer
- **Reason**: fast model 自己出答案，或者 emit `<escalate>` token 把 query 转给 slow/thinking model
- **动作空间**: $A = \{\texttt{answer}, \texttt{escalate}, \texttt{wait}\}$
- **Fast/Slow 配置**: 实验里用 Qwen2.5-VL-3B/7B 做 fast，Qwen3-VL-4B/8B-Thinking 或 Qwen2.5-VL-32B 做 slow

### 核心模块

#### 模块1: Remember — Active Forgetting

**设计动机**: Sec.3 的实证发现 1——流式信号高度集中在最近帧，历史 token 大多是噪声且抢走 attention。

**具体实现**:
- 用一个时间窗 $W$（实验默认 3 帧）把历史划为 **Nearby zone** 和 **Historical zone**
- 两段分别用不同压缩阈值 $\tau_{near} \gg \tau_{hist}$（默认 $\tau_{near}=1.0$ 不压、$\tau_{hist}=0.01$ 极致压）
- 压缩算子直接复用 [[TimeChat-Online]] 的 **DTD**（消融表明算子本身不重要，age-aware 策略才是收益来源）
- **training-free**，不引入新参数

#### 模块2: Respond — Proactive Response

**设计动机**: 在连续流里，query 可能比关键证据来得早；早答 = 幻觉。

**具体实现**:
- 在 fast VLM 之外挂一个 **轻量 readiness head $h$**，估计 $p_{ready} = h(q_t, M_t)$
- 阈值 0.5：低于则发 `<wait>` 推迟回答，高于则进入 Reason
- **训练顺序**：先做完 Reason 的 SFT cold-start 和 TB-GRPO，再**冻结** fast VLM，只用 SFT 标签训这个 readiness head（数据构造细节在附录 C）

#### 模块3: Reason — Adaptive Thinking (路由)

**设计动机**: Sec.3 的实证发现 2——大模型不总是更好；只在难 query 上烧算力。

**具体实现**: 两阶段训练 fast model 的二元路由能力——

**Stage 1 — SFT cold-start (Fig. 4)**:
- 对每个训练 query，让 fast model 采样 $K=4$ 个回答 $\{y^{(k)}\}$
- 客观题用 GT 算 binary 正确性；开放题用外部 LLM 打分
- 聚合 $\bar{s} = \frac{1}{K}\sum s^{(k)}$，按阈值 $T$ 决定 ground-truth 路由 token 是 `<answer>` 还是 `<escalate>`
- 这一步只是让模型「学会怎么输出 routing token 的格式」

**Stage 2 — TB-GRPO**:
- 这是本文真正的 RL 创新点，下面公式部分细讲

---

## 关键公式

### 公式1: [[Active Forgetting]] 的双段记忆

$$
M_t = \text{Compress}(x_{t-W+1:t},\ \tau_{near})\ \cup\ \text{Compress}(x_{1:t-W},\ \tau_{hist})
$$

**含义**: 流式历史按温度 $W$ 切两段，近窗按 $\tau_{near}$ 轻压、远窗按 $\tau_{hist}$ 重压。

**符号说明**:
- $W$: 近窗大小（默认 3 帧）
- $\tau_{near}, \tau_{hist}$: 两段压缩阈值，强制 $\tau_{near} \gg \tau_{hist}$（默认 1.0 vs 0.01）
- 压缩算子在实验中用 [[DTD]]，可替换为 [[DivPrune]]/[[VisionZip]]/Pooling 等

### 公式2: Respond 的 readiness gate

$$
p_{\text{ready}} = h(q_t, M_t),\quad
a_t = \begin{cases}\text{emit}\ \texttt{<wait>}, & p_{\text{ready}} < 0.5,\\ \text{continue to Reason}, & \text{otherwise.}\end{cases}
$$

**含义**: 一个轻量 head 在用 fast model 之前先判定证据是否足以 grounded 回答；不足就 defer。

### 公式3: TB-GRPO 的基础奖励

$$
r_i^{\text{naive}} = \begin{cases}2, & e_i=0,\ c_i=1\ (\text{fast 答对})\\ -1, & e_i=0,\ c_i=0\ (\text{fast 答错})\\ 1, & e_i=1,\ c_i=1\ (\text{slow 答对})\\ 0, & e_i=1,\ c_i=0\ (\text{slow 答错})\end{cases}
$$

**含义**: 编码部署优先级——「能自己答对最香（+2）；自己答错最该罚（-1）；调 slow 答对一般（+1）；调 slow 还答错没赏没罚」。

**符号说明**:
- $e_i = \mathbb{I}[y_i = \texttt{<escalate>}]$: 是否升级
- $c_i \in \{0,1\}$: 行动相关正确性
- $\rho = \frac{1}{G}\sum_i e_i$: 组内 escalation ratio

### 公式4: target-band 偏差与最终奖励

$$
\delta_{\text{esc}} = \mathrm{clip}(\rho - (\eta+\gamma),\ 0,\ 1),\qquad
\delta_{\text{ans}} = \mathrm{clip}((\eta-\gamma) - \rho,\ 0,\ 1)
$$

$$
r_i = \begin{cases}(1-\delta_{\text{esc}})\,r_i^{\text{naive}}, & e_i=1,\ c_i=1\\ (1-\delta_{\text{esc}})\,r_i^{\text{naive}} - \delta_{\text{esc}}, & e_i=1,\ c_i=0\\ (1-\delta_{\text{ans}})\,r_i^{\text{naive}}, & e_i=0,\ c_i=1\\ (1-\delta_{\text{ans}})\,r_i^{\text{naive}} - 2\delta_{\text{ans}}, & e_i=0,\ c_i=0\end{cases}
$$

**含义**: 当 $\rho$ 超出目标带 $[\eta-\gamma, \eta+\gamma]$ 时，对越界一侧施加 proportional penalty——**升级太多就压升级奖励、答太多就压回答奖励**，等价于对 escalation frequency 做 proportional 反馈控制。

**符号说明**:
- $(\eta, \gamma)$: 目标带中心和容差（默认 $\eta=0.3, \gamma=0.2$）
- penalty 只在偏离时激活，带内 $\delta = 0$，等价 vanilla GRPO

### 公式5: TB-GRPO 的 clipped policy loss

$$
\mathcal{L}_{\text{TB-GRPO}} = \mathbb{E}\!\left[\min\!\left(w_i A_i,\ \mathrm{clip}(w_i,\ 1-\epsilon_c,\ 1+\epsilon_c)\,A_i\right)\right] - \beta_{\mathrm{KL}}\,D_{\mathrm{KL}}(\pi_\theta\|\pi_{\mathrm{ref}})
$$

**含义**: 标准 PPO 风格的 clipped objective + KL 正则，advantage 是 group-normalized $A_i = (r_i - \bar{r})/(\mathrm{std}(\{r_j\}) + \epsilon)$。

**符号说明**:
- $w_i = \pi_\theta(y_i|x)/\pi_{\theta_{\text{policy}}}(y_i|x)$: 重要性采样比
- 与原 [[GRPO]] 的核心区别只在 $r_i$ 的算法（公式 3+4）

---

## 关键图表

### Figure 1: Empirical Motivations / 两条核心实证

![Figure 1](https://arxiv.org/html/2605.17921v1/x1.png)

**说明**:
- **(a)**: 在 OVO-Bench 上做 deletion 分析——删 Historical token 几乎不动输出分布，删 Nearby token 用 [[JSD]] 衡量 next-token distribution shift 巨大；但 Historical 拿走了大部分 attention mass，说明历史 token 不仅冗余还会**误导注意力分配**。
- **(b)**: 模型规模和流式任务性能不是单调关系——Qwen2.5-VL-3B 在某些 task group 上甚至超过 Qwen3-VL-4B-Thinking。
- **(c)**: task-level deltas 显示 3B 在 Realtime 和 Forward 任务上系统性赢过 thinking 大模型。

### Figure 2: Compression Threshold Ablation / 压缩阈值消融

![Figure 2](https://arxiv.org/html/2605.17921v1/x2.png)

**说明**: 在 OVO-Bench 和 StreamingBench 上扫 $(\tau_{near}, \tau_{hist})$ 网格。最佳点是 $\tau_{near}=1.0, \tau_{hist}=0.01$（StreamingBench 71.0，比不压缩 +2.4 点）。一旦把 Nearby 也压狠，性能崩。直接验证 Finding 1。

### Figure 3: Framework Comparison / 框架对比

![Figure 3](https://arxiv.org/html/2605.17921v1/x3.png)

**说明**: (a) 已有 decision-reaction 方法（[[Dispider]] 这类）依赖单一 reactive 模型；(b) R3-Streaming 把每步显式拆成 Remember → Respond → Reason 三个级联决策模块。这张图是全文 mental model。

### Figure 4: SFT Cold-Start Data Pipeline

![Figure 4](https://arxiv.org/html/2605.17921v1/x4.png)

**说明**: Reason Stage 1 的数据构造——每个 query 让 fast model 采 $K=4$ 回答，客观题对 GT、开放题给 LLM 打分，按平均得分阈值 $T$ 分配 `<answer>`/`<escalate>` 路由标签。

### Figure 5: TB-GRPO 训练管线与 piecewise penalty

![Figure 5](https://arxiv.org/html/2605.17921v1/x5.png)

**说明**:
- **Left**: 训练流程——policy 采样一组路由输出 → 计算 ratio-aware reward → group-normalized advantage → clipped GRPO + KL。
- **Right**: penalty 函数随 $\rho$ 的变化——$\rho < \eta-\gamma$ 时罚不升级（$\delta_{\text{ans}} > 0$），$\rho > \eta+\gamma$ 时罚升级（$\delta_{\text{esc}} > 0$），带内不罚。

### Figure 6: Training Dynamics

![Figure 6](https://arxiv.org/html/2605.17921v1/x6.png)

**说明**: 三种路由 RL 方法的 escalation ratio 和 reward 曲线对比。Vanilla [[GRPO]] 在 step 40 左右塌缩到 $\rho = 1.0$（全部 escalate）；[[AutoThink]] 不塌但停在很高 ratio；**TB-GRPO 稳定收敛到低 ratio + 高 reward**。这张图是本文方法对比 baseline 最有说服力的证据。

### Figure 7: Efficiency vs Performance

![Figure 7](https://arxiv.org/html/2605.17921v1/x7.png)

**说明**: 跨多种 slow model 配置下，adaptive routing 始终在 efficiency-performance 上 Pareto 优于直接调 slow-only inference。这是「为什么要 adaptive 而不是无脑上大模型」的硬证据。

### Table 1: OVO-Bench 主结果

| Model | Real-Time | Backward | Forward | Overall |
|-------|-----------|----------|---------|---------|
| Qwen2.5-VL-7B (offline) | 64.70 | 44.68 | 41.80 | 50.39 |
| Streamo-7B | 65.98 | 46.10 | 54.77 | 55.61 |
| TimeChat-Online-7B (85%↓) | 58.60 | 42.00 | 36.40 | 45.60 |
| StreamAgent-7B | 61.30 | 41.70 | 45.40 | 49.40 |
| FluxMem-7B | - | - | - | 53.40 |
| **R3-Streaming-3B\|4B-Thinking (96%↓)** | 63.58 | 50.23 | 55.83 | 56.55 |
| **R3-Streaming-7B\|4B-Thinking (96%↓)** | **71.89** | **51.27** | 50.60 | **57.92** |

**说明**: streaming MLLM 块里全面 SOTA。和 offline Qwen2-VL-72B (56.27) 比也更高，且只用 3B/7B + 4B-Thinking 的组合，token 数还砍 96%。

### Table 2: StreamingBench 主结果

| Model | Params | All |
|-------|--------|-----|
| GPT-4o | - | 73.28 |
| Qwen2.5-VL-7B (offline) | 7B | 73.28 |
| Dispider | 7B | 67.63 |
| TimeChat-Online-7B (83%↓) | 7B | 73.64 |
| StreamAgent | 7B | 74.28 |
| **R3-Streaming-3B\|4B-Thinking (95%↓)** | 3B/4B | 74.36 |
| **R3-Streaming-7B\|4B-Thinking (95%↓)** | 7B/4B | **76.36** |

**说明**: 超过 GPT-4o 3 个点、超 StreamAgent 2 个点；最大亮点是同时**砍 95% 视觉 token**。

### Table 4: Base Module Ablation

| 配置 | StreamingBench | OVO-Bench |
|------|----------------|-----------|
| Baseline (Qwen2.5-VL-3B) | 68.6 | 51.4 |
| w/ Remember | 71.0 | 52.8 |
| w/ Reason | 73.4 | 55.9 |
| **w/ Both** | **74.4** | **56.6** |

**关键发现**: Remember 和 Reason 是**互补**的，单上一个就涨点，两个一起才到最佳。Respond 单独在 Proactive split 评估。

### Table 7: Reason 路由策略对比

| Method | StreamingBench | Escalate Ratio |
|--------|----------------|----------------|
| SFT-only | 73.16 | 100.0% |
| Vanilla GRPO | 73.16 | 100.0% |
| AutoThink | 70.00 | 53.2% |
| **TB-GRPO** | **74.36** | **24.0%** |

**关键发现**: SFT 和 vanilla [[GRPO]] 都塌缩到 100% escalate（等价于 slow-only，没省成本）；AutoThink 不塌但点掉了；TB-GRPO 在 24% escalate ratio 下点反而更高，证明**有方向的 reward shaping 才能让二元路由学到「该升级才升级」**。

---

## 实验

### 数据集

| 数据集 | 规模 | 特点 | 用途 |
|--------|------|------|------|
| TimeChat-Online-139K | 139K | 流式视频指令数据 | Respond/Reason 训练 |
| StreamingBench | - | 在线 QA，10 个子任务 | 主评测 |
| OVO-Bench | - | causal online QA（Real-Time/Backward/Forward） | 主评测 |
| MLVU | - | 长视频理解 | 离线泛化 |
| Video-MME | - | 长视频理解 | 离线泛化 |

### 实现细节

- **Fast model**: Qwen2.5-VL-3B / 7B
- **Slow model 候选**: Qwen3-VL-4B/8B-Thinking、Qwen2.5-VL-32B
- **压缩算子**: DTD（from [[TimeChat-Online]]）
- **默认阈值**: $\tau_{near}=1.0, \tau_{hist}=0.01$, $W=3$ 帧
- **TB-GRPO**: $\eta=0.3, \gamma=0.2$
- **离线长视频用**: 把 $\tau_{hist}$ 调到 0.5（因为离线视频证据更均匀分布），这是 deterministic toggle 不是 per-benchmark 调参

### 可视化结果

附录 Fig. 11 的 subtask 级分析：感知型任务（Object Perception）在 Nearby=1.0 时既涨点又自然降低 escalation；认知型任务（Prospective Reasoning、Counting）不管 memory 怎么调都需要高 escalation——这证明 routing policy 确实是 **task-driven** 而非随机。

---

## 批判性思考

### 优点
1. **问题诊断硬核**：Fig.1a 的 JSD 实验是真的把「历史 token 抢 attention 但不影响输出」这件事量化出来了，不是 hand-waving。这种 motivation 强度在当下流式视频灌水里很罕见。
2. **TB-GRPO 是个干净的方法贡献**：vanilla [[GRPO]] 在二元路由上塌缩是真实痛点，piecewise penalty 用两个超参 $(\eta, \gamma)$ 给出**显式可调**的控制接口，部署时能直接按算力预算设 escalation ratio——这是工程友好型设计。
3. **拆解 + 互补性证据齐**：Table 4 证明三模块互补、Table 7 证明 TB-GRPO 必要、Fig. 6 证明训练稳定性，证据链完整。
4. **token 减 95% 还涨点**：不是「省算力但掉点」的妥协式 efficiency，是「省得更狠还更准」——这反过来印证 Finding 1 的发现是真的。

### 局限性
1. **Respond 评估单薄**：Table 6 只在 Proactive split 上比，readiness head 真实可靠性（FP/FN 分布、过晚答的代价）没系统量化。
2. **Worst-case latency 没解决**：作者自己 Limitations 也承认——TB-GRPO 控制的是**平均** escalation ratio，遇到连续高信息密度片段时的 latency spike 没办法。这对真实部署是大坑。
3. **依赖 fast model 的自知之明**：路由全靠 fast model 自己输出 `<escalate>` token——如果 fast model 在某类难题上「自信地答错」（dunning-kruger），TB-GRPO 训出来的 ratio control 也救不了。Table 7 里 SFT-only 100% escalate 说明 fast model 默认行为是「能塌就塌」。
4. **离线/在线靠人手切阈值**：MLVU/Video-MME 上把 $\tau_{hist}$ 从 0.01 调到 0.5，被作者轻描淡写为「deterministic toggle」，但这说明 age-aware 策略对 **evidence 分布**的假设并不普适，一旦视频证据均匀分布就要重调。
5. **没有真实流式 latency 数字**：Fig.7 是 efficiency-performance 散点，但没给「真实端到端 wall-clock latency」分布——3B+4B-Thinking 的串行调用栈在边缘设备到底跑多快没说。
6. **没和最新 RL 路由方法横向比够**：[[AutoThink]]、[[AdaptThink]] 之外还有不少 think-or-not 的工作，对比池其实有点窄。

### 潜在改进方向
1. 把 Respond 也纳入 RL 优化，让 readiness threshold 也变 learnable 而非固定 0.5。
2. Worst-case 用一个 token-budget RL（带 hard constraint）替代 average constraint。
3. Routing 决策可以引入 calibrated uncertainty（如 SelfCheck、Verbalized Confidence）做交叉验证而非纯依赖 fast model 自报。
4. 把 age-aware 推到 query-aware——不同 query 类型用不同的 $(\tau_{near}, \tau_{hist})$。

### 可复现性评估
- [ ] 代码开源（论文未注明）
- [ ] 预训练模型
- [x] 训练细节完整（超参、训练数据、阶段清晰）
- [x] 数据集可获取（用公开 TimeChat-Online-139K + 公开评测）

---

## 关联笔记

### 基于
- [[TimeChat-Online]]: 压缩算子 DTD 直接复用，训练集 TimeChat-Online-139K 也来自这里
- [[GRPO]]: TB-GRPO 是在 vanilla GRPO 基础上加 target-band reward shaping

### 对比
- [[Dispider]]: 同样 decision-reaction 架构，但单 reactive model 路线
- [[StreamAgent]]: 流式 agent，但没显式做 routing
- [[StreamingVLM]]: streaming 基线
- [[AutoThink]]: think-or-not 路由 RL，但不控 escalation frequency
- [[AdaptThink]]: 类似 think mode switching，非 streaming budget 设计

### 方法相关
- [[Active Forgetting]]: 本文核心策略
- [[GRPO]]: TB-GRPO 的母体
- [[JSD]]: Finding 1 用来量化 token 重要性
- [[Adaptive Reasoning]]: think-or-not 范式
- [[DTD]]: 压缩算子

### 评测/数据
- [[StreamingBench]]: 主评测
- [[OVO-Bench]]: 主评测，causal online QA
- [[MLVU]]: 离线长视频泛化
- [[Video-MME]]: 离线长视频泛化

### 流式视频体系
- [[Streaming Video Understanding]]: 总主题
- [[OmniPro]]: 同期 proactive streaming benchmark
- [[CRPO]]: 同期 video LLM counterfactual RL

---

## 速查卡片

> [!summary] R3-Streaming
> - **核心**: 流式视频理解 = 级联 agentic 决策（Remember → Respond → Reason），TB-GRPO 解 RL 路由塌缩
> - **方法**: age-aware forgetting 压缩历史 token + readiness head 控回答时机 + TB-GRPO 控 escalation ratio
> - **结果**: OVO-Bench 57.92 / StreamingBench 76.36 SOTA，视觉 token 减 95%~96%
> - **代码**: 论文未给出仓库链接

---

*笔记创建时间: 2026-05-26*
