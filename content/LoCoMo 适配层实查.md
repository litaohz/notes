---
title: "LoCoMo 适配层实查"
tags:
  - 评测轴
  - agent-memory
---

调查日期：2026-08-08。方法：先在本地 clone（`<本地路径>`）里 `grep -ri locomo`，再补 clone 两个外部仓库（`mem0ai/memory-benchmarks` → 本地 `memres/mb`，`WujiangXu/A-mem` → 本地 `memres/amem_repo`）逐文件读代码。**所有结论都追到了具体文件:行号。**

---

## 结论先行

1. **现成可用、最完整的是 mem0 官方外置仓 `mem0ai/memory-benchmarks`**（`benchmarks/locomo/run.py`，981 行，ingest→search→answer→judge 四阶段全含，带 checkpoint 断点续跑）。但它**只支持 mem0 后端**（OSS server / cloud 两种），换宿主要自己写 client。
2. **跨框架统一 harness 唯一现成的是 MemOS 的 `evaluation/scripts/locomo/`**，一个 `--lib` 参数支持 7 个后端：`mem0 / mem0_graph / memos-api / memos-api-online / memobase / memu / supermemory`。**要做跨框架对比，MemOS 是最佳宿主。**
3. **记忆粒度（最关键的一条）分三档**，见下方第 3 节表：mem0-benchmarks 是 **单轮 turn → LLM 抽取原子事实**（`CHUNK_SIZE = 1`），MemOS 是 **整 session 批量喂入（batch_size=2）**，A-MEM 是 **单轮 turn → 一条 MemoryNote（带 LLM 抽的 keywords/context/tags）**，EverOS 是 **单条 message 带 dia_id + 毫秒时间戳，保留 session 边界**。
4. **graphiti 和 cognee 都没有 LoCoMo 适配层**（cognee 有 eval_framework 但 benchmark 枚举里没有 LoCoMo）。
5. 打分基本统一是 **LLM-as-judge 二分类 CORRECT/WRONG**；MemOS 和 A-MEM 额外算 F1/BLEU/ROUGE/BERTScore 作为辅助指标。
6. 成本：只有 EverOS README 给了量级 —— **全 10 conv ≈ 1000 万 token、2–4 小时**。

---

## 1. mem0 —— 【已验证】代码在外置仓，本地 submodule 是空的

- 【已验证】`memres/mem0/evaluation/` 是**空目录**（`ls mem0/evaluation` 无输出）。
- 【已验证】`mem0/AGENTS.md:36`：`| evaluation/ | Submodule → mem0ai/memory-benchmarks — benchmarking (LOCOMO, LongMemEval, BEAM) lives in that repo |`
- 【已验证】`mem0/AGENTS.md:230` 给了填充方法：`git submodule update --init evaluation`。

实际代码（我已 clone 到 `memres/mb/`）：

| 文件 | 行数 | 作用 |
|---|---|---|
| `mb/benchmarks/locomo/run.py` | 981 | 主 pipeline：下载数据集 / 解析 session / ingest / search / answer / judge / 指标 |
| `mb/benchmarks/locomo/prompts.py` | 322 | 答案生成 prompt + judge prompt（含 evidence 变体） |
| `mb/benchmarks/common/mem0_client.py` | 477 | mem0 OSS / cloud 双后端 HTTP client |
| `mb/benchmarks/common/metrics.py` | 205 | 聚合指标、Kendall tau-b |
| `mb/results/platform/locomo_results.json` | — | 官方跑出的结果快照（可对拍） |

四阶段都齐全，【已验证】：
- 数据加载：`mb/benchmarks/locomo/run.py:96 download_dataset()` / `:117 load_dataset()` / `:144 get_sorted_sessions()`
- 记忆写入：`:238 ingest_conversation()` → `:340 await mem0.add(messages, user_id, timestamp=session_epoch)`
- 检索：`:417 search_results = await mem0.search(question, user_id, top_k=top_k, ...)`
- 打分：`:389 process_question()` / `:501 apply_locomo_judge_to_saved_result()` / `:622 compute_locomo_metrics()`

