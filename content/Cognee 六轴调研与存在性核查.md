---
title: "Cognee 六轴调研与存在性核查"
tags:
  - 框架轴
  - agent-memory
---

调研对象：本地 clone `memres/cognee`，commit `38eece5bbb0cb9f5706fed908abd16dba0f5505e` (2026-08-01)
方法：只读源码 + arXiv API 核验。所有结论均给文件路径与行号；找不到的写"未找到"。

---

## 第一部分：Cognee 六轴

### 轴 1 — 存储结构

**四类存储并存（多库混合，非单库分层）**，目录 `cognee/infrastructure/databases/`：
- `relational/`（SQLAlchemy，SQLite/Postgres）：元数据主表 `Data`，见 `cognee/modules/data/models/Data.py`
- `graph/`（Kuzu / Neo4j / Postgres / Turso / ladybug 适配器）：知识图谱节点与边
- `vector/`（LanceDB / PGVector / Turso）：embedding 检索
- `hybrid/`（Neptune Analytics、postgres）：图+向量合一后端
- `cache/`、`provenance/`、`unified/`：辅助层

**数据模型层级**：`Document → DocumentChunk → Entity/EntityType`（`cognee/modules/chunking/models/DocumentChunk.py`、`cognee/modules/engine/models/Entity.py`）。
基类 `DataPoint`（`cognee/infrastructure/engine/models/DataPoint.py:75` 带 `importance_weight: float|None = 0.5`）。

**记忆间的依赖/影响关系**：以**知识图谱的边**表达（`relationship_name` 字段，见 `access_tracking.py:52`）。图边是语义关系（entity↔chunk↔document），
**未找到**表达"记忆 A 的价值依赖于记忆 B"这类价值传播/影响力依赖边的机制——没有 PageRank 式价值扩散、没有 causal/影响权重边。
图内存投影 `CogneeGraph.project_graph_from_db`（`cognee/modules/graph/cognee_graph/CogneeGraph.py`）只投影 id/type/relationship_name。

**特殊子结构**：`cognee/modules/truth_subspace/`（真值子空间，见轴 6）；`cognee/modules/engine/models/SkillRun.py`（技能执行记录，带评分）。

### 轴 2 — 生命周期

- 摄入：`cognee/api/v1/add/add.py`（`add()`，`importance_weight` 在 `add.py:44` 作为入参，默认 0.5）
- 结构化：`cognify`（`cognee/api/v1/cognify/`）→ 抽取实体/关系建图
- 二次加工：`memify`（`cognee/modules/memify/memify.py` + `skill_improvement.py`）
- 一站式：`cognee/api/v1/remember/remember.py`（把 kwargs 路由到 add/cognify，`remember.py:75` 传 `importance_weight`）
- 检索：`cognee/modules/search/methods/get_retriever_output.py`
- 删除：`cognee/api/v1/forget/forget.py`（统一替代旧 prune/delete）、`cognee/api/v1/prune/`
- 迁移：`alembic/versions/*`（`e1ec1dcb50b6_add_last_accessed_to_data.py`、`f1a2b3c4d5e6_add_importance_weight_to_data.py`）

### 轴 3 — 晋升触发（短期→长期 / 冷→热的层级提升）

**未找到**。Cognee 没有 working/short-term/long-term 分层记忆，也没有任何"晋升/降级"状态机。
最接近的是 `memify` 的技能改进流程（`cognee/modules/memify/skill_improvement.py`）：低分/报错的 `SkillRun` 被挑出（`skill_improvement.py:250-274`，`score_threshold`、`is_low_score = run.success_score < score_threshold`）→ LLM 生成 `SkillImprovementProposal` → 人审后落库。这是**技能文本的改写提案**，不是记忆条目的层级晋升。

### 轴 4 — 冲突消解

**基本没有语义级冲突消解**，只有主键去重：
- `cognee/modules/graph/methods/upsert_nodes.py:63-74`：`insert(Node).on_conflict_do_nothing(index_elements=["id"])`，注释明写 "on_conflict_do_nothing intentionally preserves the FIRST run's ..." —— 即**先写先赢，后来者直接丢弃**。
- `upsert_edges.py` 同理。
**未找到**：矛盾检测（contradiction detection）、事实版本化/失效（invalidate old fact）、LLM 仲裁式冲突消解。代码里 `contradict` 关键词只出现在 eval framework 与异常类，非记忆冲突逻辑。

