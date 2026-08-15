---
title: "LongMemEval harness 审计"
tags:
  - 评测轴
  - agent-memory
---

审计对象：`https://github.com/xiaowu0162/LongMemEval`（已 clone 到 `<本地路径>`，depth=1）
数据：已下载 `longmemeval_oracle.json`(15MB) 与 `longmemeval_s_cleaned.json`(278MB) 到 `lme/data/`，**未下载 m（2.7GB）**，仅用 HF API 读元信息。
标注约定：【已验证:文件:行号】/【推断】/【未找到】

---

## 结论先行

1. **可以做，但官方 harness 本身几乎不用改——因为它本来就没有"记忆系统"，只是一个"把 haystack_sessions 拼成 prompt → 调 OpenAI → LLM judge"的 400 行脚本。**
   拔除单条 session 的最简做法是 **不改代码，在数据层面做**：读入 json，把某个 `haystack_session_ids[i]` 及其对应的 `haystack_sessions[i]`/`haystack_dates[i]` 三元组同步删掉，写成新 json 喂给 `--in_file`。**0 行代码改动**。
2. **可重放性：生成端和评分端都是 `temperature: 0`【已验证】，但全局无任何 seed 设置、无 top-p/seed 参数、无缓存**。temperature=0 只是"近似确定"，OpenAI 侧仍有非确定性。反事实差值的噪声底需要用 **同输入重复 N 次** 实测，或自己加 `seed=` 参数（OpenAI 支持）。
3. **turn 级拔除也可行**：turn 在数据里**没有显式 ID 字段**，但代码里用 `session_id + '_' + str(i_turn+1)` 合成稳定 corpus id【已验证:run_generation.py:79-80】，位置索引即稳定 ID（只要不重排 list）。
4. **最大的坑：官方 harness 没有 checkpoint / 缓存 / 断点续跑**，异常直接 `continue` 丢样本【已验证:run_generation.py:376-378】。跑 Shapley 排列采样（成百上千次重评）**必须自己套一层缓存**。
5. **推荐用 oracle 版做反事实实验**：每题中位 2 个 session、平均 ~6.6k token，500 题一轮全量约 **330 万 prompt token**；s 版一轮 **~6100 万 token**（每题 ~122k），贵约 18.5 倍。Shapley 采样场景下 oracle 是唯一现实选项。

---

## A. 官方 harness 能力

### A1. 目录结构与代码规模【已验证】

全仓库仅 **15 个 Python 文件，2354 行**（含 data 目录的构造脚本）：

| 文件 | 行数 | 作用 |
|---|---|---|
| `src/generation/run_generation.py` | 388 | **主入口**：拼 prompt + 调 LLM + 落 jsonl |
| `src/generation/run_generation.sh` | 87 | 上层 wrapper，model_zoo / retriever 别名映射 |
| `src/retrieval/run_retrieval.py` | 422 | BM25 / dense 检索 |
| `src/retrieval/run_retrieval.sh` | - | 检索 wrapper |
| `src/retrieval/eval_utils.py` | 47 | 检索指标 |
| `src/retrieval/index_expansion_utils.py` | 81 | 索引扩展工具 |
| `src/evaluation/evaluate_qa.py` | **134** | **LLM-as-judge 评分（核心）** |
| `src/evaluation/print_qa_metrics.py` | 33 | 从 log 聚合分数 |
| `src/evaluation/print_retrieval_metrics.py` | 43 | 检索指标聚合 |
| `src/index_expansion/*.py` (7 个) | 791 | 离线 key expansion（summ/keyphrase/userfact/temp-event） |
| `data/custom_history/sample_haystack_and_timestamp.py` | 417 | 复现 haystack 构造 |

**完整评测入口命令**（两步，README:170-182 + 92-107，与代码一致）：
```bash
# 1) 生成
cd src/generation
bash run_generation.sh ../../data/longmemeval_s.json gpt-4o full-history-session 1000 json false con
# 2) 评分
cd src/evaluation
export OPENAI_API_KEY=...
python3 evaluate_qa.py gpt-4o <hyp_file> ../../data/longmemeval_oracle.json
```

