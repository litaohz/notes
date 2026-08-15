---
title: "MemOS-EverOS-Letta 六轴对比"
tags:
  - 框架轴
  - agent-memory
---

调研方式：MemOS、EverOS 读本地 clone 源码；Letta 读官方 GitHub main 分支源码 + MemGPT 论文 (arXiv 2310.08560)。
路径：
- MemOS: `<本地路径>`
- EverOS: `<本地路径>`
- Letta: 远程 `github.com/letta-ai/letta` (main)

凡未在源码中找到的，明写「未找到」。

---

## 一、总表

| 轴 | MemOS | EverOS | Letta |
|---|---|---|---|
| 1 存储结构 | 图库为主(Neo4j/PolarDB) + 向量 + KV-Cache + LoRA，三态 MemCube | Markdown 为源 + LanceDB 索引 + SQLite 元数据，7 张表多层 | Postgres/SQLite 关系表 + pgvector/Turbopuffer，扁平三层 |
| 2 层级/生命周期 | WorkingMemory→LongTermMemory/UserMemory 等 10 型，status 四态 | memcell→episode/atomic_fact→cluster→skill/profile/foresight | core block / recall(message) / archival(passage) |
| 3 晋升触发 | 容量 FIFO + 异步 reorganizer + scheduler | 事件驱动 (Immediate) + Cron 反思 | LLM 自行调工具，无自动晋升 |
| 4 冲突消解 | LLM 判定 contradictory/redundant → 融合新节点 + MERGED_TO 边 | 反思阶段聚类合并 → `deprecated_by` 指针 | 无系统级冲突消解（LLM 覆写 block） |
| 5 遗忘/淘汰 | WorkingMemory FIFO(20)、LongTerm 上限 1500、status=archived | `deprecated_by` 软删除，检索时过滤；无 TTL/衰减 | 上下文 buffer evict→摘要；archival 不删 |
| 6 价值评估 | `confidence`(LLM,0-100) + `usage[]` 列表；usage 写回**代码已被注释掉** | `quality_score`/`confidence`/`maturity_score`，全部 LLM 一次性生成，**不随使用更新** | **未找到任何价值/重要性分数字段** |

---

## 二、轴 1：存储结构（重点）

### MemOS
- 三态 MemCube：`src/memos/mem_cube/general.py:28,34,39` — `_text_mem` / `_act_mem` / `_para_mem`。
  - plain-text：`src/memos/memories/textual/`（general/naive/tree/preference）
  - activation：`src/memos/memories/activation/item.py:32` `KVCacheItem.memory: DynamicCache`（transformers KV cache）
  - parametric：`src/memos/memories/parametric/item.py:8` `ParametricMemoryItem`，实现在 `lora.py`
- 主力是 **tree_text_memory**：图数据库后端（Neo4j / PolarDB / Nebula），节点=记忆，边=类型化关系。
- **有超越相似度的关系边**（该领域少见）：`src/memos/memories/textual/tree_text_memory/organize/relation_reason_detector.py:232` `valid = {"CAUSE","CONDITION","RELATE","CONFLICT","NONE"}`；
  `reorganizer.py:342` `PARENT`、:389 `INFERS`、:395 `FOLLOWS`、:411 `AGGREGATE_TO`；`handler.py:186` `MERGED_TO`。
- 判断：**CAUSE / CONDITION / INFERS 是最接近「记忆间依赖/影响关系」的表达**，但它们是 LLM 对两条陈述的语义关系判定（因果/条件），**不是「A 存在会改变 B 对任务的贡献」这种效用依赖**；边上无权重字段。

### EverOS
- 混合三层持久化：
  - **Markdown 文件为唯一事实源**（`src/everos/infra/persistence/markdown/`，每行都有 `md_path` + `content_sha256`），
  - **LanceDB 作检索索引**（`src/everos/infra/persistence/lancedb/tables/`，7 张表），
  - **SQLite 存元数据/聚类/游标**（`src/everos/infra/persistence/sqlite/repos/`：cluster.py、memcell.py、knowledge.py、reflection_report.py、unprocessed_buffer.py、md_change_state.py）。