**可独立运行**（【已验证】`mem0/AGENTS.md:238` 与 `mem0/docs/core-concepts/memory-evaluation.mdx:200-210` 给出同一命令，且 `mb/benchmarks/locomo/run.py:681-716` 的 argparse 参数与之一致）：

```bash
git clone https://github.com/mem0ai/memory-benchmarks && cd memory-benchmarks
pip install -r requirements.txt
# cloud 后端
python -m benchmarks.locomo.run --project-name my-test --backend cloud --mem0-api-key $MEM0_API_KEY
# OSS 后端（自建 mem0 server，docker-compose.yml 已提供）
python -m benchmarks.locomo.run --project-name oss-test --backend oss --mem0-host http://localhost:8888
```

关键参数（`run.py:681-716`）：`--conversations 0..9`、`--top-k 200`、`--top-k-cutoffs 10,20,50,200`、`--categories 1,2,3,4`、`--answerer-model gpt-5`、`--judge-model gpt-5`、`--resume`、`--predict-only`、`--with-evidence`、`--max-questions`（快速冒烟）。

**换宿主的改造点**：唯一耦合面是 `mb/benchmarks/common/mem0_client.py` 的 `add()/search()/delete_user()` 三个方法（`:117/:241/:350`）。OSS 路径就是 `POST {host}/memories` 和 search（`mem0_client.py:158`），接口非常薄，**替换成别的框架的 client 大概只需改这一个文件**。这是 mem0-benchmarks 作为宿主的最大优势。

---

## 2. 其他框架逐一核查

### MemOS —— 【已验证】有，而且是唯一的跨框架 harness
目录 `memres/MemOS/evaluation/scripts/locomo/`，7 个脚本：
- `locomo_ingestion.py`（写入）、`locomo_search.py`（检索）、`locomo_responses.py`（生成答案）、`locomo_eval.py`（judge + NLP 指标）、`locomo_metric.py`（聚合出表）、`locomo_rag.py`、`locomo_openai.py`（OpenAI Memory 对照组）
- 入口脚本 `MemOS/evaluation/scripts/run_locomo_eval.sh`（【已验证】完整 5 步串联，`LIB="memos-api" VERSION=default WORKERS=10 TOPK=20`）

```bash
cd MemOS/evaluation
bash scripts/run_locomo_eval.sh
# 或分步，切换后端：
python scripts/locomo/locomo_ingestion.py --lib mem0 --version v1 --workers 10
python scripts/locomo/locomo_search.py     --lib mem0 --version v1 --top_k 20 --workers 10
python scripts/locomo/locomo_responses.py  --lib mem0 --version v1
python scripts/locomo/locomo_eval.py       --lib mem0 --version v1 --workers 10 --num_runs 3
python scripts/locomo/locomo_metric.py     --lib mem0 --version v1
```
【已验证】后端枚举在 `MemOS/evaluation/scripts/locomo/locomo_ingestion.py:216-224`：`mem0, mem0_graph, memos-api, memos-api-online, memobase, memu, supermemory`。
【已验证】判分跑 3 次取平均并算一致率（`run_locomo_eval.sh` 的 `--num_runs 3`；`locomo_eval.py:238 process_group_responses(..., num_runs)`）。
文档：`MemOS/docs/{cn,en}/open_source/evaluation/overview.md`、`.../openai_memory_locomo_eval_guide.md`。

### EverOS —— 【已验证】有，自成一套单文件 harness
- `memres/EverOS/benchmarks/run.py` + `config.py/config.toml` + `README.md`（325+ 行文档）
- 【已验证】stage 化：`EverOS/benchmarks/run.py:9` 注释 `python benchmarks/run.py --run-name baseline-v1 --stages search answer judge`；README:152 给完整命令 `python benchmarks/run.py --run-name locomo-agentic [--everos-root <path>]`，`--conv 0` 单会话，`--stages judge` 只重判。
- 数据加载 `run.py:560 load_conversation()`，**显式保留 session_N 边界以支持 per-session flush**（`run.py:541` 注释原文："preserve LoCoMo session_N structure for per-session flushing"）。
- 还有一个现成 fixture：`EverOS/tests/fixtures/long_conversation_locomo_caroline_melanie.json`（真实 LoCoMo 会话，可做 e2e 冒烟）。
- 数据集下载命令在 `EverOS/benchmarks/README.md:117`：`curl -o data/locomo10.json https://raw.githubusercontent.com/snap-research/locomo/main/data/locomo10.json`。