### 轴 5 — 遗忘 / 淘汰

**有，且是全场少见的"基于访问时间的自动淘汰"**：
- `cognee/tasks/cleanup/cleanup_unused_data.py`
  - `cleanup_unused_data(minutes_threshold, dry_run=True, user_id=None)`（L27）
  - 门控：`ENABLE_LAST_ACCESSED != "true"` 直接 skip（L48-56）
  - 安全检查：若无任何 `last_accessed` 非空记录则 skip（L62-79）
  - 淘汰判据 L119-123：`where(or_(Data.last_accessed < cutoff_date, Data.last_accessed.is_(None)))` —— **纯 LRU 时间阈值，未使用 importance_weight，也没有衰减函数**
  - 粒度：**整文档删除**（document-level），非条目级
  - 默认 `dry_run=True`，需显式调用，不是后台自动 GC
- 手动删除：`cognee/api/v1/forget/forget.py`（按 data/dataset/everything/memory_only）
**未找到**：指数衰减（decay）、TTL 字段、遗忘曲线、基于价值的淘汰排序。

### 轴 6 — 记忆价值评估 ★

Cognee 有**三套互不相通的分数**，都很浅：

**(a) `importance_weight`（唯一进入检索排序的记忆价值分）**
- 定义：`cognee/infrastructure/engine/models/DataPoint.py:75` `importance_weight: float | None = 0.5`
- SQL 列：`cognee/modules/data/models/Data.py:40` `importance_weight = Column(Float, nullable=True)`
- 迁移：`alembic/versions/f1a2b3c4d5e6_add_importance_weight_to_data.py:35`
- **分数来源：纯人工/调用方传入**。`cognee/api/v1/add/add.py:44` `importance_weight: Optional[float] = 0.5`，经 `add.py:219` 传给 `ingest_data`。默认 0.5。
- 传播：Document → chunk 单向继承（`cognee/modules/chunking/TextChunker.py:36,58,83`、`JsonListChunker.py:51`、`cognee/modules/graph/utils/expand_with_nodes_and_edges.py:66,79,163,229`），各向量库 payload 也带该字段（`LanceDBAdapter.py:76,1320`、`PGVectorAdapter.py:62,407`、`TursoVectorAdapter.py:37,255`、`NeptuneAnalyticsAdapter.py:46,483,562`、`hybrid/postgres/adapter.py:354`）。
- **消费点 1（混合检索加权）**：`cognee/modules/retrieval/hybrid/ranking.py:36-37,55-59`
  ```
  final_score = rrf_score
  if use_importance_weight: final_score *= _importance_factor(chunk)
  _importance_factor: 0.75 + 0.5 * clamp(importance_weight, 0, 1)   # 值域 [0.75, 1.25]
  ```
- **消费点 2（图距离加权）**：`cognee/modules/graph/cognee_graph/CogneeGraph.py:515-534`，`distance = (2 - importance_weight) * distance`
- **是否随使用更新回写：否。** 全仓库 grep `importance_weight` 只有读取/透传，**没有任何一处 UPDATE / 赋新值**。写入时是什么值，永远是什么值。

**(b) `success_score`（技能执行成功度，不影响记忆检索）**
- `cognee/modules/engine/models/SkillRun.py:7` `UNSCORED_SKILL_RUN_SCORE = 0.5`；`:41` `success_score: float = UNSCORED_SKILL_RUN_SCORE  # 0.0 to 1.0`
- 来源：异常时硬编码 0.0（`cognee/modules/retrieval/agentic_retriever.py:246`），正常执行时**不评分**、落到中性 0.5（`agentic_retriever.py:263-264,316` `resolved_score = UNSCORED_SKILL_RUN_SCORE if success_score is None else success_score`）。外部反馈可经 `cognee/memory/entries.py:96-112`（校验 [0,1]）、`cognee/modules/tools/skill_runs.py:101-113` 写入。
- 消费：仅 `skill_improvement.py:271` 挑低分 run 生成改进提案。**不参与任何记忆排序，也不回写到 importance_weight。**