- 7 张 LanceDB 表（均继承 `BaseLanceTable`，都有 `vector: Vector(_DIM)`）：
  | 表 | 文件 | 关键字段 |
  |---|---|---|
  | atomic_fact | `tables/atomic_fact.py:20` | :46 `fact`、:43 `parent_id`、:62 `deprecated_by` |
  | episode | `tables/episode.py:22` | :57 `episode`、:55-56 `subject`/`summary`、:77 `deprecated_by`、:83 `subject_vector`（双向量） |
  | agent_case | `tables/agent_case.py:21` | :52 `quality_score`、:55 `task_intent`、:62 `approach`、:73 `key_insight` |
  | agent_skill | `tables/agent_skill.py:20` | :57 `confidence`、:60 `maturity_score`、:64 `source_case_ids`、:67 `cluster_id` |
  | knowledge_topic | `tables/knowledge_topic.py:13` | :25 `topic_path`、:26 `depth`、:27 `parent_node_id`（树形） |
  | user_profile | `tables/user_profile.py:24` | :46 `explicit_info_json`、:52 `implicit_traits_json` |
  | foresight | `tables/foresight.py:21` | :57 `foresight`、:64 `evidence`、:42-48 `start_time`/`end_time`/`duration_days` |
- 层级用 `parent_type` 枚举串起：`tables/_parent_type.py:24-29` → `MEMCELL` / `EPISODE` / `CLUSTER`。即 memcell（原始会话单元，SQLite）→ episode / atomic_fact / agent_case（LanceDB 抽取层）→ cluster（SQLite 聚类）→ agent_skill / user_profile（蒸馏层）。
- **记忆间关系表达：只有三种指针** —— `parent_id`（层级归属）、`source_case_ids`（skill 的来源 case 血缘，`agent_skill.py:64`）、`deprecated_by`（被谁取代）。**没有图、没有依赖/影响边、没有边权重**。血缘 `source_case_ids` 是最接近「影响」的，但只表达「由谁蒸馏而来」，不表达贡献度。

### Letta
- 纯关系数据库 + 向量列，**没有图**：
  - core memory = `Block`，`letta/schemas/block.py:13` `BaseBlock`，字段 `value`/`limit`(默认 `CORE_MEMORY_BLOCK_CHAR_LIMIT`)/`label`/`read_only`/`description`/`metadata`。常驻上下文窗口。
  - recall memory = `Message` 表（`letta/schemas/message.py`），完整对话历史。
  - archival memory = `Passage`（`letta/schemas/passage.py:35`）挂在 `Archive`（`letta/schemas/archive.py:24`）下，`archive.vector_db_provider` 选 pgvector 或 Turbopuffer。Passage 字段：`text`、`embedding`、`tags`、`metadata`、`archive_id`、`source_id`、`file_id`、`is_deleted`。
- 与 MemGPT 论文（arXiv 2310.08560）一致：main context / external context 两级 + 分页调用，论文本身也没有记忆间关系结构。
- **记忆间关系：完全没有**。Passage 之间无任何边或指针；唯一的组织维度是 `tags` 和 `archive_id`。

**轴 1 结论**：三者中只有 MemOS 有类型化的记忆间关系边（含 CAUSE/CONDITION/INFERS），EverOS 只有层级/血缘/废弃三种指针，Letta 完全扁平。**没有任何一个把「记忆之间的效用依赖/相互影响」建模成带权结构。**

---

## 三、轴 2：层级定义与生命周期