### A-MEM —— 【已验证】有（GitHub `WujiangXu/A-mem`，已 clone 到 `memres/amem_repo`）
- **数据集直接内置**：`amem_repo/data/locomo10.json`（不用另外下）
- `amem_repo/load_dataset.py`：`:59 parse_session()`、`:79 parse_conversation()`、`:98 load_locomo_dataset()`、`:214 get_dataset_statistics()`，有完整 dataclass（`Session/Conversation/LoCoMoSample`）
- `amem_repo/test_advanced.py:217 evaluate_dataset(...)` 是主入口；`run_all_experiments.sh` / `run_k_sweep.sh` 是批量脚本（含 retrieve_k 扫参）
- 后端支持 openai / ollama / sglang / litellm（`memory_layer.py:30/56/103/175`），默认 sglang 本地推理 → **成本最低的一家**

### graphiti —— 【未找到】
【已验证】`grep -rin locomo graphiti --include=*.py --include=*.md` 无任何命中。graphiti 只有 **LongMemEval**：`graphiti/tests/evals/data/longmemeval_data/`、`graphiti/tests/evals/eval_e2e_graph_building.py`。**要在 graphiti 上跑 LoCoMo 必须自己写适配层。**

### cognee —— 【未找到】LoCoMo
有通用 eval 框架（`cognee/cognee/eval_framework/`，含 corpus_builder / runner / eval_config / CLI `cognee eval`），但【已验证】`benchmark_adapters/benchmark_adapters.py:15-20` 的枚举只有：`Dummy, HotPotQA, LogisticsSystem, Musique, TwoWikiMultiHop, BEAM` —— **没有 LoCoMo**。唯一命中 "locomo" 的文件是 `cognee/tests/test_data/Chinook_PostgreSql.sql`（无关，是数据里的字符串巧合）。
不过它的 adapter 抽象是现成的，**加一个 LocomoAdapter 的成本较低**（照抄 `hotpot_qa_adapter.py`）。

---

## 3. ★ 记忆条目粒度（决定「最小可拔除单元」）

| 框架 | 切分粒度 | 证据 |
|---|---|---|
| **mem0-benchmarks** | **单轮 turn**（每次 `add` 只送 1 条 message），再由 mem0 服务端 LLM 抽成**原子事实**存储 | 【已验证】`mb/benchmarks/locomo/run.py:88 CHUNK_SIZE = 1  # turns per ingestion chunk`；`:189 for i in range(0, len(messages), CHUNK_SIZE)`；`:186 messages.append({"role": role, "content": f"{speaker}: {text}"})`；`:340 await mem0.add(messages, user_id, timestamp=session_epoch)` → OSS `POST /memories`（`mem0_client.py:158`）返回 `{"results":[...]}` 即抽出的事实列表（`run.py:348 for mem_item in results`）。**最小可拔除单元 = mem0 抽出的一条 fact（有独立 id）**，而且因为 CHUNK_SIZE=1，**fact ↔ 源 turn 的对应关系是干净的**（这对做「拔除某条记忆看效果」极其友好）。 |
| **MemOS** | **整个 session 一次性提交**（speaker_a / speaker_b 各建一份视角，`batch_size=2`） | 【已验证】`MemOS/evaluation/scripts/locomo/locomo_ingestion.py:22 ingest_session(client, session, frame, ...)`；`:38-45` 遍历 session 内所有 chat 拼成 `speaker_a_messages` / `speaker_b_messages`（**同一句对两个 speaker 分别标 user/assistant**）；`:52-83` 一次 `client.add(speaker_a_messages, ...)`。每条 message 带 `chat_time`（`:48-51`）。**粒度较粗：拔除单元取决于后端自己抽了什么，源头 traceability 弱于 mem0。** |
| **A-MEM** | **单轮 turn → 一条 MemoryNote**，LLM 再给它抽 keywords / context / tags / links | 【已验证】`amem_repo/test_advanced.py:301-305`：`for _,turns in sample.conversation.sessions.items(): for turn in turns.turns: conversation_tmp = "Speaker "+turn.speaker+"says : "+turn.text; agent.add_memory(conversation_tmp, time=turn_datatime)`；`memory_layer.py:261 class MemoryNote`（`:266 keywords / :271 context / :275 tags / :267 links`，`:281` 缺省时调 LLM 补全）。**最小可拔除单元 = 一条 MemoryNote ≈ 一个 turn**，且 note 之间有 links 图 —— 拔除会级联影响链接。 |
| **EverOS** | **单条 message**，带原始 `dia_id` + 30 秒递增的毫秒时间戳，**显式保留 session 边界** | 【已验证】`EverOS/benchmarks/run.py:597-609`：`msgs.append({"dia_id": msg["dia_id"], "speaker":..., "text":..., "timestamp_ms": base_ts_ms + i*30000})`；`:609 sessions.append({"session_idx": session_idx, "messages": msgs})`；`:541` 注释说明是为 per-session flush。**保留 dia_id 意味着它是几家里唯一能把记忆条目反查回原始对话行的**，做「可拔除单元」实验最友好的数据结构。 |

