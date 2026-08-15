---
title: "冲突benchmark全景"
tags:
  - 评测轴
  - agent-memory
---

> 核查日期：2026-08-08。所有 arXiv 号均通过 `https://export.arxiv.org/api/query?id_list=...` 实查标题；所有 GitHub / HuggingFace 链接均 curl 实测 HTTP 状态码。
> 标记：【已验证】= 亲手拉到元数据/数据文件；【推断】= 论文摘要或 README 明说但未逐条验数据；【未找到】= 核不上。

---

## 0. 结论先行

1. **没有任何现成 benchmark 直接满足「agent 记忆库内部、集合级（>2 条）互相矛盾」的评测需求。**最接近的只有两个，都要改造：
   - **MemoryAgentBench 的 FactConsolidation 子集（Conflict Resolution 类别）**——这是**唯一**把「一堆事实条目扔进 agent 记忆、其中若干条彼此打架」直接做成任务的。但**实测其矛盾是严格成对的**（6k 版本：455 条事实 / 307 个 subject-relation key，multiplicity 分布 {1:159, 2:148}，即最多两条对立），大规模版本里出现的 3+ 只是我 key 归并的噪声。**没有集合级干扰。**
   - **RAMDocs（RAG with Conflicting Evidence）**——实测每题 1~12 篇文档、**每题 correct/misinfo/noise 混合**，distinct answer 数分布 {1:71, 2:168, 3:170, 4:82, 5:8}，即**约 52% 的题超过成对矛盾**，还叠加 noise 干扰。**这是目前唯一天然具备「多条共同误导 + 集合级冲突」结构的公开集**，但它是 RAG 文档级，不是 agent 记忆条目级 —— 可直接改造成 memory item。
2. **知识冲突（knowledge conflict）确实是成熟子领域**，有 survey（2403.08319）三分类：context-memory / inter-context / intra-memory。但**绝大多数 benchmark 的冲突粒度是「1 个正确 vs 1 个反事实」的成对替换**（ConflictBank、Chameleon、MAGIC、WhoQA 部分例外）。
3. **长期对话记忆 benchmark 里，只有 LongMemEval 有明确的「知识更新」类型（78/500 题），且实测每题恰好 2 个证据 session（旧值 → 新值），严格成对。** LoCoMo 已确认无矛盾设计（你之前的实测结论成立）。PersonaMem 是「偏好演变」而非逻辑矛盾。MemBench / PerLTQA 未见矛盾类型。

---

## 1. 直接面向 agent memory 的矛盾

### 1.1 MemoryAgentBench —— FactConsolidation / Conflict Resolution 【已验证，最相关】
- arXiv **2507.05257**，标题实查 = *Evaluating Memory in LLM Agents via Incremental Multi-Turn Interactions*；README 自述 ICLR 2026。
- 代码 https://github.com/HUST-AI-HYZ/MemoryAgentBench （200，418★）；数据 https://huggingface.co/datasets/ai-hyz/MemoryAgentBench （API 200，非 gated，10.8k downloads）。
- 四大能力之一即 **Conflict Resolution (CR)**，对应新造数据集 **FactConsolidation**。
- **矛盾构造：合成注入。**上下文是一串编号事实句（"The chairperson of Fatah is Mahmoud Abbas." / "... is Moshe Kahlon."），同一 (subject, relation) 给出两个不同 object，模型须以**后出现/最新**者为准（Wikidata 模板 + 反事实替换，风格同 MQuAKE）。
- **粒度：严格成对。**实测 `Conflict_Resolution-00000-of-00001.parquet`：8 行 × 每行 100 题 = **800 题**，分 `factconsolidation_sh`（单跳）/ `mh`（多跳）× 上下文长度 6k/32k/64k/262k。6k 档 455 条事实中，307 个 key 里 148 个恰有 2 条对立、159 个无冲突。**没有 3 条以上共同误导的集合级干扰。**
- 指标：Accuracy（JSON 字段 `substring_exact_match`）。

### 1.2 MemBench 【已验证存在，但无矛盾类型】
- arXiv **2506.21605** = *MemBench: Towards More Comprehensive Evaluation on the Memory of LLM-based Agents*。
- 代码 https://github.com/import-myself/Membench （200），数据托管在百度网盘 / Google Drive（非 HF）。
- 维度是 factual/reflective × participation/observation，另有 **NoiseData** 噪声扩展 —— 噪声≠矛盾。**未找到矛盾/更新子集。**