**(c) `truth_alignment` / 真值子空间（唯一"语义质量"信号）**
- `cognee/modules/truth_subspace/{align,build,centroids,models,constants}.py`
- `build.py:108+ build_truth_subspace()`：取 session_learnings 节点做 anchor → 嵌入 → 聚成 centroids（带 `truth_epoch` 版本号，`build.py:142,193-206`）→ 对**全部 DocumentChunk** 嵌入并投影得 `coords`，写回节点属性 `{"truth_alignment": coords, "truth_epoch": current_epoch}`（`build.py:251-259`）
- `align.py`：`truth_score` = 查询权重加权的方向对齐度，缺省中性 0.5；`truth_factor` = `0.75 + 0.5*truth_score`，值域 [0.75,1.25]，缺省中性 1.0
- 消费：`hybrid/ranking.py:40-43`，且**仅当 `truth_state["truth_epoch"] == current_truth_epoch` 时才生效**（旧 epoch 分数自动失效）
- 这是 Cognee 里**唯一会被重算并回写到图节点的分数**，但它由离线批处理 `build_truth_subspace` 触发，**与"这条记忆被用过几次/用得好不好"无关**。

**反事实基础（counterfactual）：未找到。** 无任何 counterfactual / ablation / "若不召回这条会怎样" 的评估代码。

---

## ★ access_tracking 深挖结论：只是"最后访问时间"，**没有回写到价值/排序**

文件：`cognee/modules/retrieval/utils/access_tracking.py`（147 行）

**追踪了什么**
- 入口 `update_node_access_timestamps(items)`（L19），由 `cognee/modules/search/methods/get_retriever_output.py:88` 在每次检索后调用（`get_retriever_output.py:10` import）。
- 默认**关闭**：`os.getenv("ENABLE_LAST_ACCESSED", "false") != "true"` 直接 return（L20-21）。
- 从检索结果里抽节点 id（L74-114，支持 Edge 列表与 {chunks, entities} 字典）。
- 把命中的 `DocumentChunk` 沿图边上溯到 `TextDocument`/`Document`（L45-71 `_find_origin_documents_via_projection`，需全图投影，代价不低）。
- 写库（L136-147）：
  ```python
  update(Data).where(Data.id.in_(doc_ids)).values(last_accessed=timestamp_dt)
  ```

**关键判定：写了什么、没写什么**
- ✅ 写的**只有一个字段**：`Data.last_accessed`（`cognee/modules/data/models/Data.py:39` `DateTime(timezone=True), nullable=True`）——**一个时间戳，会被每次访问覆盖**。
- ❌ **没有访问计数器**（no `access_count` / `hit_count` 字段，全仓库不存在）。
- ❌ **没有回写到 `importance_weight`**：该函数从头到尾不触碰 importance_weight，全仓也无任何 importance_weight 的 UPDATE 语句。
- ❌ **不影响检索排序**：排序发生在 `hybrid/ranking.py`，其输入只有 RRF rank、`importance_weight`、`truth_alignment`；`last_accessed` **完全不在排序公式里**。
- ✅ 唯一下游消费者：`cognee/tasks/cleanup/cleanup_unused_data.py:122`，用作**删除阈值**（LRU 淘汰）。
- 另一消费者：`cognee/api/v1/activity/routers/get_activity_router.py:178-179`，`max(Data.last_accessed)` 做**活动展示（遥测/看板）**。

**结论一句话**：Cognee 的 access tracking 是「**默认关闭的、文档粒度的、只存最后一次时间戳的 LRU 标记**」。
它**既不是访问频次统计，也不回写记忆价值分，也不参与排序**——只服务于「多久没被用过 → 整篇删掉」的垃圾回收，外加一个活动看板。
所以严格讲：Cognee 有"使用信号 → 生存/淘汰"的闭环，但**没有"使用信号 → 价值分 → 排序"的闭环**。记忆价值分（importance_weight）在 Cognee 里是**写死的静态先验，终身不变**。

---

## 第二部分：存在性核查（arXiv API 实测）

核查方式：`https://export.arxiv.org/api/query`（注意 http 明文在本机被拦，须用 https）。