### A2. 支持哪些 memory 方法/后端【已验证】

**没有任何第三方 memory 系统接入（无 mem0 / MemGPT / Zep / Letta 的一行代码）**。枚举点有两处：

- **generation 侧 `retriever_type`**，`run_generation.py:91-169` 的 if/elif 链共 7 种：
  `orig-session` / `orig-turn`（长上下文直灌全历史）、`oracle-session` / `oracle-turn`、`flat-turn` / `flat-session`（消费检索结果）、`no-retrieval`（闭卷）。
  注意 **`oracle-session`/`oracle-turn` 的实现和 `orig-*` 逐字相同**（run_generation.py:105-116 vs 91-102），"oracle"是靠喂 `longmemeval_oracle.json` 这个数据文件实现的，不是代码逻辑。
- **retrieval 侧 `--retriever`**，`run_retrieval.py:234` / `276-278`：`flat-bm25`、`flat-contriever`、`flat-stella`、`flat-gte`、`oracle`。
- **reader 模型枚举**：`run_generation.sh:18-31`（model_zoo，gpt-4o / llama3.x / mistral / phi / FILM-7B）+ `run_generation.py:315-327`（model2maxlength）。开放权重模型通过 vllm 起 OpenAI 兼容 server（`src/utils/serve_vllm.sh`）。

→ **结论：官方 harness = long-context 直灌 + 一组经典 RAG 检索器 + 索引扩展 trick。第三方 memory 系统需要自己写适配层。**

### A3. 评分方式【已验证】

**纯 LLM-as-judge，无字符串匹配。**
- judge prompt 在 **`src/evaluation/evaluate_qa.py:24-43`，函数 `get_anscheck_prompt(task, question, answer, response, abstention)`**。
- **确实按 question_type 分流**（evaluate_qa.py:26-39）：
  - `single-session-user` / `single-session-assistant` / `multi-session` → 共用同一个通用模板（L27）
  - `temporal-reasoning` → 额外容忍 off-by-one 天数误差（L30）
  - `knowledge-update` → 允许回答里带旧信息，只要更新后的答案对（L33）
  - `single-session-preference` → 按 rubric 判"是否正确调用了用户个人信息"（L36）
  - **abstention**：由 `question_id` 是否以 `_abs` 结尾触发（L101），走单独模板（L41）
- judge 模型只允许三个：`llama-3.1-70b-instruct` / `gpt-4o-mini` / `gpt-4o`（`evaluate_qa.py:11-15`），非白名单直接 exit（L58-60）。
- judge 调用 `temperature: 0, max_tokens: 10`（L108-109），判定逻辑是 `'yes' in eval_response.lower()`（L113）——**极其脆弱的字符串后处理**，judge 若输出 "no, ... yes" 会误判。

### A4. checkpoint / 缓存 / 断点续跑【已验证:不存在】

- `run_generation.py`：输出文件名带 `datetime.now().strftime("%Y%m%d-%H%M")`（L307-309），**每次跑都是新文件，无法续跑**；主循环 L337 直接 `for entry in tqdm(in_data)`，无 skip-if-done；异常时 `print(...); continue`（L376-378）**静默丢样本**。
- `evaluate_qa.py`：同样是全量重跑，`with open(result_file, 'w')` 覆盖写（L88）。
- 唯一的"缓存"是 `src/index_expansion/*.py` 里的离线扩展结果缓存（如 `batch_expansion_session_summ.py:48-52`），且路径是作者硬编码的 `/local2/diwu/...`，**对该方向无用**。
- `run_retrieval.py:31` 有 `--cache_dir`，但那是 HF 模型权重缓存，不是结果缓存。

→ **反事实实验必须自建缓存层**：key = `hash(question_id + 被拔除的 session 集合 + model + prompt 模板)`。

---

## B. ⭐ 单条记忆可拔除性

### B5. 能否控制"注入哪些 session"【已验证：数据层即可，代码无需改】