### 1.3 PerLTQA 【已验证存在，无矛盾类型】
- arXiv **2402.16288**，8,593 题 / 30 个角色，semantic+episodic memory。代码 https://github.com/Elvin-Yiming-Du/PerLTQA （200）。README/摘要 grep 无 conflict/contradict/update。**未找到矛盾设计。**

### 1.4 LoCoMo 【已验证：无矛盾设计】
- arXiv **2402.17753** = *Evaluating Very Long-Term Conversational Memory of LLM Agents*。与你的实测一致：category 5 "adversarial" 实为「对话中未提及之事」的弃权/幻觉测试。

---

## 2. 长期对话记忆 benchmark 里的「更新」子集

### 2.1 LongMemEval 【已验证，含 knowledge-update】
- arXiv **2410.10813**，ICLR 2025。代码 https://github.com/xiaowu0162/LongMemEval （200，984★），另有 https://github.com/xiaowu0162/LongMemEval-V2 （200）。数据 https://huggingface.co/datasets/xiaowu0162/longmemeval （200）。
- 实测下载 `longmemeval_oracle`：500 题，`question_type` 分布 = temporal-reasoning 133 / multi-session 133 / **knowledge-update 78** / single-session-user 70 / single-session-assistant 56 / single-session-preference 30（另 30 题为 `_abs` 弃权题）。
- **矛盾构造：合成生成的用户-助手对话。**如"5K 成绩 27:12" → 后续 session 更新为 "25:50"，答案取最新。
- **粒度：严格成对。**实测全部 78 道 knowledge-update 题 `len(answer_session_ids)==2` 且 `len(haystack_sessions)==2`（oracle 版），即**只有旧值/新值两条证据，无集合级干扰**。
- 指标：GPT-4o as judge 的 QA accuracy + 检索 Recall@k / NDCG@k。

### 2.2 PersonaMem 【已验证，但是「偏好演变」不是「事实矛盾」】
- arXiv **2504.14225** = *Know Me, Respond to Me: Benchmarking LLMs for Dynamic User Profiling and Personalized Responses at Scale*。
- 代码 https://github.com/bowen-upenn/PersonaMem （200）。HF：`bowen-upenn/PersonaMem`（200）、`PersonaMem-v1/v2` 存在；`bowen-chen/PersonaMem` 是我瞎猜的 id → 404/401，**不要用**。v2 另有论文 arXiv 2512.06688（README 声称，我**未**独立核标题，标【推断】）。
- 7 类 in-situ query，含「追踪用户偏好随时间变化」。**是偏好覆盖（新偏好取代旧偏好），非逻辑对立**；无明确"conflict"标注字段。

---

## 3. 知识冲突（knowledge conflict）主线

Survey：**2403.08319** *Knowledge Conflicts for LLMs: A Survey*（context-memory / inter-context / intra-memory 三分类）——建议作为综述起点。【已验证】

