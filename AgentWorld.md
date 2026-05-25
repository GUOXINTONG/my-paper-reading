---
title: "Agent-World: Scaling Real-World Environment Synthesis for Evolving General Agent Intelligence"
method_name: "AgentWorld"
authors: [Guanting Dong, Junting Lu, Junjie Huang, Wanjun Zhong, Zhicheng Dou, ByteDance Seed, Renmin University of China]
year: 2026
venue: arXiv
tags: [agent-rl, environment-scaling, mcp, self-evolving-agent, tool-use, grpo]
zotero_collection: 论文笔记
image_source: online
arxiv_html: https://arxiv.org/abs/2604.18292
created: 2026-05-25
---

# 论文笔记：Agent-World

## 元信息

| 项目 | 内容 |
|------|------|
| 机构 | 人民大学高瓴 AI 学院 + ByteDance Seed |
| 日期 | April 2026 |
| 项目主页 | [agent-tars-world.github.io](https://agent-tars-world.github.io/-/) |
| 对比基线 | [[EnvScaler]], [[TOUCAN]], [[AWM]], [[ScaleEnv]] |
| 链接 | [arXiv](https://arxiv.org/abs/2604.18292) / [PDF](https://arxiv.org/pdf/2604.18292) |

---

## 一句话总结

> 通过自动挖掘 1978 个真实世界环境 + 19822 个可执行工具构建 [[Agent RL]] 训练 arena，并用诊断 agent 驱动环境-策略协同进化，让 8B/14B 小模型在 23 个 agent benchmark 上对标甚至超越 685B 大模型。

---

## 核心贡献

1. **真实环境大规模合成**: 从 [[MCP|Model Context Protocol]] 服务器、工具文档、产业 PRD 三类来源出发，用 deep-research agent 自动从 Web 挖掘 topic-aligned 数据库 + 可执行工具集，规模达 **1978 个环境 / 19822 个工具**
2. **双策略可验证任务合成**: [[Graph-Based Task Synthesis]] 处理顺序依赖（工具图随机游走），[[Programmatic Task Synthesis]] 处理条件分支/循环/聚合
3. **自进化 Agent Arena**: 用诊断 agent δ 分析失败 trace → 定位弱环境 → 定向生成难任务 → 继续 [[GRPO]] 训练，形成协同进化闭环
4. **可观测的扩展规律**: 环境数 0→1978，下游 agent 平均分 18.4%→38.5%（+20.1）；自进化轮数 vs 性能也呈正相关

---

## 问题背景

### 要解决的问题

通用 agent 需要在**真实、有状态、组合性强**的环境中训练（如订机票需要 check inventory → book → update calendar 的有效顺序，每个动作都改变状态）。当前 agent RL 多在 stateless / 单工具场景训练，难以应对真实开放世界。

### 现有方法的局限

| 方法 | 优势 | 缺陷 |
|------|------|------|
| **模拟环境**（LLM 当 world model） | 可扩展性强 | 幻觉严重，偏离真实动态 |
| **程序化环境**（EnvScaler / AWM / AutoForge） | 有真实数据库 + 可执行工具 | 单轮静态训练，缺少持续诊断与改进机制 |

两大瓶颈：
- **真实性与复杂度兼顾困难**: 纯 LLM 合成难以匹配真实交互逻辑
- **缺乏自进化机制**: 现有工作只做环境构造，没有用环境主动诊断 agent 弱点并定向训练的闭环

### 本文的动机

- World Wide Web 本身就有海量真实结构化数据，应该**直接挖**而不是合成
- 真实环境本质上就是天然的诊断 arena，可以同时用来训练和找弱点

---

## 方法详解

### 整体架构

Agent-World 由两个紧耦合组件构成闭环：

- **输入**: 真实世界环境主题 $m \in \mathcal{M}$（来自 MCP Servers + 工具文档 + 产业 PRD）
- **核心模块 1**: [[Agentic Environment-Task Discovery]] —— 挖环境 + 造任务
- **核心模块 2**: [[Continuous Self-Evolving Agent Training]] —— 多环境 RL + 自进化 arena
- **输出**: 经过 R 轮共同进化的 agent 策略 $\pi_{\theta^{(R)}}$
- **训练 backbone**: Qwen3-8B / Qwen3-14B，冷启动 SFT 用 40K 轨迹 + RL 用 5K 样本

### 核心模块 1: Agentic Environment-Task Discovery

#### 1.1 环境主题收集

三类来源构成主题集 $\mathcal{M} = \mathcal{M}_1 \cup \mathcal{M}_2 \cup \mathcal{M}_3$:
- $\mathcal{M}_1$: Smithery 平台真实 MCP Server 规范
- $\mathcal{M}_2$: 开源数据集中的工具文档，用 LLM 逆向推主题
- $\mathcal{M}_3$: 产业 PRD（产品需求文档）

#### 1.2 Agentic 数据库挖掘

- 设计 deep-research agent $\mathcal{G}$，配备 search / browser / 代码编译器 / OS 工具
- 对每个主题 $m$ 迭代深度信息检索 → 用 OS 工具结构化存储
- **数据库复杂化** $\phi$ 迭代 $N$ 轮，把简单数据库扩成复杂数据库

#### 1.3 工具接口生成与验证

- 编码 agent $\psi$ 给每个工具 $\hat{f}$ 同时生成单元测试集 $\hat{\mathcal{C}}_{\hat{f}}$
- **执行验证过滤**：编译通过 + 测试通过率 > 0.5 才保留
- 最终 1978 个有效环境，每个平均 >10 个工具，总计 19822 个工具

#### 1.4 环境分类法构建

层级聚类得到 50 个二级类，3 个标注员合并为 20 个一级类。最终：**20 一级 / 50 二级 / 2K+ 三级** 标签的层级 taxonomy。

#### 1.5 双策略可验证任务合成

##### Graph-Based Task Synthesis（处理顺序依赖）

对每个环境构造工具图 $G=(V, E)$，三种边：

| 边类型 | 权重 | 含义 |
|--------|------|------|
| 强依赖 $f_i \to f_j$ | $w=3$ | $f_j$ 输入严格依赖 $f_i$ 输出 |
| 弱依赖 $f_i \leftrightarrow f_j$ | $w=2$ | 可由 $f_i$ 提供也可来自其他途径 |
| 独立边 $f_i \leftrightarrow f_j$ | $w=1$ | 无依赖，保证图连通的 fallback |

**流程**: 随机游走 $\tau = [f_1, ..., f_k]$ → 参数实例化 → LLM 精炼成 $\tau^*$ → 沙盒执行 → 反推任务描述 $q_{init}$（**禁止包含工具名/数据库 schema 防止泄露**）→ 生成 ground truth $a^*$ + 评分 rubric $R$

##### Programmatic Task Synthesis（处理条件/循环/聚合）

- LLM 直接生成可执行 Python 脚本 $\pi_{code}$，含 for / if-else / 聚合等复杂控制流
- ReAct loop 在沙盒中调试代码直到无错
- 同时生成验证脚本 $V_{code}(a, a^*)$，做多级断言

##### 一致性验证（两种合成共用）

> 用 [[ReAct]] agent 跑 5 次，**至少 2 次成功**才保留，保证任务"难但可解"

##### 难度缩放

- Graph-based: 增加 walk 步数 + 提高弱依赖/独立边采样概率 + 重写任务描述抹去显式工具名
- Programmatic: 增加工具数 + 引入跨数据库聚合/排序/过滤

### 核心模块 2: Continuous Self-Evolving Agent Training

#### 2.1 多环境 Agent RL

- **三方闭环**: LLM 策略 $\pi_\theta$ + 工具运行时 + 数据库 $\mathcal{D}^{(N)}(m)$
- **POMDP 建模** $(U, S, A, O, P)$，$S = S_E \times S_H$（环境状态 + 对话状态）
- **结构化可验证奖励** —— 两类任务独立计算（详见公式 1）
- **策略更新**: [[GRPO]]，clip ratio $\varepsilon_{low}=0.2$, $\varepsilon_{high}=0.28$（参考 [[DAPO]]）

#### 2.2 自进化 Agent Arena 四步循环

1. **分层采样构造 arena**: 每个一级类 $c$ 取 $K=5$ 个环境
2. **动态任务合成**: 每轮重新合成新任务（防过拟合）
3. **诊断 agent δ**: 输入失败 trace + 错误分布 + 环境元数据，输出弱环境集 $\mathcal{W}^{(r)}$ + 任务生成指南 $\mathcal{G}_{guide}^{(r)}$
4. **协同进化**: 在弱环境上做数据库复杂化 + 定向任务合成 + 继续 RL

---

## 关键公式

### 公式 1: [[结构化可验证奖励]]

$$
r(x, y) = \begin{cases}
\mathbb{I}\Big[\frac{1}{n}\sum_{j=1}^{n} \mathbb{I}\big[\mathrm{Judge}(x, y, r_j)\big] == 1\Big], & x \in \mathcal{X}_{\text{graph}} \\
\mathbb{I}\big[\mathrm{Execute}(V_{\text{code}}(y, y^*))\big], & x \in \mathcal{X}_{\text{prog}}
\end{cases}
$$

**含义**: 两类任务用不同验证方式 —— graph 任务用 LLM-as-judge 按 rubric 打分（**全 rubric 通过才得 1**），programmatic 任务用沙盒跑验证脚本得 binary 信号

**符号说明**:
- $\mathbb{I}[\cdot]$: 指示函数
- $\mathrm{Judge}(x, y, r_j)$: rubric-conditioned LLM 评判器
- $V_{code}$: 任务特定的 Python 验证脚本
- $y^*$: ground truth 答案

### 公式 2: [[GRPO]] 目标函数

$$
J_{\text{GRPO}}(\theta) = \mathbb{E}_{x \sim D, \{y_i\} \sim \pi_{\theta_{\text{old}}}} \Big[\frac{1}{G}\sum_{i=1}^{G}\frac{1}{|y_i|}\sum_{t=1}^{|y_i|}\min\big(r_{i,t}(\theta)\hat{A}_{i,t}, \mathrm{clip}(r_{i,t}(\theta), 1-\epsilon, 1+\epsilon)\hat{A}_{i,t}\big) - \beta D_{KL}(\pi_\theta \| \pi_{ref})\Big]
$$

**含义**: 每个任务采 $G$ 条轨迹，组内归一化 advantage，带 clip 和 KL 约束的策略梯度

**符号说明**:
- $G$: 每个任务的 rollout 数量
- $\hat{A}_{i,t}$: 第 $i$ 条轨迹在 $t$ 时刻的 token-level normalized advantage
- $\epsilon, \beta$: clip 范围与 KL 系数

### 公式 3: 数据库复杂化迭代

$$
\mathcal{D}^{(n+1)}(m) = \phi\big(\mathcal{D}^{(n)}(m), m, \mathcal{T}\big), \quad n = 0, \dots, N-1
$$

**含义**: 用 deep-research agent 迭代扩展和丰富主题特定的数据库，N 轮后得到 $\mathcal{D}^{(N)}(m)$

### 公式 4: 自进化循环

$$
\pi_{\theta^{(r)}} \xrightarrow{\text{evaluate}} \mathcal{W}^{(r)} \xrightarrow{\text{diagnose+target}} \mathcal{X}^{(r)}_{\text{target}} \xrightarrow{\text{continue RL}} \pi_{\theta^{(r+1)}}
$$

**含义**: 评估 → 诊断弱环境 → 定向任务合成 → 继续 RL 的循环

---

## 关键图表（文字描述）

### Figure 1: Agent-World 总体框架与环境扩展性能

左侧两阶段闭环设计；右侧报告 MCP-Mark / BFCL V4 / $\tau^2$-Bench 在环境数量增加时的平均分提升曲线，呈现明显的正向 scaling 趋势。

### Figure 4: 环境与任务统计（6 个子图）

- (a) 1978 个环境覆盖多种类型
- (b) 每个环境平均 >10 个工具，部分 >40
- (c) 总计 19822 个不同工具
- (d) 数据库文件格式多样：json、csv、sql、html、tex、yaml
- (e) 所有任务至少 7 轮交互，平均 >20 轮，部分 >40 轮
- (f) Pass@10 验证显示难度合理：少数任务全过，多数仅 1/10 通过

### Figure 8: 训练环境数量的 scaling 规律

环境数 0 → 10 → 100 → 500 → 1000 → 1978，四个代表性领域平均分从 **18.4% 升至 38.5%（+20.1）**。10→100、100→500 涨幅最大；500→2000 仍上升但边际递减。

### Table 1: 三大 agent benchmark 主结果（节选）

| Method | MCP-Mark Avg | BFCL V4 Avg | $\tau^2$-Bench Avg |
|--------|--------------|-------------|---------------------|
| GPT-5.2 High | 53.1 | 62.9 | 80.2 |
| Claude Sonnet-4.5 | 33.3 | 73.2 | 84.7 |
| Gemini-3 Pro | 50.8 | 72.5 | 85.4 |
| DeepSeek-V3.2-685B | 36.7 | 54.1 | 80.3 |
| Qwen3-8B | 2.4 | 40.4 | 26.2 |
| Qwen3-14B | 3.4 | 41.0 | 32.4 |
| Qwen3-235B-A22B | 5.8 | 47.9 | 58.5 |
| EnvScaler-8B | 5.6 | 47.6 | 37.9 |
| AWM-14B | 5.1 | 42.4 | 39.0 |
| **Agent-World-8B** | **8.9** | **51.4** | **61.8** |
| **Agent-World-14B** | **13.3** | **55.8** | **65.4** |

**关键发现**:
- 14B 在 BFCL-V4 上 **55.8%**，**超过 685B 的 DeepSeek-V3.2 (54.1%)**
- $\tau^2$-Bench Airline 子项 14B 达 52.0%，远超 Qwen3-235B-A22B (45.6%)
- MCP-Mark 上 14B 比 Qwen3-14B 高 +9.9 个百分点

### Table 2: 自进化循环效果

| Model / Round | $\tau^2$-Bench | BFCL-V4 | MCP-Mark (Postgres) |
|----------|------------|---------|------------------|
| Agent-World-14B (base) | 60.2 | 52.4 | 29.5 |
| +1 round | 63.5 (+3.3) | 54.9 (+2.5) | **36.3 (+6.8)** |
| +2 rounds | 65.4 (+1.9) | 55.8 (+0.9) | 38.1 (+1.8) |
| EnvScaler-8B (base) | 37.9 | 47.6 | 9.5 |
| +1 round | 40.2 (+2.3) | 49.1 (+1.5) | **13.9 (+4.4)** |
| +2 rounds | 41.6 (+1.4) | 50.0 (+0.9) | 15.1 (+1.2) |

**关键发现**:
- 自进化对 **MCP-Mark 提升最大**（需要复杂状态追踪），Agent-World +8.6%，EnvScaler +5.6%
- 自进化对其他 baseline（EnvScaler）也有效，证明方法的通用性
- 第二轮收益递减但仍正

---

## 实验

### 数据集

| 数据集类型 | 数量 | 用途 |
|------------|------|------|
| 合成环境 | 1978 个 | RL 训练 + arena 评估 |
| 合成工具 | 19822 个 | 跨环境工具调用训练 |
| 冷启动 SFT 轨迹 | 40K | Qwen3 backbone 初始化 |
| RL 训练样本 | 5K | GRPO 训练 |

### 评测基准（23 个）

- **核心 agent**: MCP-Mark, BFCL V4, $\tau^2$-Bench
- **高级 Assistant**: SkillsBench, ARC-AGI-2, Claw-Eval
- **通用推理**: MATH500, GSM8K, MATH, AIME24/25, KOR-Bench, OlympiadBench
- **Search & Coding**: WebWalkerQA, SWE-Bench Verified/Multilingual, Terminal-Bench 1.0/2.0, GAIA, HLE
- **Knowledge & MCP**: MMLU, SuperGPQA, MCP-Universe 5 子领域

### 实现细节

- **Backbone**: Qwen3-8B / Qwen3-14B
- **环境挖掘 / 任务合成 / 诊断 policy**: GPT-OSS-120B
- **冷启动 SFT 数据生成**: in-house Doubao-Seed-1.8
- **优化器**: GRPO with clip $\varepsilon_{low}=0.2, \varepsilon_{high}=0.28$
- **轨迹长度**: max 80K tokens，单步生成 max 32K
- **采样**: 每步 32 个任务 × 8 rollouts, temperature=1.0, top_p=1.0
- **评估**: 重复 8 次取平均

---

## 批判性思考

### 优点

1. **真实性 vs 可扩展性的好权衡**: 利用 Smithery MCP Server 和真实 Web 数据库，避免了纯 LLM 模拟的幻觉问题，又通过自动化挖掘保证规模
2. **任务合成方法论扎实**: graph + programmatic 两种合成路径互补，覆盖了顺序依赖和复杂控制流；ReAct 5 次跑 2 次过的过滤策略简单有效
3. **自进化机制有诊断闭环**: 不是简单的 self-play 或随机数据扩展，而是用诊断 agent 分析 trace 定位弱环境，再做定向训练
4. **小模型对标大模型**: 14B 模型在 BFCL-V4 超过 685B DeepSeek-V3.2，证明环境质量比模型尺寸重要
5. **scaling 规律清晰**: 环境数量与性能、自进化轮数与边际效益的曲线都很有说服力
6. **泛化能力强**: 在没专门训练的 23 个 benchmark 上几乎都有提升，说明学到的是 transferable interaction logic

### 局限性

1. **MCP-Mark 绝对分数仍然不高**: Agent-World-14B 在 MCP-Mark Avg 只有 13.3%，远低于 GPT-5.2 (53.1%)，说明面对最难的真实 MCP 场景仍有显著差距
2. **依赖大模型生成数据**: 环境挖掘、任务合成、诊断都用 GPT-OSS-120B，本质上是大模型蒸馏到小模型，开源可复现性受限
3. **任务合成质量不可控**: 虽然有 ReAct 5 次跑 2 次过的过滤，但 LLM 生成的任务描述、rubric、验证脚本本身可能有偏差
4. **自进化轮数有限**: 论文只跑了 2 轮，长期是否会饱和或退化未知
5. **缺失部分基线数据**: Table 1 中 DeepSeek-V3.2 在 $\tau^2$-Bench 的 Telecom/Airline 数据缺失
6. **没有消融**: 没看到分别消融 graph synthesis vs programmatic synthesis、自进化 arena vs 单纯 RL 的对比
7. **诊断 agent 自身是黑盒**: 用大模型当诊断器，诊断准确性如何评估？是否会引入新的 bias？

### 潜在改进方向

1. **跨环境迁移学习**: 显式建模环境间的相似度
2. **更细的诊断粒度**: 现在按"弱环境"，可以细化到"弱工具组合"或"弱状态转移类型"
3. **环境与任务的联合 scaling law**: 任务难度、轮数、工具复杂度的联合作用值得深入
4. **多 agent 协作的环境**: 当前都是单 agent vs 工具

### 可复现性评估

- [ ] 代码开源（需查项目主页）
- [ ] 预训练模型
- [x] 训练细节完整
- [ ] 数据集可获取（1978 个环境是否开源？）
- [x] 关键 prompt 公开（Appendix 7 有诊断 agent prompt）

---

## 关联笔记

### 基于

- [[GRPO]]: 策略优化算法
- [[ReAct]]: 推理-行动框架，用于任务一致性验证
- [[MCP|Model Context Protocol]]: 真实环境主题的主要来源
- [[POMDP]]: 多环境交互的形式化建模
- [[DAPO]]: clip 不对称设置的参考

### 对比

- [[EnvScaler]]: 程序化环境合成的代表，被本文当作 baseline 且也用本文方法提升
- [[TOUCAN]]: 反向工程的任务合成（先合成工具链再生成任务），本文继承并扩展
- [[AWM|AgentWorldModel]]: 另一个环境合成 baseline
- [[ScaleEnv]]: 环境合成 baseline
- [[Simulator-8B]]: LLM 模拟环境路线的代表

### 方法相关

- [[Agentic Environment-Task Discovery]]: 本文核心方法 1
- [[Continuous Self-Evolving Agent Training]]: 本文核心方法 2
- [[Graph-Based Task Synthesis]]: 处理顺序依赖
- [[Programmatic Task Synthesis]]: 处理条件/循环/聚合
- [[Self-Evolving Agent Arena]]: 协同进化循环

### 评测相关

- [[MCP-Mark]], [[BFCL V4]], [[tau2-Bench]], [[SWE-Bench]], [[Terminal-Bench]], [[GAIA]], [[HLE]]

---

## 速查卡片

> [!summary] Agent-World
> - **核心**: 真实环境挖掘 + 自进化训练，让 8B/14B agent 对标大模型
> - **方法**: 1978 个 web-mined 环境 + graph/programmatic 双策略任务合成 + 诊断驱动的协同进化 GRPO
> - **结果**: 23 个 benchmark 全面领先开源 baseline，14B 在 BFCL-V4 超过 DeepSeek-V3.2 685B
> - **代码**: [agent-tars-world.github.io](https://agent-tars-world.github.io/-/)

---

## 我的快速点评

这是一篇典型的"**通过更好的数据来训练更小的模型对标大模型**"的工作，但比同类工作高出一截的地方在于：

1. **不是简单的数据 scaling，而是有诊断闭环的自进化**——这才是真正的"co-evolution"，不是噱头
2. **真实数据来源（Smithery MCP / Web 挖掘）远比 LLM 模拟靠谱**，从 Table 1 看 Simulator-8B 表现最差就能印证
3. **对 MCP 生态的押注是对的**，2026 年随着 MCP 标准化，基于真实 MCP server 的训练范式会越来越主流

值得追踪的点：
- 项目主页是否开源完整的环境集和工具集（这才是真正的核心资产）
- 后续是否会推广到多模态 agent（论文目前只针对文本工具调用）
- Doubao-Seed / GPT-OSS-120B 等大模型生成数据，对小模型的"蒸馏天花板"在哪

对于做 VLA / 多模态 agent 的我们，这套思路也可以借鉴：
- **环境主题收集** → 真实机器人任务库（Atlas / SimplerEnv / RoboCasa）
- **数据库复杂化** → 物理场景的程序化扩展
- **诊断驱动的任务合成** → 针对 policy 弱项生成针对性失败 case 训练
- **自进化 arena** → VLA 的 evaluation suite 也应该是动态、可诊断的

---

*笔记创建时间: 2026-05-25*
