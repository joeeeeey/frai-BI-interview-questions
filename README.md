# 现场案例题：候选人面试助手数据分析 / BI（Vibe Coding）

> 本题将在 **45–60 分钟技术面试** 中以「vibe coding」形式完成，**不是离线作业**。  
> 你可以现场自由使用 **任何工具**（如 BigQuery、VS Code、Cursor、ChatGPT、Excalidraw、draw.io、浏览器搜索等）。

请在面试前 **阅读完整题目说明**，我们会直接基于本题进入讨论和动手环节。

---

## 1. 面试形式与「交付物」说明

本题主要考察你在真实业务场景下的：

- 业务 / 数据理解能力  
- 指标与分析设计能力  
- SQL / BigQuery 实战能力  
- 使用 AI 工具（Cursor / ChatGPT 等）辅助工作的习惯与方式  
- 沟通与解释能力

在这 45–60 分钟的 session 中，我们期望你 **至少完成以下交付**：

1. **思路 / 方案结构**
   - 用任意形式表达你的整体思路，例如：
     - 在 Excalidraw / draw.io / 白板工具中画一个简单框架图（数据流、指标框架、分析步骤），或  
     - 写下几条清晰的要点 / TODO 列表  
   - 目标：让我们快速看到你如何从产品问题 → 指标 → 数据 → 查询 / 分析。

2. **代码 / 查询实现**
   - 基于下面给出的两张表结构，完成 **至少 2 个 SQL 查询**（接近 BigQuery 风格即可）：
     - 任务 2.1：按周 + job_level 的助手启用率  
     - 任务 2.2：助手使用深度 vs 通过率 / 体验  
   - 你可以：
     - 在 BigQuery 或任意 SQL 环境实际运行，或  
     - 在编辑器中写出接近可执行的 SQL 伪代码  
   - 允许并鼓励使用 **Cursor / ChatGPT 等 AI 工具** 生成或补全 SQL，但请在面试中说明你的思路、修改和校验方式。

3. **输出结果的解释 / 分析**
   - 选择至少 **一个你写的查询 / 指标结果**，进行简短分析，包含：
     - 你期望这个查询在真实数据上的输出大致是什么样；  
     - 这个结果在业务上意味着什么；  
     - 你会如何向产品 / 业务同事解释；  
     - 你认为这个分析的局限性 / 潜在偏差在哪里（可简要说明）。
   - 如果现场没有真实数据，也可以基于「假设输出示例」进行分析说明。

> 总结：  
> - **思路**（图或要点）  
> - **查询 / 代码**（至少 2 个 SQL）  
> - **结果解释**（至少对 1 个结果做业务向解读）  

---

## 2. 产品背景：候选人「面试助手」

我们有一个 **「面试助手」产品，是给候选人（面试者）使用的**：

- 候选人在 Zoom / Google Meet 等视频面试中，通过浏览器插件或网页接入面试助手；
- 系统实时抓取面试中的音频流 → 做语音转文本（ASR）；
- 再调用大语言模型（LLM），为候选人生成 **实时回答建议**（例如：
  - 回答要点；
  - STAR 结构 / 逻辑框架；
  - 可能的 follow-up 思路等）；
- 这些建议显示在候选人浏览器端，用于辅助他在面试中更好地组织答案。

我们已经积累了一部分使用日志，希望用数据回答：

1. 候选人是否愿意使用这个面试助手？使用情况如何（是否被打开、用得多不多）？  
2. 不同岗位 / 级别 / 区域，对助手的依赖程度是否不同？  
3. 使用助手与面试结果 / 候选人体验之间，是否存在有意义的相关性或模式？

---

## 3. 数据表结构（BigQuery 风格，简化版）

你可以假设已经存在以下两张主表，所有后续任务均基于此展开。

### 3.1 `interview_sessions`（每场面试一行）