**MemOS**：`TreeNodeTextualMemoryMetadata.memory_type`（`src/memos/memories/textual/item.py:176-187`）枚举 10 种：WorkingMemory / LongTermMemory / UserMemory / OuterMemory / ToolSchemaMemory / ToolTrajectoryMemory / RawFileMemory / SkillMemory / PreferenceMemory / Context。
生命周期状态 `item.py:108` `status: "activated" | "resolving" | "archived" | "deleted"`；`version`(:120)、`history[]`(:124 归档旧版本)、`evolve_to[]`(:116 记录演化到哪些新节点)。
容量：`organize/manager.py:75-80` `{"WorkingMemory":20, "LongTermMemory":1500, "RawFileMemory":1500, "UserMemory":480}`。

**EverOS**：层级即上表的 parent_type 链条。生命周期靠 `deprecated_by`（null=活跃）+ Markdown 文件 sha256 diff（cascade handler 跳过未变内容，`memory/cascade/handlers/agent_skill.py:99-108`）。无 status 字段、无版本号。

**Letta**：三层是**位置**而非生命周期状态——core（永驻上下文）/ recall（DB 中全部消息）/ archival（向量库）。唯一的生命周期是上下文 buffer 的 evict + 摘要（`letta/services/summarizer/summarizer.py:136` `_partial_evict_buffer_summarization`、:244 `_static_buffer_summarization`）。Passage 只有 `is_deleted` 软删除标志。

---

## 四、轴 3：晋升触发

**MemOS**：
- 写入即建 WorkingMemory 节点，超 20 条 FIFO 淘汰（`organize/manager.py:252-261` `_cleanup_working_memory` → `graph_store.remove_oldest_memory(keep_latest=...)`）。
- 结构重组走异步队列：`manager.py:243` `self.reorganizer.add_message(QueueMessage(op="add", ...))`，由 `GraphStructureReorganizer.optimize_structure`（`reorganizer.py:211`）**周期性**执行：聚类→LLM 摘要→建 PARENT 树，参数 `local_tree_threshold=10, min_cluster_size=4, min_group_size=20, max_duration_sec=600`。
- 另有 `mem_scheduler/` 一套独立调度（`memory_manage_modules/enhancement_pipeline.py:116` 等）在 LongTermMemory 上做增强。
- 晋升判定混合：**几何聚类 + LLM 摘要**。

**EverOS**：纯事件驱动 + Cron。
- `@offline_strategy(trigger=Immediate(on=[AgentCaseExtracted]))`（`strategies/trigger_skill_clustering.py:43-47`）：case 落库立刻聚类。
- 阈值门：`trigger_skill_clustering.py:38` `_SKIP_QUALITY_THRESHOLD = 0.2`，`quality_score < 0.2` 的 case 直接跳过（连 embedding 都不调）。
- 聚类走 LLM：`from everalgo.clustering import cluster_by_llm`（:24），注释明说选 LLM 精修路径而非纯几何。
- 蒸馏 skill：`extract_agent_skill.py` 由 `SkillClusterUpdated` 触发，取 top-K 相关 skill（`MAX_SKILLS_IN_PROMPT = 10`，:73）+ 支撑 case（`MAX_SUPPORTING_CASES = 9`，:80，按 `(quality_score desc, timestamp desc)` 排序）喂给 `AgentSkillExtractor`。
- Cron：`strategies/reflect_episodes.py` 默认 `0 2 * * 1`（每周一凌晨 2 点）跑 ReflectionOrchestrator。

**Letta**：**未找到自动晋升机制**。记忆写入完全由 LLM 主动调用工具（`core_memory_append/replace`、`archival_memory_insert`）完成，MemGPT 论文称之为 self-editing memory。近版有 sleep-time agent 做后台整理，但仍是 LLM 自主决策，无阈值/定时晋升规则。

---

## 五、轴 4：冲突消解

