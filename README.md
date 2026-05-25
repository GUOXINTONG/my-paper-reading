# my-paper-reading

> 个人 AI 论文阅读记录。以 World Model / VLA / Embodied AI / Video Understanding / Multimodal 为主线，
> 通过自动化 skills 抓取 arXiv + HuggingFace 论文，做带态度的锐评，并对重点论文写精读笔记。

## 仓库定位

这不是一个"什么都收"的论文聚合器，而是一个**有筛选、有判断、有锐评**的个人阅读日志：

- 每周/每天扫描 1000+ 篇候选，最终保留 20~30 篇做点评
- 重点论文出独立笔记（带方法详解 / Baselines 对比 / 锐评）
- 主题搜索（如 "video × agent"）按需触发，做窄方向深扫
- 笔记互相 wikilink，最终在 Obsidian 里形成知识图谱

## 文件结构

```
my-paper-reading/
├── YYYY-MM-DD-论文推荐.md          # 周度/日度推荐文件（带锐评、分流表、Top 5）
├── YYYY-MM-DD-<topic>专题搜索.md   # 专题深扫（例：video-agent）
├── YYYY-MM-DD-<topic>-raw.json     # 专题原始检索数据
├── <PaperName>.md                  # 单篇精读笔记（如 AgentWorld.md）
└── README.md
```

文件命名规则：

| 类型 | 命名 | 示例 |
|------|------|------|
| 周度推荐 | `YYYY-MM-DD-论文推荐.md` | `2026-05-25-论文推荐.md` |
| 专题搜索 | `YYYY-MM-DD-<topic>专题搜索.md` | `2026-05-25-video-agent专题搜索.md` |
| 专题原始数据 | `YYYY-MM-DD-<topic>-raw.json` | `2026-05-25-video-agent-raw.json` |
| 单篇笔记 | `<MethodName>.md` | `AgentWorld.md` |

## 推荐文件的结构

每份 `YYYY-MM-DD-论文推荐.md` 包含：

1. **本周锐评**：主旋律、值得关注的 2~3 篇、灌水重灾区
2. **分流表**：🔥 必读 / 👀 值得看 / 💤 可跳过
3. **按方向分章**：World Model · VLA · Video Generation · Video LLM · 3D · 架构 · 应用
4. **每篇格式**：核心方法 / 对比 Baselines / 借鉴意义 / 锐评 / "想精读？运行 `读一下 XXX`"
5. **本周趋势判断**：5 条总结

## 单篇笔记的结构

每份 `<MethodName>.md` 是一篇标准化精读笔记：

- 元信息（机构 / 日期 / 项目主页 / 对比 Baselines / 链接）
- 一句话总结
- 核心贡献
- 问题背景（要解决的问题 / 现有方法局限 / 本文动机）
- 方法详解（整体架构 / 核心模块拆解 / 公式与符号）
- 实验结论
- 优缺点 + 锐评
- 与相关工作的联系（Obsidian wikilink）

## 自动化工作流

仓库依赖以下 [Cursor / Claude skills](https://docs.cursor.com/) 形成流水线，
**无需手动维护**，命令触发即可：

| Skill | 触发词 | 作用 |
|-------|--------|------|
| `daily-papers` | "今日论文推荐"、"过去 3 天论文推荐"、"过去一周论文" | 一句话总入口，串联抓取 → 点评 → 笔记 |
| `daily-papers-fetch` | "论文抓取" | arXiv + HuggingFace 抓取 + 打分筛选 + 富化 |
| `daily-papers-review` | "论文点评" | 生成带态度的推荐锐评（输出 `YYYY-MM-DD-论文推荐.md`） |
| `daily-papers-notes` | "批量笔记" | 给推荐文件里的重点论文生成精读笔记 |
| `paper-reader` | "读一下 XXX"、"读一下这篇" | 单篇精读（支持 Zotero / PDF / arXiv URL） |
| `generate-mocs` | "更新索引"、"刷新 MOC" | 重建 Obsidian 论文 / 概念目录页 |

典型一周流程：

```
"过去一周论文推荐"
  → 抓取 + 筛选 + 锐评 + 自动生成重点论文笔记
  → 输出 YYYY-MM-DD-论文推荐.md + N 篇 <MethodName>.md

"读一下 GEM-4D"
  → 单篇精读，生成 GEM-4D.md

"video × agent 专题搜索过去 30 天"
  → 输出 YYYY-MM-DD-video-agent专题搜索.md
```

## 阅读建议

如果你是第一次浏览这个仓库：

1. **先看最新的 `*论文推荐.md`** —— 了解当前关注的方向和锐评风格
2. **跟着分流表 🔥 必读** —— 这是经过筛选的本周精华
3. **想深入某篇就读对应的 `<MethodName>.md`** —— 例如 `AgentWorld.md`
4. **专题搜索 `*专题搜索.md`** —— 用于窄方向深扫，做调研时按需查看

## 关注的方向

- **World Model**：video WM、3D/4D WM、physical-grounded WM、WAM (VLA + WM)
- **VLA / Embodied AI**：dual-system VLA、Agent RL、environment scaling
- **Video Understanding**：long video、video agent、streaming、memory + retrieval
- **Video Generation**：DiT / Linear DiT、long-form、controllable
- **Multimodal / 架构**：unified MLLM、linear attention、tokenization

锐评原则：**不吹不黑，方法可信看 Baseline 比较和真机实验，定调论文标 👀 而非 🔥**。

## License

[MIT](LICENSE)