**给宿主选型的建议（推断）**：若实验核心是「拔掉某条记忆看下游影响」，粒度最细且溯源最干净的是 **mem0-benchmarks（CHUNK_SIZE=1）** 和 **EverOS（保留 dia_id）**；MemOS 的整 session 批喂会让「哪条 turn 生成了哪条记忆」变得模糊，但它是唯一能一键跨 7 个后端对比的。

---

## 4. 打分方式

**主指标全部是 LLM-as-judge 二分类**（沿用 LoCoMo/mem0 论文口径），F1/BLEU 只在部分仓库作辅助。

- **mem0-benchmarks**：【已验证】`mb/benchmarks/locomo/prompts.py:7` 注释原文 "binary LLM judge (CORRECT/WRONG) on categories 1-4"；`:33 CATEGORIES_TO_EVALUATE = [1,2,3,4]`（**category 5 被排除**）。现成 judge prompt：`:203 JUDGE_SYSTEM_PROMPT`、`:218 _JUDGE_TEMPLATE`、`:270 JUDGE_PROMPT`、`:277 get_judge_prompt()`、`:295 get_judge_prompt_with_evidence()`。judge 规则相当宽松且写得很细（`:222` PARTIAL CREDIT 命中 1 项即算对、`:224` 同义改写算对、`:226` 多给细节不扣分、`:228` **日期 14 天容差、时长 50% 容差**、`:230` 情绪同极性算对、`:232` 同实体算对）。评分阈值：`run.py:629 correct = sum(1 for s in scores if s >= 0.5)`。答案生成 prompt 在 `prompts.py:40 ANSWER_GENERATION_PROMPT`，喂给 answerer 的记忆上限 `:101 ANSWERER_MEMORY_LIMIT = 200`。
  - ⚠️ 注意：这套 judge prompt 对「部分正确」极度宽容（1/4 命中即 CORRECT），**用它跟其他论文的数字横比要小心**。
- **MemOS**：【已验证】两套都有。judge：`locomo_eval.py:74 async def locomo_grader(...)` + `:75 system_prompt` + `:79 accuracy_prompt`，返回 JSON label（`:51 extract_label_json`）。NLP 指标：`:123 calculate_rouge_scores`、`:136 calculate_bleu_scores`、`:155 calculate_meteor_score`、`:163 calculate_semantic_similarity`、`:178 calculate_f1_score`、`:197 calculate_nlp_metrics`。聚合输出含 `f1/rouge1_f/rouge2_f/rougeL_f/bleu1-4/bert_f1/similarity/llm_judge_score(±std)`（`locomo_metric.py:56-69, :98-99, :267-268`），还能导 Excel（`:303 save_to_excel`）。**judge 跑 3 次取均值+方差**，是几家里统计最严谨的。
- **EverOS**：【已验证】`benchmarks/run.py:179` judge prompt 文本 "which you will score as CORRECT/WRONG"，`:197` "Just return the label CORRECT or WRONG in a json format with the key as \"label\""。README:237 显示默认 judge 跑 3 次并统计 "unanimous 95.2%"（三次一致率）。
- **A-MEM**：【已验证】`test_advanced.py:342-344` 指标字段 `exact_match / f1 / rouge1_f / rouge2_f / rougeL_f / bleu1-4 / bert_f1 / meteor / sbert_similarity` —— **纯 NLP 指标，没有 LLM judge**（这也是 A-MEM 论文数字跟 mem0 那套 judge 数字不可直接比的原因）。它按 category 定制答案 prompt（`:127 answer_question`，`:155/:162/:169/:175` 分支）。

