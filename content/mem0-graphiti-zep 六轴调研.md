---
title: "mem0-graphiti-zep 六轴调研"
tags:
  - 框架轴
  - agent-memory
---

调研方式：mem0、Graphiti 读本地 clone 源码（`memres/mem0`、`memres/graphiti`）；Zep 无本地 clone，读 arXiv 2501.13956 摘要 + 其开源内核 Graphiti 源码（Zep 论文明确表述 Graphiti 是 Zep 的 core component，故 Zep 的存储/时序机制 = Graphiti 机制，Zep 独有的是托管服务层与 benchmark）。

**结论一句话**：三个框架没有任何一个具备「记忆价值分数」这一持久化字段，更没有随使用回写、也没有任何反事实/移除影响评估。已确认的基线缺口是**全领域普遍现象**，不是个例。

---

## 3 × 6 汇总表

| 轴 | mem0 | Graphiti | Zep |
|---|---|---|---|
| **1 存储结构** | 向量库（主 collection + 独立 entity collection）+ SQLite 审计表；无图库内核。记忆间关系仅有 `linked_memory_ids`（LLM 判定的"相关"），**无依赖/影响关系** | 图数据库（Neo4j/FalkorDB/Kuzu…），4 类节点 Episodic/Entity/Community/Saga + 双时间边 EntityEdge；节点/边都带 embedding，图 = 存储也 = 索引。边表达语义关系与**时序失效**关系，**无价值依赖关系** | 同 Graphiti（论文 §core component）+ 托管 API 层；论文描述为 episode / semantic entity / community 三层时序知识图 |
| **2 层级与生命周期** | 两层：raw messages(SQLite) → fact(向量库)。写入=`add()` LLM 抽取；更新=UPDATE 事件；淘汰只有显式 `delete()` | 三/四层：Episodic → Entity+EntityEdge → Community → Saga。写入=`add_episode()`；社区增量 `update_community()`；`remove_episode()` 显式删除 | 同 Graphiti；论文强调 episode 不可变、事实层可失效 |
| **3 晋升触发** | LLM 判断（DEFAULT_UPDATE_MEMORY_PROMPT / ADDITIVE_EXTRACTION_PROMPT），无定时、无阈值 | LLM 抽取 + 社区检测（label propagation, `build_communities()` 手动/批量触发）；Saga 增量摘要有 `last_summarized_at` 水位线 | 同 Graphiti |
| **4 冲突消解** | LLM 三态裁决 ADD/UPDATE/DELETE/NONE + 0.95 余弦近重复短路 | **双时间模型**：`valid_at/invalid_at`（事件时间）+ `created_at/expired_at`（系统时间），`resolve_edge_contradictions()` 按时间序判定谁作废谁 | 论文核心卖点即此时序失效机制（LongMemEval 时序推理） |
| **5 遗忘/淘汰** | OSS 版**无自动遗忘**；`decay` 参数在 OSS SDK 显式报错（仅云端有） | **无遗忘/淘汰**：边只 expire 不删除，历史全保留（设计即"不遗忘"） | 同上；论文未描述容量或价值驱动的淘汰 |
| **6 记忆价值评估** | **只有查询期 relevance，无持久价值分**。`score_and_rank()` 现算现扔，不回写 | **只有查询期 reranker**（rrf / node_distance / episode_mentions / mmr / cross_encoder），`episode_mentions` 是提及计数但**查询时现算，不落库不回写** | 同 Graphiti；论文只报端到端 accuracy，无 per-memory 价值度量 |

---

## 轴 1 · 存储结构（证据）