**MemOS**（唯一做得完整的）：
- 检测：`organize/handler.py:30-74` `detect()` — 先 embedding 召回 top_k 候选（`EMBEDDING_THRESHOLD`），再逐对喂 `MEMORY_RELATION_DETECTOR_PROMPT` 让 LLM 输出 `contradictory` / `redundant` / `independent`。
- 消解：`handler.py:76+` `resolve()` LLM 融合成新节点，然后 `handler.py:186-187` 给两个旧节点各加一条 `MERGED_TO` 边指向 merged 节点。
- 阈值：`manager.py:61-62` `threshold=0.80`, `merged_threshold=0.92`。
- 另有 `mem_feedback/feedback.py`（:434 `semantics_feedback`、:788 `filter_fault_update`）处理用户显式反馈引发的更正。

**EverOS**：反思阶段合并。`memory/reflection/orchestrator.py:867,903` 把被合并的 episode / atomic_fact 批量 `update({"deprecated_by": merged_entry_id})`；检索侧 `memory/search/filters.py:111-114` 自动加 `deprecated_by IS NULL`。另 `cascade/handlers/_daily_log_base.py:236-263` `_mark_deprecated` 在每次 md 同步/重建时刷新该列。**是「合并取代」而非「矛盾裁决」——没有 LLM 判定矛盾的环节。**

**Letta**：**未找到系统级冲突消解**。core memory 的 `core_memory_replace` 由 LLM 自己覆写，archival passage 只增不改。

---

## 六、轴 5：遗忘/淘汰依据

**MemOS**：
- WorkingMemory 严格 FIFO 保最新 20（`manager.py:256-260`、:284-286 `remove_oldest_memory(keep_latest=...)`）。
- LongTermMemory / RawFileMemory 上限 1500、UserMemory 480（`manager.py:76-79`）。
- status 可置 `archived`/`deleted`（`item.py:108`）。
- **淘汰依据是时间 + 容量，不是价值分数。** `dream/maintenance.py` 描述了一套基于 `last_hit_at` TTL / `usefulness_score` / `invalidated_by_feedback` 的淘汰设计，但该文件通篇是设计文档字符串（"Implementation hints"、"Entry point suggestion"），`grep` 显示这四个字段除 `dream/types.py:80-83` 的定义外**在全仓无任何读写点** → **未实现**。

**EverOS**：只有 `deprecated_by` 软删除（见轴 4）。全仓 grep `forget|decay|evict|ttl|prune` 无命中任何记忆淘汰逻辑（命中项全是连接池/队列/配置热重载）。**没有 TTL、没有衰减、没有容量上限、没有低价值清理。**

**Letta**：淘汰只发生在**上下文窗口**，不发生在存储。`summarizer.py:279-311`：buffer 超 `message_buffer_limit` 时 evict 到 `retain_count`，被 evict 的消息做递归摘要；`partial_evict_summarizer_percentage = 0.30`（:49）。被 evict 的消息仍完整留在 recall memory 表里。archival passage 无淘汰。**依据是位置 + 数量，与内容价值无关。**

---

## 七、轴 6：记忆价值评估（最重点）

### MemOS
1. **`confidence`**：`src/memos/memories/textual/item.py:135-138`，`float | None`，描述 "A numeric score (float between 0 and 100) indicating how certain you are about the accuracy or reliability of the memory."
   - 来源：**LLM 抽取时一次性生成**（提示词让模型自评）。
   - 是否更新：抽取/融合时可被新值覆盖，但**没有任何基于使用效果的回写**。
   - 语义是「可信度/正确性」，**不是「对任务有多大用」**。
2. **`usage: list[str]`**：`item.py:202-205`，"Usage history of this node"。
   - 本应是唯一的「使用回写」通道，但 **`searcher.py:1343-1367` 的 `_update_usage_history` 整个函数体被包在三引号里注释掉了**——函数只剩 docstring，实际是 no-op。配套的 `_update_usage_history_worker`（:1369）因此永不被调用。
   - 全仓其余 30+ 处 `usage` 引用都是 `usage=[]` 硬编码空列表（`mem_reader/`、`retrieve/`、`api/handlers/` 等）。
   - 唯一消费点 `mem_os/utils/format_utils.py:535-536`：`frequency = len(usage)`，仅用于**前端可视化的节点热度**，不参与检索排序、不参与淘汰。
   - **结论：MemOS 有使用回写的骨架，但当前 main 分支上是关闭的死代码。**