| 名称 | arXiv | 代码/数据 | 冲突怎么造 | 粒度 | 规模 | 指标 |
|---|---|---|---|---|---|---|
| **ConflictBank** | 2408.12076（NeurIPS 2024 D&B）| github.com/zhaochen0110/conflictbank (200,72★)；HF `Warrieryes/CB_qa`(200)、`Warrieryes/CB_claim_evidence`(200) | 合成：Wikidata+SLING 抽事实，LLM 生成对立 claim-evidence，分 **misinformation / temporal / semantic** 三种成因 | 主要**成对**（correct vs 一种 conflict evidence）；可组合多种冲突类型 | **7,453,853** claim-evidence 对 + **553,117** QA 对（README 实读） | QA accuracy / 冲突下的 memorization vs 受影响率 |
| **WikiContradict** | 2406.13805 | ❌ `github.com/IBM/wikicontradict` 404；HF `ibm/Wikipedia_contradict_benchmark` **200【已验证】**（`ibm-research/wikicontradict`、`ibm/WikiContradict` 均 401，别用） | **真实发生**：Wikipedia 同源段落之间天然矛盾，人工标注 | **成对**（两段互相矛盾的 passage） | 253 高质量人工标注实例 | 人工 + LLM-judge 评"是否指出两种答案" |
| **WhoQA** | 2410.15737（EMNLP 2024 Findings）| github.com/VinAIResearch/WhoQA (200)，含 `WhoQA.json` + `eval/eval.py` | **真实/半自动**：同名不同实体的共同属性 → 一问多答 | **⭐ 超过成对**：README 明写"questions with **up to 8 distinctive answers**" | 5K 问题 / 13 种 Wikidata 属性 / 150K Wikipedia 实体 | 是否透明汇报所有冲突答案（覆盖率类指标） |
| **Adaptive Chameleon or Stubborn Sloth**（framework，非命名数据集）| 2305.13300（ICLR'24 Spotlight）| github.com/OSU-NLP-Group/LLM-Knowledge-Conflict (200) | 合成：先诱出模型 parametric memory，再构造 counter-memory | **成对**（memory vs counter-memory），但可放多条 evidence 做**证据数量/一致性消融** | 见论文 | 模型采纳外部证据的比例 |
| **Resolving Knowledge Conflicts (KNOWLEDGE CONFLICT framework)** | 2310.00935 | github.com/yikee/Knowledge_Conflict (200) | 合成模拟 contextual conflict | 成对为主，任务含"定位冲突片段"、"给出多个不同答案" | 见论文 | 冲突识别率 / 定位精度 / 多答案覆盖 |
| **MAGIC** | 2507.21544（EMNLP 2025 Findings）| github.com/HYU-NLP/MAGIC (200)，数据在 repo `dataset/{single,multi}-hop/{1..4}-*_conflict.json` | **合成（KG 驱动）**：从知识图子图生成两段近似但相冲突的 context（`context1` vs `context2`，`perturb_triplet` 记录扰动） | **成对上下文**，但文件名前缀 1~4 = **每对 context 中被扰动 triplet 的条数**（多点冲突，接近集合级） | 8 个 json，总计约 2.4 MB；单跳/多跳各 4 档 | 冲突**检测**与**定位** 准确率 |
| **RAMDocs** | 2504.13079 | github.com/HanNight/RAMDocs (200)，`RAMDocs_test.jsonl` 可直接 raw 下载 | 合成 + 真实混合：基于 AmbigDocs，注入 **ambiguity + misinformation + noise** | **⭐⭐ 集合级**：实测 500 题，每题 1–12 篇文档（众数 5），文档类型计数 correct 1918 / noise 541 / misinfo 307；**每题 distinct answer 数 {1:71, 2:168, 3:170, 4:82, 5:8}**，gold_answers 数 {1:100, 2:200, 3:200} | 500 题 | 是否覆盖全部 gold answer + 是否被 misinfo 带偏（多答案 F1 类） |
| **DynamicQA** | 2407.17023（EMNLP 2024 Findings）| github.com/copenlu/dynamicqa (200)；HF `copenlu/dynamicqa` (200) | **真实**：intra-memory conflict —— temporal（随时间变）+ disputable（随视角变） | 单事实的**多值并存**（非人为对立） | Static 2500 / Temporal 2495 / **Disputable 694** | Semantic Entropy、Coherent Persuasion Score |
| **Rich Knowledge Sources...（Recalibrating）** | 2210.13701 | 【未找到】仓库未核（未实查，故不给链接） | 合成：parametric 答案 vs 多篇 passage 各持不同答案 | **多篇 passage 分歧**（早期的集合级探索） | 见论文 | calibration / accuracy |

---

## 4. 知识编辑（天然含新旧事实冲突）

| 名称 | arXiv | 代码 | 冲突构造 | 粒度 |
|---|---|---|---|---|
| **CounterFact / ROME / zsRE** | 2202.05262 = *Locating and Editing Factual Associations in GPT* | github.com/kmeng01/rome (200) | 合成反事实：把 (s,r,o) 的 o 换成 o* | **成对**（原事实 vs 编辑事实） |
| **MQuAKE** | 2305.14795 | github.com/princeton-nlp/MQuAKE (200) | 合成：编辑后用**多跳**问题查连锁效应 | 成对编辑，但**多跳链**上多个事实同时被牵动 → 有限的集合效应 |
| **RippleEdits** | 2307.12976 = *Evaluating the Ripple Effects of Knowledge Editing in Language Models* | github.com/edenbiran/RippleEdits (200)，数据在 `data/benchmark/`，三子集 RECENT / RANDOM / POPULAR | 合成编辑 + Wikidata 推导涟漪事实 | **一编辑 → 多条派生事实需同步更新**（最接近"集合级一致性"的编辑集） |