### [48] O-Mem — ✅ **真实存在**
- **arXiv: 2511.13593**（当前版本 v3）— https://arxiv.org/abs/2511.13593
- 标题：*O-Mem: Omni Memory System for Personalized, Long Horizon, Self-Evolving Agents*（与既有文献记录一致）
- 作者：Piaohong Wang, Motong Tian, Jiaxian Li, Yuan Liang, Yuqing Wang, Qianben Chen, Tiannan Wang, Zhicong Lu, Jiawei Ma, Yuchen Eleanor Jiang, Wangchunshu Zhou
- 首次提交：2025-11-17；最新更新：2025-12-10（v3）
- 方法：基于 **active user profiling** 的记忆框架——从用户主动交互中动态抽取并更新「用户特征 + 事件记录」，支持 persona 属性与话题上下文的**分层检索**（hierarchical retrieval）。作者批评现有系统"检索前先做语义分组"会漏掉语义无关但关键的用户信息并引入噪声。
- 结果：LoCoMo 51.67%（超 LangMem ~3%）；PERSONAMEM 62.99%（超 A-Mem 3.5%）；token 与响应时延效率也更优。
- 与「记忆价值评估」的关系：论文摘要层面**只提到 profile 的动态更新（update user characteristics）**，属于"内容自演化"，**未见显式的记忆打分/价值排序机制**（摘要未提；如需精确结论建议读全文，本次仅据 arXiv metadata）。
- 开源代码：arXiv metadata 未附 code link，GitHub API 关键词搜索**未检出**明确的官方仓库（**未找到**，不排除存在但未被检索到）。

### [49] MemFly — ✅ **真实存在**（正文写成 "Memly" 确是笔误）
- **arXiv: 2602.07885**（当前版本 v2）— https://arxiv.org/abs/2602.07885
- 标题：*MemFly: On-the-Fly Memory Optimization via Information Bottleneck*（与参考文献**完全一致**）
- 作者：Zhenyuan Zhang, Xianzhang Jia, Zhiqin Yang, Zhenbo Song, Wei Xue, Sirui Han, Yike Guo（HKUST 系）
- 首次提交：2026-02-08；最新更新：2026-08-05（v2）
- **Comment: Accepted by ICLR 2026 MemAgents Workshop**
- 方法（据摘要）：以**信息瓶颈原理**为基础的 on-the-fly 记忆演化框架——**最小化压缩熵、最大化相关性熵**，用**gradient-free optimizer**求解，构造**分层记忆结构（stratified memory structure）**；配套**混合检索**融合**语义 / 符号 / 拓扑三条路径**，并加入迭代精化处理多跳查询。指标：memory coherence、response fidelity、accuracy 全面超 SOTA。
- 与「NKT（Note→Keyword→Topic）三路径并行召回」传闻的对照：摘要里明确的是「semantic / symbolic / topological 三通路混合检索」+「stratified memory」。**"Note→Keyword→Topic" 这一命名在 arXiv 摘要中未出现**——很可能是同一设计的另一种表述（stratified 三层 ≈ Note/Keyword/Topic），但**本次核查无法从摘要证实该命名，不做断言**，需读全文确认。
- 与「记忆价值评估」的关系：其价值判据是**信息论的**——一条记忆保留与否/压到多细，取决于「压缩熵 vs 相关性熵」的权衡，而非使用频次或人工先验。这与 Cognee 的静态 `importance_weight` 形成鲜明对比。
- 开源代码：GitHub API 关键词搜索**未检出**官方仓库（**未找到**）。

### 核查副产品
- 检索 `all:"Memly"` → **0 结果**，佐证正文 "Memly" 为笔误。
- 相关邻近工作（信息瓶颈 × agent memory）：*MIND: Lightweight and Effective Memory Injection Defense for LLM Agents via Intent-Aware Information Bottleneck* (arXiv:2607.28103)。

---

## 一句话总结

- **Cognee 轴 1**：多库并存（relational + graph + vector + hybrid + cache），Document→Chunk→Entity 三级数据模型，图边只表达语义关系，**无价值依赖/影响传播边**。
- **Cognee 轴 6**：三套分数各自为政——`importance_weight`（唯一进排序，纯人工静态，永不更新）、`success_score`（技能执行，不进排序）、`truth_alignment`（唯一被重算回写，但由离线批处理驱动、与使用无关）；**无反事实基础**。
- **access_tracking**：只写 `Data.last_accessed` 一个时间戳，默认关闭、文档粒度、无计数器；**不回写价值分、不参与排序**，仅供 LRU 删除与活动看板。
- **O-Mem / MemFly**：均真实存在，arXiv 2511.13593 / 2602.07885。

---

## 相关笔记

- [[记忆框架全景]]
- [[mem0-graphiti-zep 六轴调研]]