---

## 5. 有没有跨框架统一 harness？

- **【已验证】有，MemOS 的就是。** `MemOS/evaluation/scripts/locomo/locomo_ingestion.py:216-224` 一个 `--lib` 支持 `mem0 / mem0_graph / memos-api / memos-api-online / memobase / memu / supermemory`，且 `:64` `elif "mem0" in frame:` 和 `:101 if frame == "mem0" or frame == "mem0_graph":` 有针对 mem0 的分支处理（含 `client.client.delete_all(user_id=...)` 清库，`:107-108`）。还带 `locomo_openai.py`（OpenAI 原生 Memory 对照）和 `locomo_rag.py`（朴素 RAG baseline）。**这是目前最现成的跨框架 LoCoMo harness。**
- **mem0 官方 harness 只覆盖自家**（`mb/benchmarks/common/mem0_client.py` 是唯一的后端 client），但抽象层薄、易扩展。它同时含 LoCoMo + LongMemEval + BEAM 三个基准（`mb/benchmarks/{locomo,longmemeval,beam}/`），还自带一个 Next.js 结果对比 UI（`mb/src/app/compare/page.tsx`、`mb/src/app/api/compare/route.ts`）。
- **cognee 有历史第三方对比代码但只针对 HotpotQA**：【已验证】`cognee/evals/old/hotpot_qa_24_2025/src/modal_apps/modal_qa_benchmark_{cognee,graphiti,lightrag,mem0}.py` —— 结构可参考，但**不是 LoCoMo**。
- 【未找到】除以上之外的、公认的第三方跨框架 LoCoMo 对比仓库（本次只查了本地 5 个仓 + 2 个 GitHub 仓，未做全网普查，此项证据不充分）。

---

## 6. 成本量级

- **EverOS 是唯一给了明确数字的**，【已验证】`EverOS/benchmarks/README.md:318-325`：

| 范围 | 时间 | Token 量级 |
|---|---|---|
| Smoke（冒烟） | 2–5 min | ~80k tokens |
| 单个 conversation（完整） | 15–30 min | ~1M tokens |
| **全 10 conv 完整跑** | **2–4 小时** | **~10M tokens** |

  README:236-239 的实测明细：Answer 阶段 1540 题、平均 4.7s、7,224,168 tokens；Judge 阶段 1540 题 × 3 次、2,335,683 tokens；**合计 9,559,851 tokens**。README:325 明确说 answer+judge 两阶段占了绝大部分 token 成本（写入阶段相对便宜）。
  - 【推断】按 gpt-5 级别定价粗算（输入 ~$1.25/M、输出 ~$10/M，假设 9 成是输入），**一次全量跑约 $15–40 美元**；用 gpt-4o-mini 级别做 judge 可压到 $2–5。此为我的估算，非仓库原文。
- **mem0-benchmarks**：【未找到】美元/token 成本说明。只有 `mem0/README.md:49` 的 "7.0K tokens / 0.88s" 是**单次检索**的 token 数（不是全量跑成本），`mem0/docs/core-concepts/memory-evaluation.mdx:131` 同口径 "LoCoMo 92.5 / 6,956 tokens"。【推断】LoCoMo 约 1540 题 × 7K ≈ 10.7M 检索侧 token，量级与 EverOS 的 ~10M 吻合。
- **MemOS / A-MEM**：【未找到】成本说明。A-MEM 默认走本地 sglang（`memory_layer.py:103 SGLangController`），且带 memory/retriever pickle 缓存（`test_advanced.py:280-292`，重跑不重新建记忆），**实际 API 花费可接近 0**。

---

## 相关笔记

- [[评测与数据集全景]]
- [[LoCoMo 真实规模实测]]
- [[mem0-graphiti-zep 六轴调研]]