关键代码 `run_generation.py:92-96`（`orig-session` 分支）：
```python
for session_date, session_entry in zip(entry['haystack_dates'], entry['haystack_sessions']):
    retrieved_chunks.append((session_date, session_entry))
```
注入内容**完全由 entry 的三个平行 list 决定**：`haystack_dates` / `haystack_session_ids` / `haystack_sessions`（三者按 index 对齐，L75-77 明确 zip 三者建映射）。

**屏蔽指定 session 的最小改动路径（三选一）：**

| 方案 | 改动量 | 说明 |
|---|---|---|
| **A. 数据层预处理（推荐）** | **0 行改 harness**，写 ~15 行独立脚本 | 读 json → 对每个 entry 按 `haystack_session_ids.index(sid)` 删掉三个 list 的同一下标 → dump 新 json → 正常跑。完全不碰仓库代码 |
| B. 在 run_generation.py 加 `--exclude_sessions` | **~8 行**：`parse_args` 加 1 行（L17 附近）；`prepare_prompt` 开头加 5-6 行做 zip 三 list 后 filter | 需同时过滤 L75 的 `corpusid2*` 构建循环，否则 flat-* 分支会脏 |
| C. 只做 no-retrieval 上限/下限对照 | 0 行 | `retriever_type=no-retrieval`，L165-166 直接空 context |

**注意两个陷阱：**
- L172-174：`orig-*` 走 `retrieved_chunks[-topk_context:]`（**保留最新的 K 个**），其他走 `[:topk_context]`。做反事实时必须把 `TOPK` 设到 1000+（README:177 也这么建议），否则"拔掉一条"会连带把截断边界推移，污染 treatment。
- L266-279：超长会按 tokenizer 截断到 `model_max_length - gen_length - 1000`。**s 版每题 ~122k token，对 128k 窗口的 gpt-4o 已经贴边**，拔一条 session 可能改变是否触发截断 → 差值被截断效应污染。**再次支持用 oracle 版做反事实。**

### B6. session 内可否细分到 turn；turn 有无稳定 ID【已验证】

- 数据里 turn 是 `{"role", "content"}`，**证据 turn 额外带 `has_answer: true`**（oracle 版实测：10960 个 turn 中 896 个 `has_answer=True`）。**没有显式 turn id 字段**。
- 代码里合成稳定 ID：`session_id + '_' + str(i_turn+1)`（**run_generation.py:78-80**），检索侧也用同一套（`run_generation.py:121-122` 做反解析）。
- turn 级注入已有现成分支：`orig-turn`（L97-102）和 `flat-turn`（L119-144）。
- **注意 `flat-turn` 会自动把 turn 扩成一个 round（把下一条也带上，L126-128）**，做 turn 级反事实时这会串味。
- `useronly` 开关（L93-94 等）可只保留 user 侧消息，是一种粗粒度"拔除 assistant 记忆"。

→ **turn 级拔除可行**：位置索引即 ID，只要不重排 `haystack_sessions[i]` 这个 list。建议自己在预处理时把 `(session_id, turn_idx)` 显式物化成 `turn_uid` 字段，避免下游漂移。

### B7. 随机性与可重放【已验证 + 部分未找到】

- **`import random` 存在但从未调用、从未 `random.seed()`**（run_generation.py:7；全仓库 grep `seed` 在 src/ 下 **0 命中**）。
- 生成：`'temperature': 0`（**run_generation.py:366**），`n: 1`。**没有 `seed=` 参数、没有 `top_p`**。
- judge：`'temperature': 0`（**evaluate_qa.py:108**），`max_tokens: 10`。
- 唯一真正用 `random` 的是 **数据构造脚本** `data/custom_history/sample_haystack_and_timestamp.py`（L30/40/191/196/217/407 等大量 `random.choice`/`shuffle`），**同样没有 seed** → 官方 haystack 构造过程不可复现。但我们用发布好的 json，这条不影响。

