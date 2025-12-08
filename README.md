# 现场案例题：候选人面试助手数据分析 / BI（Vibe Coding）

> 本题将在 **45–60 分钟技术面试** 中以「vibe coding」形式完成，**不是离线作业**。  
> 你可以在现场自由使用 **任何工具**（如 BigQuery、VS Code、Cursor、ChatGPT、浏览器搜索等），以你最顺手的方式完成任务。

---

## 1. 面试形式说明（Vibe Coding）

- 面试将以 **共享屏幕 + 即时思考 / 动手** 的形式进行；
- 你可以：
  - 在 BigQuery / 任何 SQL 环境中实际写 SQL 并运行；
  - 或在编辑器 / 文本工具中写出接近 BigQuery 的 SQL 伪代码；
  - 全程使用 **Cursor / ChatGPT 等 AI 工具** 辅助都是允许且鼓励的；
- 我们更关注：
  - 你如何理解业务问题并拆解为数据问题；
  - 你如何设计指标与查询；
  - 你如何解释结果与限制；
  - 以及你如何与 AI 工具配合，而不是是否「纯手写」。

如对题目或字段含义有疑问，可以在面试现场直接提问并做合理假设。

---

## 2. 产品背景：候选人「面试助手」

我们有一个 **「面试助手」产品，是面试者（候选人）使用的**：

- 候选人在 Zoom / Google Meet 等视频面试中，通过浏览器插件或网页接入；
- 系统实时抓取面试中的音频流 → 做语音转文本（ASR）；
- 再调用大语言模型（LLM），为候选人生成 **实时回答建议**（如要点、结构化回答框架、可能的 follow-up）；
- 这些建议在候选人浏览器端显示，用于辅助他在面试中更好地组织答案。

我们已经积累了一部分使用日志，希望用数据回答：

1. 候选人是否愿意使用这个面试助手？使用情况如何？  
2. 不同岗位 / 级别 / 区域，对助手的依赖程度是否不同？  
3. 使用助手与面试结果 / 候选人体验之间，是否存在有意义的相关性？

---

## 3. 数据表结构（BigQuery 风格，简化版）

你可以假设已经存在以下两张主表。

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
> * 如无特别说明，你可以基于以上字段做 **合理业务假设** 并在面试中说明。

---

## 4. 任务说明

面试中我们会围绕以下任务展开，不要求全部完成，会根据时间深度适当调整。
你可以任选熟悉的方式（直接写 SQL、先画指标草稿、再用工具补全等）来完成。

---

### 任务 1：核心指标设计（业务 & 数据理解）

请基于上述两张表，设计 **3–4 个你认为最关键的核心指标**，帮助我们理解：

* 面试助手是否被使用、使用程度如何（Adoption & Usage）；
* 使用面试助手与 **面试结果 / 候选人反馈** 是否存在可见相关性。

对每个指标，请简要说明：

1. **指标定义**：使用哪些字段，如何计算（例如：按日 / 周 / job_level 聚合等）；
2. **业务含义**：该指标回答了什么问题，为什么重要。

你可以自由命名指标，例如：

* 助手启用率（Assistant Adoption Rate）；
* 每场平均展示建议数 / 被实际使用的建议数；
* 使用助手 vs 未使用助手的通过率差异；
* 使用助手 vs 未使用助手的平均候选人评分差异；
* 响应延迟相关指标（平均 / P95 `latency_ms`）等。

---

### 任务 2：SQL / BigQuery 查询（实战）

请使用接近 BigQuery 的 SQL（可以在任意支持 SQL 的环境中，也可以写伪代码），完成以下问题。
如你使用 AI 工具生成部分 SQL，请在面试中说明你的修改与校验方式。

#### 任务 2.1：按周 + job_level 的助手启用率

目标：分析最近 8 周面试助手的启用情况。

请编写 SQL：

* 限制时间范围为「最近 8 周」；
* 按 **每周** + `job_level` 进行聚合；
* 输出每个（周，job_level）下：

  * 总面试场次；
  * `assistant_enabled = TRUE` 的场次；
  * 启用率（assistant_enabled = TRUE 的场次 / 总场次）。

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

请基于以下步骤设计查询（可以在一条 SQL 中完成，也可以分步说明）：

1. 在 `assistant_suggestions` 中按 `session_id` 聚合，计算：

   * `used_suggestions` = 每场面试中 `was_sent = TRUE` 的建议条数；
2. 将上一步结果与 `interview_sessions` 按 `session_id` 关联；
3. 仅保留 `outcome IN ('pass', 'fail')` 的记录，忽略 `no_show`；
4. 按 `used_suggestions` 将面试分为三档（可使用 `CASE WHEN`）：

   * `0_used`：`used_suggestions = 0`
   * `1_3_used`：`1 <= used_suggestions <= 3`
   * `gt_3_used`：`used_suggestions > 3`
5. 对每个档位统计：

   * 面试总数；
   * 通过场次数；
   * 通过率（通过场次 / 总场次）；
   * 平均候选人评分（`candidate_feedback_score`）。

推荐输出字段：

```text
usage_bucket                 -- '0_used' / '1_3_used' / 'gt_3_used'
sessions_count
pass_sessions
pass_rate
avg_candidate_feedback_score
```

面试过程中，我们会进一步与你讨论：

* 如果看到某一档位通过率或评分明显更高 / 更低，你会如何向产品 / 业务解释？
* 这种相关性中可能有哪些偏差或混淆因素（例如候选人自选择使用助手、岗位差异等）？

---

### 任务 3（可选，若时间允许）：进一步利用 LLM 的分析思路

此任务偏概念思考，不强制要求写代码，可以口头说明。

> 假设我们还存在一张更细粒度的转写表 `conversation_turns`，记录面试中每一轮发言的文本（包括 interviewer 与 candidate），你可以认为其字段大致为：
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

   * 打哪些标签（字段）？
   * 采用怎样的处理流程（批处理 / 流式、调用方式、结果格式）？
2. 在数据表设计上，你会如何落地这些标签？

   * 新增字段还是单独建标签表？大致 schema 是什么样？
3. 你会如何验证这些标签的质量，确保可以用于 BI 分析与决策？

---

## 5. 评估关注点（供参考）

你不需要刻意迎合，只需自然发挥。我们大致会从以下维度观察：

* **业务与数据理解**：能否将产品问题抽象成合理指标和数据问题；
* **SQL / BigQuery 能力**：查询是否清晰、可读，能否处理 join / group by / 条件过滤等典型场景；
* **分析与解释**：能否用自然语言解释指标与结果，并意识到数据局限与潜在偏差；
* **AI 工具使用**：如何利用 Cursor / ChatGPT 等工具加速工作，同时保持校验与自主判断；
* **沟通与协作方式**：遇到不确定信息时是否会主动澄清并做合理假设。

你可以根据自己的习惯准备，比如熟悉的 SQL 环境、偏好的编辑器或 AI 工具等；现场可以自由调整使用方式。