### mem0
- 主存 = 向量库，工厂 `VectorStoreFactory.create`（`mem0/mem0/memory/main.py`），支持 20+ backend（`mem0/mem0/configs/vector_stores/*`）。
- **第二个向量 collection 存实体**：`Memory.entity_store` 属性，`mem0/memory/main.py:554-575`，collection 名由 `_entity_collection_name()`（`main.py:417`）派生。
- 关系型层只是**审计日志**，不是记忆存储：`mem0/memory/storage.py:102` `_create_history_table()`、`:128` `_create_messages_table()`；字段为 memory_id/old/new/event。
- 记忆之间的显式连接只有一个字段：`payload["linked_memory_ids"]`（`main.py:624`、`main.py:1149`），由 LLM 在 `ADDITIVE_EXTRACTION_PROMPT`（`mem0/configs/prompts.py:464-516`）按"同主题/同实体/偏好变更/叙事延续"判定 —— 语义相关，**不是依赖或影响**。prompt 第 701 行还明确禁止"仅主题模糊相关"的连接。
- 检索期实体连边只用来加分：`main.py:1782-1793`，`boost = similarity * ENTITY_BOOST_WEIGHT * memory_count_weight`，`ENTITY_BOOST_WEIGHT = 0.5`（`mem0/utils/scoring.py:57`）。这是一个**共享实体数量的惩罚项**（`1/(1+0.001*(n-1)^2)`），不是记忆间影响建模。
- 层数判定：2 层（raw message → fact），实体索引算旁路。

### Graphiti（= Zep 内核）
- 节点类型（`graphiti_core/nodes.py`）：`EpisodicNode:318`（`content` 原始文本、`valid_at:322`、`entity_edges:325`）、`EntityNode:499`（`name_embedding:500`、`summary:501`、`attributes:502`）、`CommunityNode:687`（`summary`）、`SagaNode:867`（`summary`、`first/last_episode_uuid`、`last_summarized_at`）。
- 边类型（`graphiti_core/edges.py`）：`EpisodicEdge:143`（MENTIONS）、`EntityEdge:263`（含 `episodes:267` 溯源列表、`expired_at:271`、`valid_at:274`、`invalid_at:277`、`reference_time:280`）、`CommunityEdge:575`、`HasEpisodeEdge:689`、`NextEpisodeEdge:822`。
- 因此层级 = 4 层（Episode / Entity+Fact edge / Community / Saga），每层数据形态见上；embedding 与图同库（节点属性 `name_embedding`、边 `fact_embedding`），是**图 + 向量 + 全文（BM25）的混合**，检索方法枚举 `graphiti_core/search/search_config.py:32-50`：cosine_similarity / bm25 / breadth_first_search。
- **依赖/影响关系：未找到。** 边语义是「事实关系」（LLM 抽取的 subject-predicate-object）与「时序作废」（invalid_at/expired_at），`NextEpisodeEdge` 只是时间序链，`HasEpisodeEdge` 是归属。没有任何字段表达"记忆 A 的存在改变了记忆 B 的效用"。

### Zep
- 论文摘要（arXiv 2501.13956）：Zep 的 core component 是 Graphiti，"a temporally-aware knowledge graph engine that dynamically synthesizes both unstructured conversational data and structured business data while maintaining historical relationships"。故存储结构结论与 Graphiti 一致。
- Zep 独有的托管层实现（分片、多租户、缓存）**未开源、未找到**，不做推测。

---

## 轴 2 · 层级定义与生命周期

- mem0 写入：`Memory.add()` → LLM 抽 facts → 与检索到的相似记忆一起丢给 UPDATE prompt 裁决。时间戳写入 `main.py:1023-1025`（`created_at`/`updated_at` 初始相同）；更新时 `main.py:2049-2050` 保留 created_at、刷新 updated_at。历史全部落 `history` 表（`storage.py:150 add_history`）。
- mem0 淘汰：只有用户显式 `delete()` / `delete_all()`（`main.py:1873`、`:1929`），**没有自动生命周期**。
- Graphiti 写入：`Graphiti.add_episode()`（`graphiti.py:980`）；批量 `add_episode_bulk()`（`:1230`）。社区构建 `build_communities()`（`graphiti.py:1490` → `utils/maintenance/community_operations.py:216`），增量 `update_community()`（`community_operations.py:340`，在 `graphiti.py:1187` 被 add_episode 路径触发）。
- Graphiti 删除：`remove_episode()`（`graphiti.py:1765`）——按 episode 回滚，是**运维接口不是遗忘策略**。

---

## 轴 3 · 晋升触发

