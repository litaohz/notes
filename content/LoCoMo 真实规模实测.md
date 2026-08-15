---
title: "LoCoMo 真实规模实测"
tags:
  - 评测轴
  - agent-memory
---

数据来源：`https://raw.githubusercontent.com/snap-research/locomo/main/data/locomo10.json`
文件大小：**2,805,274 字节（约 2.67 MiB）**，实测下载并用 Python 3.12.10 完整解析。所有数字均为实际统计，非论文引用。

---

## 结论先行

1. **有效样本量对排列采样是够的，但结构上很不平衡。** 全量 **1,986 条 QA**、**5,882 个对话轮次**、**272 个 session**，但只有 **10 个 conversation**——如果估计量的方差主要来自 conversation 级别（把 conversation 当独立单元），n=10 会非常吃紧；如果 QA 级别独立可用，n≈1,986（去掉对抗题后 1,540）就宽裕得多。建议把方差分析做成分层的：conversation 是 cluster，QA 是 unit，用 cluster-robust 或 conversation-level bootstrap。
2. **最关键的 evidence 字段：存在，100% 覆盖，格式是 `dia_id` 字符串列表，如 `["D3:13", "D2:14"]`**，可以精确定位到某个 session 的第几轮。**这意味着"证据轮次"完全可以作为可拔除的记忆单元**——反事实拔除实验在数据层面是可行的。
3. 但**可拔除性的信号强度分布极偏**：1,986 条 QA 中 **1,559 条只有 1 条 evidence**（78.5%），只有 **423 条（21.3%）是多证据**，其中 **329 条跨 session**。单证据题的"拔除 → 答不出"是近乎确定性的（信息量低），真正能体现"记忆价值排序"的是那 423 条多证据题。**这才是实验的有效样本量，n≈423，按 conversation 分层后每个 conversation 平均只有 42 条**——这是方差风险的真正来源，需要重点评估。
4. observation 字段是天然的记忆条目库：**共 2,541 条 observation，每条自带 `dia_id` 溯源**，与 QA 的 evidence 用的是同一套 ID 空间，可以直接对齐。这是比"原始轮次"更好的记忆单元粒度。
5. event summarization 任务标注**存在**（`event_summary`，272 个 session、669 条事件），另有 `session_summary`（每 session 一段自然语言摘要）。

---

## 1. 顶层结构

- 顶层是一个 **list，长度 = 10**（核实：确为 10 个 conversation）。
- 每个元素的 key：`['qa', 'conversation', 'event_summary', 'observation', 'session_summary', 'sample_id']`。

## 2. Session / 轮次 / 体量

| conv | sessions | 总轮次 | 正文字符 | QA 数 | observation 数 |
|---|---|---|---|---|---|
| 0 | 19 | 419 | 57,690 | 199 | 184 |
| 1 | 19 | 369 | 43,587 | 105 | 169 |
| 2 | 32 | 663 | 89,736 | 193 | 324 |
| 3 | 29 | 629 | 71,843 | 260 | 266 |
| 4 | 29 | 680 | 86,298 | 242 | 267 |
| 5 | 28 | 675 | 80,224 | 158 | 277 |
| 6 | 31 | 689 | 80,947 | 190 | 268 |
| 7 | 30 | 681 | 73,258 | 239 | 291 |
| 8 | 25 | 509 | 62,435 | 196 | 240 |
| 9 | 30 | 568 | 80,738 | 204 | 255 |
| **合计** | **272** | **5,882** | **726,756** | **1,986** | **2,541** |

- 每个 conversation 平均 **27.2 个 session**、**588 轮**、约 **72.7K 字符**。
- 单 session 轮次范围 **10 ~ 47**，中位数约 20。
- 总词数 **133,772**，粗估 token ≈ **178K**（×1.33）；单 conversation 平均约 **17.8K token**——注意这是纯文本，塞进 128K 上下文完全放得下，"长上下文 baseline 打不过"不是长度限制导致的。
- 每个 turn 的字段：`speaker` / `dia_id` / `text`（5,882 全覆盖），另有 **1,226 个轮次带 `blip_caption`（图片描述）**、910 个带 `img_url`、888 个带 `query`、206 个带 `re-download`。即 **约 20.8% 的轮次是多模态的**，反事实拔除时需要决定图片信息是否一并拔除。
- 会话时间戳在 `conversation` 里以 `session_<n>_date_time` 形式给出（与 session 平级的 key），另有 `speaker_a` / `speaker_b`。

## 3. QA 总数与类型

- **总计 1,986 条 QA**，**每个 conversation 平均 198.6 条**（范围 105 ~ 260，最少的 conv1 只有 105，最多 conv3 有 260）。
- QA 字段：`question`(1986) / `evidence`(1986) / `category`(1986) / `answer`(1542) / `adversarial_answer`(446)。
- **category 是整数编码，数据文件里没有字符串标签。** 实测分布：

| category | 条数 | 占比 | 平均 evidence 数 | 推断类型 |
|---|---|---|---|---|
| 4 | 841 | 42.3% | 1.07 | single-hop |
| 5 | 446 | 22.5% | 1.03 | adversarial（无 `answer`，只有 `adversarial_answer`） |
| 2 | 321 | 16.2% | 1.17 | temporal（答案多为日期，如 "7 May 2023"、2022） |
| 1 | 282 | 14.2% | **3.13** | multi-hop |
| 3 | 96 | 4.8% | 2.08 | open-domain / commonsense 推断 |