```sql
interview_sessions (
  session_id                STRING,
  candidate_id              STRING,
  job_title                 STRING,     -- 如 "Data Analyst", "Backend Engineer"
  job_level                 STRING,     -- 如 "junior", "mid", "senior"
  region                    STRING,     -- 如 "US", "EU", "IN"
  start_time                TIMESTAMP,
  end_time                  TIMESTAMP,
  assistant_enabled         BOOL,       -- 候选人在本场是否打开 / 使用面试助手
  outcome                   STRING,     -- "pass" / "fail" / "no_show"
  candidate_feedback_score  INT64       -- 候选人对面试助手/整体体验的评分 1–5，可为空
);
````

---

### 3.2 `assistant_suggestions`（每条助手建议一行）

```sql
assistant_suggestions (
  suggestion_id     STRING,
  session_id        STRING,     -- 关联到 interview_sessions.session_id
  turn_index        INT64,      -- 面试中的第几轮（第几个问题）
  created_ts        TIMESTAMP,  -- 该条建议生成完成时间
  latency_ms        INT64,      -- 从识别出问题到建议生成完成的延迟（毫秒）
  was_shown         BOOL,       -- 是否成功展示给候选人
  was_copied        BOOL,       -- 候选人是否点击「复制」该建议
  was_sent          BOOL        -- 该建议是否最终被候选人回答中实际使用（粗略标记）
);
```

> 说明：
>
> * 实际生产环境会有更多字段和表，这里为面试场景做了简化；
> * 如无特别说明，你可以基于以上字段做 **合理业务假设**（并在面试中简单说明）。

---

## 4. 任务说明

面试中我们会围绕以下任务展开。你可以根据时间先覆盖核心部分，再视情况延伸。

---

### 任务 1：核心指标设计（业务 & 数据理解）

基于上述两张表，设计 **3–4 个你认为关键的核心指标**，帮助我们理解：

* 面试助手的 **使用情况**（是否被使用、使用程度）；
* 面试助手与 **面试结果 / 候选人体验** 是否存在可见相关性。

对每个指标，请简要说明：

1. **指标定义**：用到哪些字段、如何计算（例如按日 / 周 / job_level 聚合等）；
2. **业务含义**：该指标回答了什么问题，为什么重要。

你可以自由命名指标，例如：

* 助手启用率（Assistant Adoption Rate）；
* 每场平均展示建议数 / 被实际使用的建议数；
* 使用助手 vs 未使用助手的通过率差异；
* 使用助手 vs 未使用助手的平均候选人评分差异；
* 响应延迟相关指标（平均 / P95 `latency_ms`）等。

> 提示：你可以先在 Excalidraw / draw.io / 文本中画出一个简单「指标框架图」或要点列表，再细化到 SQL。

---

### 任务 2：SQL / BigQuery 查询（实战）

请使用接近 BigQuery 的 SQL 完成以下问题。
你可以在任意支持 SQL 的环境中实际运行，或在编辑器中写出接近可执行的 SQL。
如使用 AI 工具生成部分查询，请在现场说明你的调整和验证方式。

#### 任务 2.1：按周 + job_level 的助手启用率

目标：分析最近 8 周面试助手的启用情况。

请编写 SQL：

* 时间范围：限制为「最近 8 周」；
* 维度：按 **每周** + `job_level` 聚合；
* 输出每个（周, job_level）下：

  * `total_sessions`：总面试场次；
  * `assistant_sessions`：`assistant_enabled = TRUE` 的场次；
  * `assistant_adoption_rate`：启用率 = 助手场次 / 总场次。

推荐输出字段：

```text
week_start                -- 每周的起始日期（如用 DATE_TRUNC）
job_level
total_sessions
assistant_sessions
assistant_adoption_rate
```

---

#### 任务 2.2：助手使用深度 vs 通过率 / 体验

目标：观察「使用面试助手的深度」与面试表现、体验的关系。

请设计查询（可用 CTE / 子查询等实现），完成：

1. 在 `assistant_suggestions` 中按 `session_id` 聚合，计算：

   * `used_suggestions` = 每场面试中 `was_sent = TRUE` 的建议条数；
2. 将上述结果与 `interview_sessions` 按 `session_id` 关联；
3. 仅保留 `outcome IN ('pass', 'fail')` 的记录，忽略 `no_show`；
4. 按 `used_suggestions` 将面试分为三档（可使用 `CASE WHEN`）：

   * `0_used`：`used_suggestions = 0`
   * `1_3_used`：`1 <= used_suggestions <= 3`
   * `gt_3_used`：`used_suggestions > 3`
5. 对每个档位统计：

   * `sessions_count`：面试总数；
   * `pass_sessions`：通过场次数；
   * `pass_rate`：通过率 = 通过场次 / 总场次；
   * `avg_candidate_feedback_score`：平均候选人评分。

推荐输出字段：

```text
usage_bucket                 -- '0_used' / '1_3_used' / 'gt_3_used'
sessions_count
pass_sessions
pass_rate
avg_candidate_feedback_score
```

在面试中，我们会请你对其中至少一个结果做业务向解读，例如：

* 如果发现 `gt_3_used` 档位的通过率更高，你会如何解释？
* 这是否足以证明「使用助手导致通过率提升」？如果不能，你认为有哪些可能的混淆因素或样本偏差？

---

### 任务 3（可选，若时间允许）：进一步利用 LLM 的分析思路

此任务偏概念思考，不强制要求写代码，可以口头说明。

> 假设我们还存在一张更细粒度的转写表 `conversation_turns`，记录面试中每一轮发言的文本（包括 interviewer 与 candidate），其字段大致为：
>
> ```sql
> conversation_turns (
>   turn_id    STRING,
>   session_id STRING,
>   speaker    STRING,    -- 'interviewer' / 'candidate'
>   start_ts   TIMESTAMP,
>   end_ts     TIMESTAMP,
>   text       STRING
> );
> ```

请简要说明你的思路：

1. 如果使用 LLM 对这些 `text` 做打标（如：问题类型、回答结构化程度、自信程度等），你会：

   * 希望打哪些标签（字段）？
   * 如何组织一个高层的处理流程（例如批处理、调用方式、结果格式）？
2. 在数据表设计上，你会如何落地这些标签？

   * 新增字段或单独建标签表？大致的 schema 是什么样？
3. 你会如何验证这些标签的质量，确保可用于 BI 分析与决策？

---

## 5. 工具使用说明（Vibe Coding）

* 你可以自由使用：

  * **AI 工具**：Cursor、ChatGPT、GitHub Copilot 等；
  * **可视化 / 画图工具**：Excalidraw、draw.io、Miro、白板等；
  * **SQL 环境**：BigQuery 控制台、本地工具、在线 SQL 编辑器等。
* 我们不会以「是否完全不用 AI」或「是否记住所有语法细节」作为评价标准；
* 更关注你：

  * 如何把业务问题清晰地描述给工具；
  * 如何对 AI 给出的 SQL / 结果进行审查和修正；
  * 如何保持整体思路清晰、结构化。

你不需要提前准备完整答案，只需熟悉题目和场景，现场我们会一起从产品问题出发，逐步构建指标、查询和分析。