- mem0：纯 LLM 判断。`DEFAULT_UPDATE_MEMORY_PROMPT`（`configs/prompts.py:176`）定义四事件 ADD/UPDATE/DELETE/NONE（`:182-185`）；v3 路径 `ADDITIVE_EXTRACTION_PROMPT`（`prompts.py:464`）只 ADD 并产出 `linked_memory_ids`。**无定时、无频次阈值、无学习信号**。
- Graphiti：实体/边抽取全 LLM（`utils/maintenance/combined_extraction.py`、`edge_operations.py`）；边时间戳也是单独一次轻量 LLM 调用 `_extract_edge_timestamps()`（`edge_operations.py:581`，注释 "Extract valid_at / invalid_at timestamps for an edge via a lightweight LLM call"），且 `:587` 已有时间戳则跳过。
- Community 晋升是**图算法**（label propagation）而非 LLM 打分，触发是显式调用或 add_episode 时的增量更新。
- Saga 晋升有水位线：`SagaNode.last_summarized_at` / `last_summarized_episode_valid_at`（`nodes.py:871-876`）——这是三者中唯一接近"阈值触发"的机制，但阈值是**时间水位**不是价值。

---

## 轴 4 · 冲突消解（Graphiti 重点）

mem0：LLM 在 UPDATE prompt 里直接输出 `event: UPDATE/DELETE`（`prompts.py:246/290`），旧值进 `old_memory` 字段（`prompts.py:445`）。另有硬短路：余弦 ≥ 0.95 视为同一条记忆（`main.py:616`、`main.py:1142`：`existing[0].score >= 0.95`）。**没有时间维度**，靠 LLM 主观判断谁覆盖谁。

Graphiti / Zep：**双时间（bi-temporal）模型**，是三者里唯一严肃处理矛盾的。
- 字段：`EntityEdge.valid_at / invalid_at`（事件时间）、`created_at / expired_at`（系统时间），`edges.py:271-280`；持久化见 `edges.py:352-354`，读回 `edges.py:998-1000`。
- 主流程 `resolve_extracted_edge`（`edge_operations.py:~800-845`）：
  1. `edge_operations.py:822-823` —— `if resolved_edge.invalid_at and not resolved_edge.expired_at: resolved_edge.expired_at = now`（LLM 说它已失效 → 立刻系统时间上作废）。
  2. `:826-838` —— 新边若发现有 `valid_at` 更晚的候选边，**新边自己被作废**：`resolved_edge.invalid_at = candidate.valid_at; resolved_edge.expired_at = now`。即乱序摄入也能保证时间正确性。
  3. `:842` 调 `resolve_edge_contradictions()`。
- `resolve_edge_contradictions()`（`edge_operations.py:537-572`）逻辑：
  - 若两边的有效区间**不重叠**（`edge.invalid_at <= resolved.valid_at` 或 `resolved.invalid_at <= edge.valid_at`）→ `continue`，不算矛盾（历史并存）。
  - 若旧边 `valid_at < 新边 valid_at` → `edge.invalid_at = resolved_edge.valid_at`；`edge.expired_at = utc_now()`（`:569-570`）。
- 关键点：**作废 = 打标不删除**，`invalidated_edges` 仍然重新算 embedding 并落库（`edge_operations.py:530-532`）。历史可完整回放 —— 这对可回放的评测实验是重要参照：Graphiti 有"这条事实何时不再成立"的表达，但**没有"这条事实值多少"的表达**。

---

## 轴 5 · 遗忘 / 淘汰

- mem0 OSS：**明确不支持 decay**。`mem0/memory/notices.py:49` `DECAY_FEATURE_ERROR_MESSAGE = "The decay parameter is not supported by the OSS Memory SDK."`；`main.py:462-478` 传 `decay=True` 直接 `raise ValueError`。`notices.py:256 detect_decay_usage_from_delete()` 只是在你手动删够 `DECAY_USAGE_DELETE_THRESHOLD` 次后弹广告提示去用云端。→ 依据是**商业分层**，不是任何时间/容量/价值算法。
- Graphiti / Zep：**未找到任何遗忘、淘汰、容量上限、TTL 机制**。全仓 grep `decay|forget|salience|importance` 只命中 prompt 文案和注释（`prompts/extract_nodes.py:163`、`:563`，`utils/content_chunking.py:284/345/446`），无实现。边只 `expired_at` 打标；`remove_episode()` 是显式运维删除。设计哲学是"全量保留 + 时序过滤"。