3. **`dream/` 模块**：`dream/types.py:80-83` 定义了 `last_hit_at` / `hit_count` / `usefulness_score` / `invalidated_by_feedback`——**这是三个框架里唯一出现「usefulness」概念的地方**，但如轴 5 所述，仅有定义无实现；`dream/maintenance.py:23-26` 自己写着 "Suggested approach: integrate with the retrieval layer to update usefulness_score…"，即作者明确知道该做而未做。
4. **反事实**：**未找到**。无任何「移除该记忆后表现变化」的评估、消融或归因逻辑。
5. 检索期的 `score`（`item.py:296`）是相似度/rerank 分，属查询时排序，非记忆自身价值属性。

### EverOS
1. **`quality_score: float`**：`tables/agent_case.py:52`。来源：`memory/cascade/handlers/agent_case.py:16` 注释 "float 0.0–1.0 (**LLM-emitted**)"，从 md inline 字段解析（:90-91 `require_float(s.inline.get("quality_score"))`）。
   - 用途：① `trigger_skill_clustering.py:38` 低于 0.2 直接不聚类；② `extract_agent_skill.py:337` 支撑 case 排序 `key=lambda c:(c.quality_score or 0.0, c.timestamp)`。
   - **写回路径只有一条：case 抽取时 LLM 给分**（`strategies/extract_agent_case.py:159` 打进 inline）。此后无任何更新。
2. **`confidence: float`**：`tables/agent_skill.py:57-58`，docstring 明写 "0.0–1.0; **LLM-emitted** confidence in the skill."
3. **`maturity_score: float`**：`tables/agent_skill.py:60`；`docs/api.md:803` / `:1009` "Maturity assessed at clustering time, in [0,1]"。
   - **核实上次线索：确认成立。** 全仓 `maturity_score` 只有 9 个非测试命中点（见下），**没有任何一处是基于使用/效果的回写**：
     - 定义：`lancedb/tables/agent_skill.py:60`、`markdown/mds/agent_skill.py:52`
     - 解析入库：`cascade/handlers/agent_skill.py:69,100,108,170` — `float(fm.get("maturity_score", 0.0))`，从 SKILL.md frontmatter 读
     - 传出：`memory/get/dto.py:187`、`get/manager.py:182`、`search/dto.py:220`、`search/shaper.py:176,196`（纯读，透传给 API）
     - 写：`strategies/extract_agent_skill.py:386,407` 和 `_persist_skill()`（:391-418）—— 值来自 `AlgoAgentSkill.maturity_score`，即 **`everalgo.agent_memory.AgentSkillExtractor` 这个外部 LLM 抽取器的输出**（everalgo 包未随仓库提供，本机未安装，无法读其内部；但 EverOS 侧确定是「拿来即写」）。
   - **判定：maturity_score 是每次 skill 重蒸馏时由 LLM 重新给的一个数，不是随使用累积的成熟度。** 它随 cluster 长大而可能变化（因为 supporting_cases 变多、LLM 重新评估），但**变化源是「有多少 case 支撑」，不是「用了多少次 / 用了有没有效」**。
   - 且注意：它也不参与任何决策——EverOS 里没有一处用 `maturity_score` 做过滤、排序或淘汰，纯粹是暴露给 API 的展示字段。相比之下 `quality_score` 至少参与了 0.2 门限。
4. **使用回写**：全仓 grep `usage_count|access_count|last_used|hit_count` → **零命中**（仅命中无关的 `visited` 局部变量和一句注释）。**EverOS 完全不记录记忆被用过几次。**
5. **反事实**：**未找到**。