> 注：`zjunlp/KnowledgeConflict` 404，我最初猜的这个 URL **不存在**，勿用。

---

## 5. 时效性 / 过时信息（旧事实被新事实推翻）

| 名称 | arXiv（实查标题） | 代码 | 说明 |
|---|---|---|---|
| **TimeQA** | 2108.06314 *A Dataset for Answering Time-Sensitive Questions* | github.com/wenhuchen/Time-Sensitive-QA (200) | Wikidata 时序事实 + 众包校验；同一问题不同时间不同答案 |
| **SituatedQA** | 2109.06157 | github.com/mikejqzhang/SituatedQA (200)（`lil-lab/situatedqa`、`mikewangwzhl/...` 均 404） | 答案随**时间/地点**语境变化；NQ-Open 中约 16.5% 是语境依赖题 |
| **StreamingQA** | 2205.11388 | github.com/google-deepmind/streamingqa (200) | 14 年时间戳新闻，季度增量评测知识更新与灾难性遗忘 |
| **FreshQA** | 2310.03214 *FreshLLMs* | github.com/freshllms/freshqa (200，399★)（`google/FreshQA` 404） | 含 fast-changing 题 + **false-premise 题**（需要反驳前提） |
| **DyKnow** | 2404.08700 | 【未找到】仓库未核 | 对 Wikidata 动态验证 24 个 LLM 的过时程度，并评 4 种编辑方法 |
| **TDBench** | 2508.02045 | 【未找到】仓库未核 | 用时序数据库/时序 SQL 自动构造 TSQA；新指标 time accuracy |
| **温馨提示** | — | — | **「TemporalQA」「TAQA」核不上**：arXiv 检索无对应标准 benchmark，`CogComp/TemporalQA` 404。**这两个名字很可能是记混了**，建议改用上表。 |

---

## 6. 对话一致性 / 自相矛盾

| 名称 | arXiv | 代码 | 说明 |
|---|---|---|---|
| **DECODE** | 2012.13391 *I like fish, especially dolphins: Addressing Contradictions in Dialogue Modeling*（ACL 2021） | 数据随 ParlAI 发布（github.com/facebookresearch/ParlAI，200）；独立 repo【未找到】 | 人-人 + 人-bot 矛盾对话；**成对**（某句 vs 历史某句），二分类 + 定位矛盾句 |
| **CDConv** | 2210.08511（EMNLP 2022） | github.com/thu-coai/CDConv (200) | **中文** 12K 多轮，三类：句内矛盾 / 角色混淆 / **历史矛盾**；自动生成 + 人工筛 |

---

## 8. 核不上 / 需要澄清的线索

- **「ConflictQA」**：作为独立 benchmark 名字**核不上**。arXiv 全文检索只返回引用它的论文（如 2412.17031）。它实际是 **2305.13300 (Adaptive Chameleon)** 论文内部构造的数据集别名，代码在 OSU-NLP-Group/LLM-Knowledge-Conflict。标【推断】。
- **「KC 系列」「Farm」**：按此关键词检索**未找到**对应 benchmark，请提供更多线索。
- **「TemporalQA」「TAQA」**：见 §5，**核不上**。
- **「LifeSide」「MemGym」**：arXiv/GitHub 检索**未找到**可对应的记忆 benchmark。若确有出处请给链接。
- **WikiContradict 的官方 GitHub**：`IBM/wikicontradict` 404，只确认了 HF `ibm/Wikipedia_contradict_benchmark`（200）。
- 若干仓库我只核了论文没核代码，已在表中标【未找到】（Rich Knowledge Sources / DyKnow / TDBench / DECODE 独立 repo）。

---

## 相关笔记

- [[评测与数据集全景]]
- [[LongMemEval harness 审计]]