**结论：**
- ✅ 输入相同 → prompt 字节级相同（拼接逻辑纯确定性，唯一排序是 `retrieved_chunks.sort(key=lambda x: x[0])` 按日期，**注意这是不稳定排序键：同日期 session 的相对顺序依赖原 list 顺序**，L225 —— 拔除操作不改变其余元素相对序，所以安全)。
- ⚠️ 输出**不保证**逐字相同：temperature=0 在 OpenAI 侧仍非确定（MoE 路由 / batch 非确定性）。
- 🔧 **必做**：(a) 给两处 `kwargs` 加 `"seed": 42`；(b) 落盘 `system_fingerprint`；(c) 跑 baseline 重复 5 次量化 judge 层面的 flip rate，作为 Shapley 差值的噪声底线。

---

## C. 三个数据版本的区别

### C8. 定义与实测数字

**README 原文定义（README.md:74-77）：**
> * `longmemeval_s.json`: ... Concatenating all the chat history roughly consumes **115k tokens (~40 history sessions)** for Llama 3.
> * `longmemeval_m.json`: ... Each chat history contains roughly **500 sessions**.
> * `longmemeval_oracle.json`: LongMemEval with oracle retrieval. **Only the evidence sessions** are included in the history.

构造参数（README.md:139-140）：`min/max_n_haystack_filler` = **80 用于 s，500 用于 m**；`enforce_json_length` = **115000 用于 s，m 不设限**。

**本地实测（oracle + s 均已下载并统计）：**

| 指标 | oracle【已验证】 | s【已验证】 | m【推断】 |
|---|---|---|---|
| 题数 | 500 | 500 | 500 |
| session/题 min/中位/均值/max | 1 / 2 / 1.90 / 6 | 38 / 48 / **47.7** / 62 | ~500（README） |
| 总 session 数 | 948 | 23,867 | ~250,000 |
| 总 turn 数 | 10,960 | 246,750 | ~2.4M |
| 内容字符总量 | 13.2 M | 244.6 M | ~2.4 G |
| **≈token/题**（chars/4） | **~6.6k** | **~122k** | **~1.2M** |
| ≈token 总量 | **3.30 M** | **61.2 M** | **~603 M** |
| 文件大小（HF API 实测） | 15,388,478 B | 277,383,467 B | 2,737,100,077 B |

- oracle 的 `haystack_session_ids` 全是 `answer_*` 前缀且 == `answer_session_ids`（实测第 0 题两者完全一致）→ **确认 oracle 无干扰项**。
- s 的 session id 混入 `sharegpt_*` 等 filler（实测 `['sharegpt_yywfIrx_0', '85a1be56_1', ...]`），`answer_session_ids` 平均 **1.90 条/题**（最多 6）→ **s 每题约 45.8 条纯干扰 session**。
- m 数字为【推断】：按文件字节比 2737100077/277383467 = **9.87×**，README 说 ~500 sessions（vs s 的 ~48）也是 ~10×，两者互相印证。**未下载验证**。
- 题型分布（oracle 实测）：temporal-reasoning 133 / multi-session 133 / knowledge-update 78 / single-session-user 70 / single-session-assistant 56 / single-session-preference 30。含 30 道 abstention（`_abs` 后缀，README:206）。
- ⚠️ **数据源已变更**：README News（L15）说 2025/09 清洗过，官方现在指向 **`xiaowu0162/longmemeval-cleaned`**（文件名 `longmemeval_s_cleaned.json` / `longmemeval_m_cleaned.json`），而任务里给的 `xiaowu0162/longmemeval` 是旧版。两 repo 的 oracle 文件 oid 完全相同（`8e6e661d...`），s/m 不同。**建议用 cleaned 版**（本报告统计基于 cleaned）。

### C9. 单次完整评测成本

**README/论文中未给出成本数字【未找到】**。以下全部为【推断】，基于上表 token 量。

| 版本 | 生成阶段 prompt token | 生成 completion | judge token | API 调用次数 |
|---|---|---|---|---|
| oracle | ~3.3 M | 500×~800 = 0.4 M | ~0.3 M | 500 生成 + 500 judge = **1000** |
| s | ~61 M（若 reader 窗口 128k 基本不截断） | 0.4 M | ~0.3 M | **1000** |
| m | ~603 M —— **超出所有 128k 模型，README:175 明确说 m 太长不适合 long-context 测试**，只能走 RAG 路径 | - | - | 生成 500 + 检索索引 ~250k 条 embedding |