### Letta
1. **未找到任何记忆价值/重要性/置信度分数字段。**
   - `Block`（`letta/schemas/block.py`）：无 score / importance / confidence / usage 字段。
   - `Passage`（`letta/schemas/passage.py`）：字段仅 `text` / `embedding` / `tags` / `metadata` / `archive_id` / `source_id` / `file_id` / `is_deleted` / `created_at` / `organization_id` —— 无任何分数。
   - `Archive`（`letta/schemas/archive.py`）：无。
2. 唯一的「价值判断」是 LLM 自身的隐式决策：模型决定把什么写进 core memory、什么 append 到 archival。MemGPT 论文（arXiv 2310.08560）也是这个设计——把记忆管理当作 OS 分页问题，**分页依据是上下文容量压力，不是内容价值**。
3. **使用回写**：无（无 hit_count / last_accessed 字段）。
4. **反事实**：**未找到**。

---

## 八、对论文基线缺口的确认

针对「共享上下文里多知识单元的价值评估与相互影响」，三框架的实证结论：

1. **「移除这条记忆，任务表现降多少」——三个框架全部无法回答。**
   零个反事实/消融/归因机制。所有分数都是**生成时的自评**（LLM 说这条 case 质量 0.8），而非**事后的效用测量**。

2. **价值分数不随使用更新——全部成立，但成因各异**：
   - Letta：连分数字段都没有，无从更新。
   - EverOS：有三个分数（quality/confidence/maturity），全是 LLM 一次性生成；连「被用过几次」都不记录。maturity_score 甚至不参与任何决策。
   - MemOS：**最接近实现但已关闭** —— `usage[]` 回写代码被注释掉（`searcher.py:1343`），`usefulness_score`/`hit_count` 只有 dataclass 定义无实现（`dream/types.py:80-83`）。这反而是强证据：**社区已识别出该需求，但做不出来（因为没有可靠的效用信号，正是反事实缺失所致）。**

3. **「记忆之间的相互影响」建模——最接近的是 MemOS 的 CAUSE/CONDITION/INFERS 边**（`relation_reason_detector.py:232`、`reorganizer.py:389`），但：
   - 是 LLM 对两条**文本陈述**的语义关系判断，不是对**任务贡献**的依赖关系；
   - 边**无权重**，无法表达「影响强度」；
   - 不参与检索排序或淘汰决策，只用于图结构组织。
   EverOS 的 `source_case_ids` 是血缘（谁蒸馏出谁），Letta 无任何边。
   **结论：没有任何系统把「记忆 A 的存在如何改变记忆 B 的边际价值」建模出来。**

4. **淘汰依据全是时间/容量，无一基于价值**：MemOS FIFO+容量上限，EverOS 只有合并取代（连淘汰都没有），Letta 按上下文窗口位置 evict。

综上：**基线缺口成立且普遍**，且 MemOS 的注释掉的死代码与 dream 模块的「设计而未实现」构成了「领域已知需求 × 无解法」的直接证据，可作为论文 motivation 的强引用。

---

## 九、未找到项汇总（明确声明）

- MemOS：`usefulness_score` / `hit_count` / `last_hit_at` 的**实现**（仅有定义）；反事实评估；基于价值的淘汰。
- EverOS：使用计数字段；遗忘/TTL/衰减机制；冲突矛盾裁决（只有合并）；反事实评估；`everalgo` 包源码（未随仓库提供，本机未安装，故 `maturity_score` 的具体计算公式无法读取——但可确定 EverOS 侧不做二次更新）。
- Letta：任何记忆价值分数字段；自动晋升机制；冲突消解；记忆间关系结构；反事实评估。

---

## 相关笔记

- [[记忆框架全景]]
- [[mem0-graphiti-zep 六轴调研]]