---

## 轴 6 · 记忆价值评估（最关键）

### mem0
- **没有持久价值分数字段。** 记忆 payload 的持久 key 集合硬编码在 `main.py:1224` / `:1341` / `:1689`：`{"data","hash","created_at","updated_at","id","text_lemmatized","attributed_to"}` + promoted keys。**没有 score / importance / access_count / last_accessed**。
- 唯一的 score 是**查询期临时组合分**：`mem0/utils/scoring.py:60 score_and_rank()`，`:118-119`
  `raw_combined = semantic_score + bm25_score + entity_boost; combined = min(raw_combined/max_possible, 1.0)`，`max_possible` 依启用信号取 1.0/1.5/2.0/2.5（`:97-101`）。分数放进返回值 `MemoryItem.score`（`main.py:1704`），`explain=True` 时附 `score_details`（`main.py:1716-1717`）——**返回后即丢弃，不回写向量库**。
- 存库时甚至主动剔除 score：`main.py:1353` `.model_dump(exclude={"score"})`。
- 全仓 grep `access_count|last_accessed|usage_count|retrieval_count|reinforce` → **0 命中**。
- 全仓 grep `counterfactual|ablation|leave-one-out` → **0 命中**。
- 结论：分数来源 = 相似度 + BM25 + 实体加成（全部 query-time、无学习、无主观打分持久化）；**不随使用更新**；**无任何反事实基础**。

### Graphiti
- **没有持久价值分数字段。** `Node`/`Edge` 基类字段（`nodes.py:93-98`、`edges.py:49-54`）只有 uuid/name/group_id/labels/created_at；`EntityEdge` 额外字段全是时间与溯源（`edges.py:263-280`）。无 score/importance/weight/access_count。
- 排序全在查询期，reranker 枚举 `search/search_config.py:53-77`：`rrf` / `node_distance` / `episode_mentions` / `mmr` / `cross_encoder`；默认 `rrf`（`:82/90/98/106`）。
- 最接近"使用频次价值"的是 `episode_mentions_reranker`（`search/search_utils.py:1844`）：
  ```cypher
  MATCH (episode:Episodic)-[r:MENTIONS]->(n:Entity {uuid: node_uuid})
  RETURN count(*) AS score, n.uuid AS uuid
  ```
  —— 这是**写入侧被提及次数**（多少 episode 提到这个实体），不是**读取侧被检索/被使用次数**；而且是 query 时现场 count，**不落库、不回写**，检索行为对它零影响。
- `node_distance_reranker`（`search_utils.py:1782`）按到 center node 的图距离排序 —— 结构邻近，非价值。
- 全仓 grep `access_count|last_accessed|usage_count|importance|salience|decay|forget` → 仅命中 prompt 文案（`prompts/extract_nodes.py:563` 明确 **禁止** LLM 推断 importance）与注释，**无实现**。
- 全仓 grep `counterfactual|ablation|leave-one-out` → **0 命中**。
- 结论：无价值分、无回写、无反事实。而且 `extract_nodes.py:563` 那句 "NEVER infer preferences, habits, recurrence, frequency, causality, intent, **importance**, or category" 是**显式的设计拒斥**——Graphiti 有意不做价值判断。

### Zep
- 论文（arXiv 2501.13956）只报**端到端指标**：DMR 94.8% vs MemGPT 93.4%，LongMemEval 最高 +18.5% accuracy、-90% latency。**没有 per-memory 价值度量，没有记忆消融/移除实验**（未在摘要中出现；全文若有更细粒度分析，本次未取到 —— 标注为「未找到」而非否定）。
- 实现层价值机制 = Graphiti 的（见上）：无持久分、无回写。
- Zep 托管服务是否额外维护使用统计 → **未找到**（闭源）。

---

## 相关笔记

- [[记忆框架全景]]
- [[冲突benchmark全景]]