类型名称说明：**文件内未找到任何字符串型 category 标签**（搜过 key `category`、`type`、`qa_type`、`label`，值均为 int）。上表的类型归属是根据 evidence 数量分布 + 样例内容 + 「category 5 独有 `adversarial_answer` 字段」推断的，**category 5 = adversarial 是文件内证据直接支持的（100% 无 `answer`）**；1/2/3/4 的命名属于强推断，正式引用前建议对照官方评测脚本确认。

- 样例：
  - `{"question":"When did Caroline go to the LGBTQ support group?","answer":"7 May 2023","evidence":["D1:3"],"category":2}`
  - `{"question":"What did Caroline realize after her charity race?","evidence":["D2:3"],"category":5,"adversarial_answer":"self-care is important"}`

## 4. evidence 字段（最关键）

**存在，1,986/1,986 全覆盖，类型 100% 为 list。**

- 元素格式：字符串 `"D<session_num>:<turn_index>"`，例如 `["D1:3"]`、`["D1:9","D1:11"]`、`["D3:13","D2:14"]`。
- 与对话轮次的对应：每个 turn 都有 `dia_id` 字段（如 `{"speaker":"Caroline","dia_id":"D1:1","text":"..."}`），**evidence 里的字符串就是 dia_id，可直接哈希索引到具体轮次**。
- **引用完整性实测：1,986 条 QA 共引用的 dia_id 中，只有 9 个在对话里找不到对应轮次（dangling ref）**，其余全部可解析。健康度 >99%，实验时把这 9 条剔除即可。
- evidence 数量分布：

| evidence 条数 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8+ |
|---|---|---|---|---|---|---|---|---|---|
| QA 数 | 4 | 1,559 | 239 | 83 | 58 | 20 | 6 | 7 | 10（最长 19） |

- **多证据 QA：423 条（21.3%）；其中跨 session 的：329 条（16.6%）。**
- 对反事实拔除实验的直接含义：
  - 单证据题（1,559 条）拔除唯一证据 → 理论上必然答不出，Shapley/排列采样在这些题上退化成常数，**贡献零方差也零信息**，可以解析求解不必采样。
  - **真正需要排列采样的是 423 条多证据题**，其中 evidence≥3 的只有 **181 条**。按 conversation 分摊，evidence≥3 的题每个 conversation 平均仅 18 条。若估计量在 conversation 层聚合，方差会相当大，建议：(a) 以 QA 为主单元、conversation 为随机效应；(b) 对 evidence≤2 的题用闭式解替代采样，把采样预算全压到 181 条长证据题上。

## 5. observation 字段

- 结构：`observation` 是 dict，key 为 `session_<n>_observation`（与 session 一一对应），value 又是 dict，key 为两个说话人姓名（如 `Caroline` / `Melanie`）。
- 每个说话人下是一个 list，元素为 **`[观察语句, dia_id]` 二元组**。样例：
  ```
  ["Caroline attended an LGBTQ support group recently and found the transgender stories inspiring.", "D1:3"]
  ["The support group has made Caroline feel accepted and given her courage to embrace herself.", "D1:7"]
  ```
- **总条目数 2,541 条**（每 conversation 169 ~ 324，平均 254.1）。
- 关键性质：**observation 自带 dia_id 溯源，与 QA evidence 共用同一 ID 空间**。所以可以建立 `observation ↔ evidence turn ↔ QA` 的三层映射，把 observation 当作原子记忆单元来拔除，比直接拔除原始轮次语义更干净（一个轮次可能承载多条 observation，也可能一条都不承载）。数量级 2,541 也比 5,882 轮次小一半，采样成本更低。

## 6. event summarization 标注

**有。**

- `event_summary`：dict，key 为 `events_session_<n>`，value 为 `{说话人A: [事件字符串...], 说话人B: [...], "date": "8 May, 2023"}`。
  - 样例：`{"Caroline": ["Caroline attends an LGBTQ support group for the first time."], "Melanie": [], "date": "8 May, 2023"}`
  - **覆盖 272 个 session，共 669 条事件**（平均每 session 2.46 条，存在大量空 list）。
- `session_summary`：dict，key 为 `session_<n>_summary`，value 为一段自然语言段落摘要（每 session 一段，共 272 段）。可作为 summarization 任务的参考答案。

---

## 附：未找到的字段

- 未找到 QA 的字符串型类别名（搜索 key：`category`/`type`/`qa_type`/`label`/`question_type`，只有 int 型 `category`）。
- 未找到显式的 train/dev/test 划分字段（顶层是扁平 list，元素只有 `sample_id`）。
- 未找到独立的 `answer` 字段用于 category 5（对抗题只有 `adversarial_answer`）。

## 复现命令

```bash
curl -sL -o /tmp/locomo10.json https://raw.githubusercontent.com/snap-research/locomo/main/data/locomo10.json
PYTHONPATH= PYTHONHOME= "C:/ProgramData/Tools/Python/3.12.10/x64/python.exe" your_script.py
```

---

## 相关笔记

- [[评测与数据集全景]]
- [[LoCoMo 适配层实查]]