**按 gpt-4o(2024-08-06) 定价 $2.5/M input 估算【推断】：oracle ≈ $9/轮，s ≈ $155/轮，m 走长上下文不可行。**
若用 `--con`（con-separate，run_generation.py:194-222），**每个 chunk 额外一次 LLM 调用** → oracle 一轮的调用次数从 1000 涨到 500+948+500 ≈ **1950 次**，s 涨到 ~24,900 次。做 Shapley 时**务必关掉 con**。

**对 Shapley 排列采样的含义【推断】**：oracle 平均 1.90 个 evidence session（最多 6），若只对 evidence session 求 Shapley，每题的全排列枚举最多 6! 但边际集合只有 2^6=64 个子集，**穷举可行**（配合缓存实际唯一评测点 ≤ 64/题）。若把干扰 session 也纳入（s 版 ~48 条），必须排列采样，成本立刻爆炸。→ **强烈建议：反事实实验跑 oracle 版，先做 evidence-set 的精确 Shapley。**

---

## D. 第三方适配层

### 本机 grep 结果（`grep -ril longmemeval` on memres/）

| 项目 | 文件路径 | 支持程度 |
|---|---|---|
| **mem0 官方 benchmarks (`mb/`)** | `mb/benchmarks/longmemeval/run.py`（**1413 行**）、`mb/benchmarks/longmemeval/prompts.py`（359 行） | ⭐**最完整**。含 `download_dataset`(L204)、`sort_sessions_chronologically`(L291)、`pair_turns`(L314)、`ingest_question`(L332)、answerer/retrieval 双路径(L509/L631)、**自带 judge**(`get_retrieval_judge_prompt` L172, `apply_longmemeval_answerer_judge_to_saved_result` L728)、**分层采样 + `--seed` 默认 42**(L240/L254/L1028/L1100)、结果落盘含 seed(L1223/L1398)、`longmemeval_predict_outputs_complete`(L840) 做**续跑判断**。已有跑分结果 `mb/results/oss/longmemeval_{gpt5,gemma4_31b,llama4_maverick,parasail_120b}.json` 和 `mb/results/platform/longmemeval_top50_results.json` |
| **MemOS** | `MemOS/evaluation/scripts/longmemeval/`：`lme_ingestion.py`(193)、`lme_search.py`(255)、`lme_rag.py`(315)、`lme_responses.py`(154)、`lme_metric.py`(282)、`lme_eval.py`(385)；驱动脚本 `MemOS/evaluation/scripts/run_lme_eval.sh` | ⭐完整流水线（ingest→search→rag→response→metric）。`lme_ingestion.py:63-65` 直接消费 `haystack_sessions` |
| **graphiti** | `graphiti/tests/evals/eval_e2e_graph_building.py:76-90`，数据 stub `graphiti/tests/evals/data/longmemeval_data/README.md` | 🟡**仅 oracle，且只测"图构建"不测 QA 端到端**。L77-78 硬编码 `data/longmemeval_data/longmemeval_oracle.json`，注释说 "Can be _oracle, _s, or _m"；按 `multi_session_idx` 逐题灌入 `user_id='lme_oracle_experiment_user_N'` |
| **cognee** | `cognee/cognee/eval_framework/runner.py:228`、`cognee/cognee/eval_framework/README.md:44` | 🔴**仅文档提及**，措辞是 "Once registered, LongMemEval works here too" —— **代码里没有 LongMemEval adapter 实现**【已验证:未找到实现】 |
| **mem0 主仓** | `mem0/docs/core-concepts/memory-evaluation.mdx:216,224` | 🟡仅文档，指向 `python -m benchmarks.longmemeval.run`，即上面的 `mb/` |
| **EverOS / amem_repo / evomemory** | — | 🔴无命中（evomemory 仅 PDF 里提到） |

## 相关笔记

- [[评测与数据集全景]]
- [[冲突benchmark全景]]
