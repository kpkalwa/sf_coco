# Snowflake Cortex: A 6-Week Learning Plan

**Tailored for:** Someone with solid Snowflake/SQL skills, new to AI/ML
**Weekly commitment:** 5–7 hours
**Goal:** Go from zero to shipping a working Cortex Agent that combines RAG, text-to-SQL, and LLM reasoning.

---

## How we'll work together

Each week follows the same rhythm so you build habits, not just knowledge:

1. **Concept primer (45–60 min)** — read docs, understand the *why*
2. **Drills (60–90 min)** — 4–6 short hands-on exercises, ~10–15 min each, with solutions you can reveal
3. **Guided lab (2–3 hrs)** — run an official Snowflake-Labs quickstart, typing (not copy-pasting) the code
4. **Stretch task (1–2 hrs)** — modify the lab with your own data or a twist I'll suggest
5. **Reflection (30 min)** — write 3–5 bullets in a running journal: what clicked, what didn't, one question

**How to use the drills:** Read the problem, open a worksheet, take a real shot before peeking at the solution. Getting stuck for 5 minutes and then seeing the answer is worth three hours of passive reading. Click the `▶ Solution` line in any markdown viewer (Snowsight Notebook, VS Code, GitHub) to expand.

Keep a Snowflake Notebook called `CORTEX_JOURNAL` for all of this. You'll thank yourself in week 5.

---

## Week 0 — Setup (one evening, ~1 hour)

Do this before week 1 so you're not fighting environment issues later.

- Create a Snowflake trial account in a region that supports Cortex — **AWS us-west-2, us-east-1, or eu-central-1** are safest
- Grant yourself the `CORTEX_USER` database role ([privileges docs](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql))
- Create a dedicated database `CORTEX_LAB` with schemas `RAW`, `CURATED`, `APPS`
- Create a small XS warehouse `CORTEX_WH` with auto-suspend 60s
- Install the [Snowflake CLI](https://docs.snowflake.com/en/developer-guide/snowflake-cli/index) and verify `snow connection test` works
- Clone these two repos locally:
  - [`Snowflake-Labs/sf-samples`](https://github.com/Snowflake-Labs/sf-samples)
  - [`Snowflake-Labs/sfguides`](https://github.com/Snowflake-Labs/sfguides)

**Checkpoint:** Run `SELECT SNOWFLAKE.CORTEX.COMPLETE('claude-3-5-sonnet', 'Say hi');` in a worksheet. If you get a response, you're ready.

**Reference docs:** [Snowflake AI and ML overview](https://docs.snowflake.com/en/guides-overview-ai-features)

---

## Week 1 — Cortex AISQL & LLM Functions

**The mental model:** Cortex LLM functions are just SQL functions. `AI_COMPLETE`, `AI_SUMMARIZE`, `AI_CLASSIFY`, `AI_EXTRACT`, `AI_TRANSLATE`, `AI_EMBED` — each one takes text in and returns text, JSON, or a vector. No Python, no API keys, no separate service.

### Concept primer
- [Snowflake Cortex AISQL overview](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql)
- [`AI_COMPLETE` / `COMPLETE` reference](https://docs.snowflake.com/en/sql-reference/functions/complete-snowflake-cortex) — especially model options and `response_format`
- [Cortex AISQL with images / multimodal](https://docs.snowflake.com/user-guide/snowflake-cortex/ai-images)
- [`TRY_COMPLETE`](https://docs.snowflake.com/en/sql-reference/functions/try_complete-snowflake-cortex) for batch work

### Drills

**Drill 1.1 — First completion.** Write a `COMPLETE` call that takes the string "Explain a CTE in one sentence" and returns a response from two different models. Compare.

<details><summary>▶ Solution</summary>

```sql
SELECT 'claude' AS model, SNOWFLAKE.CORTEX.COMPLETE('claude-3-5-sonnet', 'Explain a CTE in one sentence') AS response
UNION ALL
SELECT 'llama', SNOWFLAKE.CORTEX.COMPLETE('llama3.1-70b', 'Explain a CTE in one sentence');
```
Note the length, style, and latency difference. Smaller models are often good enough.
</details>

**Drill 1.2 — Sentiment over a table.** Create a table of 5 sample product reviews. Add a column that scores each one with `SNOWFLAKE.CORTEX.SENTIMENT`.

<details><summary>▶ Solution</summary>

```sql
CREATE OR REPLACE TABLE reviews (id INT, text STRING);
INSERT INTO reviews VALUES
  (1, 'Absolutely love this — best purchase of the year'),
  (2, 'It broke after two days. Terrible quality.'),
  (3, 'It''s fine, nothing special.'),
  (4, 'Shipping was fast but the product smells weird.'),
  (5, 'Amazing customer service, they replaced it instantly.');

SELECT id, text, SNOWFLAKE.CORTEX.SENTIMENT(text) AS score
FROM reviews;
```
Sentiment returns a float from -1 to 1. Notice how mixed reviews (#4) land near zero.
</details>

**Drill 1.3 — Structured extraction with JSON mode.** Use `AI_COMPLETE` with `response_format` to extract `{product, issue, severity}` from the review: *"My blender exploded and covered my kitchen in smoothie — total disaster."*

<details><summary>▶ Solution</summary>

```sql
SELECT AI_COMPLETE(
  'claude-3-5-sonnet',
  'Extract product, issue, and severity (low/med/high) from: My blender exploded and covered my kitchen in smoothie — total disaster.',
  {
    'response_format': {
      'type': 'json',
      'schema': {
        'type': 'object',
        'properties': {
          'product': {'type': 'string'},
          'issue': {'type': 'string'},
          'severity': {'type': 'string', 'enum': ['low','med','high']}
        },
        'required': ['product','issue','severity']
      }
    }
  }
) AS result;
```
Structured outputs are the single most useful Cortex feature for production pipelines.
</details>

**Drill 1.4 — Classification at scale.** Use `AI_CLASSIFY` to label each review from Drill 1.2 as one of: `quality`, `shipping`, `service`, `other`.

<details><summary>▶ Solution</summary>

```sql
SELECT id, text,
  AI_CLASSIFY(text, ['quality','shipping','service','other']) AS category
FROM reviews;
```
`AI_CLASSIFY` is faster and cheaper than `AI_COMPLETE` for this task. Use the specialized function whenever one exists.
</details>

**Drill 1.5 — Token counting before you spend.** Estimate how many tokens you'd send if you ran `AI_SUMMARIZE` over all 5 reviews. Then compute the total for 10,000 reviews assuming similar length.

<details><summary>▶ Solution</summary>

```sql
SELECT SUM(AI_COUNT_TOKENS('snowflake-arctic', text)) AS total_tokens,
       SUM(AI_COUNT_TOKENS('snowflake-arctic', text)) * 2000 AS projected_10k
FROM reviews;
```
Always do this math before running a Cortex function over a big table. `AI_COUNT_TOKENS` is free to call.
</details>

**Drill 1.6 — Translate and summarize in one view.** Build a view `reviews_enriched` with columns: `id`, `text`, `summary` (1 sentence), `sentiment`, `spanish_translation`.

<details><summary>▶ Solution</summary>

```sql
CREATE OR REPLACE VIEW reviews_enriched AS
SELECT
  id,
  text,
  AI_SUMMARIZE(text) AS summary,
  SNOWFLAKE.CORTEX.SENTIMENT(text) AS sentiment,
  AI_TRANSLATE(text, 'en', 'es') AS spanish_translation
FROM reviews;

SELECT * FROM reviews_enriched;
```
This pattern — one view that decorates raw text with AI-derived columns — is the backbone of most production Cortex work.
</details>

### Guided lab
- **Quickstart:** [Getting Started with Snowflake Arctic and Snowflake Cortex](https://github.com/Snowflake-Labs/sfguide-getting-started-with-snowflake-arctic-and-snowflake-cortex)

### Stretch task
Pick a text dataset you care about (your exported emails, a Kaggle reviews CSV, scraped forum posts). Build a single view that produces, per row: a 1-sentence summary, a sentiment score, an extracted topic list as JSON, and a translation. One view, one warehouse, no Python.

### Reflection prompts
- Which function surprised you? Why?
- Where did token limits or latency bite?
- What would you never use an LLM function for?

---

## Week 2 — Embeddings & Cortex Search (RAG foundations)

**The mental model:** Cortex Search is a managed service that takes a table of text, embeds it, stores the vectors, keeps them fresh, and gives you a hybrid (vector + keyword) search endpoint. You do not manage a vector database.

### Concept primer
- [Cortex Search overview](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/cortex-search-overview)
- [`CREATE CORTEX SEARCH SERVICE` reference](https://docs.snowflake.com/en/sql-reference/sql/create-cortex-search)
- [Querying a Cortex Search Service](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/query-cortex-search-service)
- [March 2026 updates](https://docs.snowflake.com/en/release-notes/2026/other/2026-03-12-recent-cortex-search) — multi-index and custom embeddings

### Drills

**Drill 2.1 — Generate an embedding.** Use `AI_EMBED` to embed the string "snowflake is a cloud data platform" with a 768-dim model. Inspect the output.

<details><summary>▶ Solution</summary>

```sql
SELECT AI_EMBED('snowflake-arctic-embed-m-v1.5', 'snowflake is a cloud data platform') AS vec;
-- returns a VECTOR(FLOAT, 768)
```
Embeddings are just arrays of floats. Similarity is cosine distance between two vectors.
</details>

**Drill 2.2 — Find most similar.** Embed three sentences about cooking, programming, and astronomy. Then embed the query "python loops" and find the closest match using `VECTOR_COSINE_SIMILARITY`.

<details><summary>▶ Solution</summary>

```sql
WITH corpus AS (
  SELECT 'Bake at 350F for 20 minutes' AS text,
         AI_EMBED('snowflake-arctic-embed-m-v1.5', 'Bake at 350F for 20 minutes') AS vec
  UNION ALL SELECT 'For loops iterate over a sequence',
         AI_EMBED('snowflake-arctic-embed-m-v1.5', 'For loops iterate over a sequence')
  UNION ALL SELECT 'Jupiter has 95 moons',
         AI_EMBED('snowflake-arctic-embed-m-v1.5', 'Jupiter has 95 moons')
),
query AS (SELECT AI_EMBED('snowflake-arctic-embed-m-v1.5', 'python loops') AS qvec)
SELECT corpus.text, VECTOR_COSINE_SIMILARITY(corpus.vec, query.qvec) AS similarity
FROM corpus, query
ORDER BY similarity DESC;
```
The programming sentence should win. This is the core of vector search in ~15 lines.
</details>

**Drill 2.3 — Create a search service.** Create a small table `faqs(id, question, answer)` with 5 rows about Snowflake basics, then create a Cortex Search service over the `question` column.

<details><summary>▶ Solution</summary>

```sql
CREATE OR REPLACE TABLE faqs (id INT, question STRING, answer STRING);
INSERT INTO faqs VALUES
  (1, 'What is a warehouse?', 'Compute resources for queries.'),
  (2, 'How does time travel work?', 'Query historical data up to 90 days back.'),
  (3, 'What is a stage?', 'A location for loading or unloading files.'),
  (4, 'What is zero-copy cloning?', 'Instant metadata-only copy of any object.'),
  (5, 'How is billing calculated?', 'Per-second compute + per-TB storage.');

CREATE OR REPLACE CORTEX SEARCH SERVICE faq_search
  ON question
  ATTRIBUTES id
  WAREHOUSE = CORTEX_WH
  TARGET_LAG = '1 hour'
  AS SELECT id, question, answer FROM faqs;
```
`TARGET_LAG` controls refresh frequency — shorter = fresher but more expensive.
</details>

**Drill 2.4 — Query your service.** Use `SEARCH_PREVIEW` to find the top 2 FAQs matching "compute engine".

<details><summary>▶ Solution</summary>

```sql
SELECT PARSE_JSON(SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
  'CORTEX_LAB.CURATED.faq_search',
  '{"query": "compute engine", "columns":["question","answer"], "limit":2}'
))['results'] AS results;
```
Note it matches "warehouse" even though that word isn't in the query — that's semantic search working.
</details>

**Drill 2.5 — RAG in one query.** Retrieve the top FAQ and feed it into `AI_COMPLETE` to answer "how do I get historical data back?" — in a single SQL statement.

<details><summary>▶ Solution</summary>

```sql
WITH hits AS (
  SELECT PARSE_JSON(SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
    'CORTEX_LAB.CURATED.faq_search',
    '{"query": "how do I get historical data back", "columns":["question","answer"], "limit":1}'
  ))['results'][0] AS top
)
SELECT AI_COMPLETE(
  'claude-3-5-sonnet',
  'Using ONLY this context, answer the question: ' ||
  'Context: ' || top:answer::STRING ||
  ' Question: how do I get historical data back?'
) AS answer
FROM hits;
```
This is RAG in its simplest form. Everything from here is just refinement (chunking, reranking, filtering).
</details>

**Drill 2.6 — Filter by metadata.** Add a `category` column to the FAQ table (values: `compute`, `storage`, `features`) and query the search service restricted to `category = 'compute'`.

<details><summary>▶ Solution</summary>

```sql
ALTER TABLE faqs ADD COLUMN category STRING;
UPDATE faqs SET category = CASE id
  WHEN 1 THEN 'compute' WHEN 2 THEN 'features' WHEN 3 THEN 'storage'
  WHEN 4 THEN 'features' WHEN 5 THEN 'compute' END;

-- Recreate service to include the attribute
CREATE OR REPLACE CORTEX SEARCH SERVICE faq_search
  ON question ATTRIBUTES id, category
  WAREHOUSE = CORTEX_WH TARGET_LAG = '1 hour'
  AS SELECT id, question, answer, category FROM faqs;

-- Query with filter
SELECT PARSE_JSON(SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
  'CORTEX_LAB.CURATED.faq_search',
  '{"query": "engine", "columns":["question"], "filter":{"@eq":{"category":"compute"}}, "limit":5}'
));
```
Metadata filters are how you get multi-tenant search, permissioned search, and time-bounded search.
</details>

### Guided lab
- [Cortex Search tutorials](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/overview-tutorials) — tutorials 1 and 2
- Sample code: [`Snowflake-Labs/cortex-search`](https://github.com/Snowflake-Labs/cortex-search)

### Stretch task
Build a minimal RAG pipeline in a Snowflake Notebook: user question → Cortex Search top 5 chunks → stuff into a prompt → `AI_COMPLETE` generates grounded answer. Test with 10 questions and note which ones fail and why.

### Reflection prompts
- What's the failure mode when the answer isn't in your corpus?
- How would you evaluate retrieval quality without a human reviewer?

---

## Week 3 — Cortex Analyst (text-to-SQL)

**The mental model:** Cortex Analyst is *not* "an LLM writing SQL against your tables." It's an LLM writing SQL against a **semantic model** — a YAML file defining business meaning of tables, columns, metrics, and synonyms. The YAML quality determines answer quality.

### Concept primer
- [Cortex Analyst overview](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst)
- [Semantic model specification](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst/semantic-model-spec) — **read end-to-end, don't skim**
- [Cortex Analyst REST API](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst/rest-api)

### Drills

**Drill 3.1 — Minimal YAML.** Create a tiny `sales` table with `date`, `product`, `region`, `revenue`. Write a 15-line semantic YAML naming the table, labeling each column with a description, and defining one metric `total_revenue`.

<details><summary>▶ Solution</summary>

```sql
CREATE OR REPLACE TABLE sales (
  sale_date DATE, product STRING, region STRING, revenue NUMBER(10,2)
);
INSERT INTO sales VALUES
  ('2026-01-15','Widget','NA',100),('2026-01-16','Gadget','EU',250),
  ('2026-02-01','Widget','EU',175),('2026-02-15','Gadget','NA',300);
```
```yaml
name: sales_model
description: Simple sales data for revenue analysis
tables:
  - name: sales
    base_table:
      database: CORTEX_LAB
      schema: CURATED
      table: SALES
    dimensions:
      - name: product
        expr: product
        data_type: STRING
        description: Product name sold
      - name: region
        expr: region
        data_type: STRING
        description: Sales geography
    time_dimensions:
      - name: sale_date
        expr: sale_date
        data_type: DATE
        description: Date of the sale
    measures:
      - name: total_revenue
        expr: revenue
        data_type: NUMBER
        description: Sum of sale revenue
        default_aggregation: sum
```
Upload to a stage, then reference it when calling the Analyst REST API.
</details>

**Drill 3.2 — Add synonyms.** Extend the YAML so "sales" and "income" both work as synonyms for `total_revenue`, and "geo" works for `region`.

<details><summary>▶ Solution</summary>

```yaml
dimensions:
  - name: region
    expr: region
    data_type: STRING
    synonyms: ['geo','territory','area']
    description: Sales geography
measures:
  - name: total_revenue
    expr: revenue
    data_type: NUMBER
    synonyms: ['sales','income','topline']
    default_aggregation: sum
```
Synonyms are the single highest-leverage field in a semantic model. Add them liberally.
</details>

**Drill 3.3 — Call the API from SQL.** Use `SYSTEM$SEND_REQUEST` or the Cortex Analyst REST endpoint via Python to ask "What was total revenue by region?" and print the SQL it generated.

<details><summary>▶ Solution (Python in Snowflake Notebook)</summary>

```python
import _snowflake, json
resp = _snowflake.send_snow_api_request(
  "POST", "/api/v2/cortex/analyst/message", {}, {},
  {"messages":[{"role":"user","content":[{"type":"text","text":"What was total revenue by region?"}]}],
   "semantic_model_file":"@CORTEX_LAB.CURATED.MODELS/sales_model.yaml"},
  {}, 30000
)
body = json.loads(resp["content"])
for item in body["message"]["content"]:
    print(item.get("type"), item.get("text") or item.get("statement"))
```
You'll see the natural-language answer, the generated SQL, and any suggestions for ambiguous questions.
</details>

**Drill 3.4 — Add a verified query.** Add a `verified_queries` section to the YAML with one example: "top product by revenue" mapped to the correct SQL. Ask the question and confirm the model uses your verified answer.

<details><summary>▶ Solution</summary>

```yaml
verified_queries:
  - name: top_product_by_revenue
    question: What is the top product by revenue?
    sql: |
      SELECT product, SUM(revenue) AS total_revenue
      FROM CORTEX_LAB.CURATED.SALES
      GROUP BY product
      ORDER BY total_revenue DESC
      LIMIT 1
```
Verified queries are Cortex Analyst's most powerful accuracy lever. Every production semantic model should have 10-20.
</details>

**Drill 3.5 — Break it on purpose.** Temporarily remove the `description` field from `product`. Ask "How much did widgets sell?" three times. Note what changes.

<details><summary>▶ Solution</summary>

Without the description, the model has to guess that "widgets" refers to `product = 'Widget'`. Sometimes it works, sometimes it filters wrong, sometimes it asks for clarification. This illustrates why descriptions are not optional — they're the model's only window into business meaning. Restore the description when done.
</details>

**Drill 3.6 — Custom instructions.** Add a top-level `custom_instructions` field saying "Always exclude test regions (TEST, DEV) from all revenue calculations." Verify a generated SQL statement includes the exclusion.

<details><summary>▶ Solution</summary>

```yaml
custom_instructions: |
  Always exclude test regions (TEST, DEV) from revenue calculations.
  When users ask about "sales," assume they mean total_revenue unless specified.
```
Ask "What was total revenue?" and confirm the SQL includes `WHERE region NOT IN ('TEST','DEV')`. Custom instructions are how you encode business rules the model would never guess from schema alone.
</details>

### Guided lab
- Hosted quickstart: [Getting Started with Cortex Analyst: Augment BI with AI](https://quickstarts.snowflake.com/guide/getting_started_with_cortex_analyst/index.html)
- Companion code: [`sfguide-getting-started-with-cortex-analyst-in-snowflake`](https://github.com/Snowflake-Labs/sfguide-getting-started-with-cortex-analyst-in-snowflake)

### Stretch task
Take the semantic model you built in the lab and **deliberately break it** in three ways (remove a synonym, misname a metric, delete a column description). For each, ask the same question five times and log where it goes wrong. Then fix each break one at a time and watch accuracy recover. This is the single most valuable exercise of the whole plan.

### Reflection prompts
- Which YAML field mattered most for accuracy?
- What's the smallest change that produced the biggest quality improvement?

---

## Week 4 — Cortex Agents (putting it all together)

**The mental model:** A Cortex Agent is an orchestrator. It receives a question, picks whether to route to Cortex Search (unstructured), Cortex Analyst (structured), or an LLM, calls one or more, and composes a final response. You configure once; it picks per turn.

### Concept primer
- [Cortex Agents overview](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents)
- [Cortex Agents REST API](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-rest-api)
- [Cortex Agents Run API](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-run) — understand the event stream
- [Configure and interact with Agents in Snowsight](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-manage)

### Drills

**Drill 4.1 — Create an agent in Snowsight.** Using the UI, create an agent `DEMO_AGENT` with just the Cortex Analyst tool pointing at your Week 3 semantic model. Chat with it.

<details><summary>▶ Solution</summary>

Navigate to **AI & ML → Agents → Create Agent**. Name: `DEMO_AGENT`. Under **Tools**, click `+ Add` next to Cortex Analyst, select your semantic view, pick `CORTEX_WH` as warehouse, add a description ("Answers revenue questions"). Save. Open the chat panel and ask "total revenue by region". The agent should call Analyst and return both a table and a prose answer.
</details>

**Drill 4.2 — Add a search tool.** Add your Week 2 `faq_search` service as a second tool. Ask a question that should route to Search (e.g., "what is time travel?") and one that should route to Analyst ("total revenue in EU").

<details><summary>▶ Solution</summary>

Back in the agent config, `+ Add` next to Cortex Search. Select `faq_search`, set ID column to `id` and title column to `question`, add a description like "Snowflake product FAQs". Save. Ask both questions and look at the trace panel — you should see different tool calls for each.
</details>

**Drill 4.3 — Read the event stream.** Call the Agent Run API from Python and print every event type in order for one question. Notice `thinking`, `tool_use`, `content_delta`, and `response` events.

<details><summary>▶ Solution</summary>

```python
import _snowflake, json
resp = _snowflake.send_snow_api_request(
  "POST", "/api/v2/cortex/agent:run", {}, {},
  {"agent_name":"CORTEX_LAB.APPS.DEMO_AGENT",
   "messages":[{"role":"user","content":[{"type":"text","text":"total revenue by region"}]}]},
  {}, 60000
)
for line in resp["content"].splitlines():
    if line.startswith("data:"):
        evt = json.loads(line[5:])
        print(evt.get("event") or evt.get("type"), "-",
              (evt.get("data") or evt.get("delta") or {}).get("type",""))
```
The event stream is how production UIs show "thinking..." indicators and citations.
</details>

**Drill 4.4 — Force a bad question.** Ask your agent "Who won the Super Bowl last year?" and observe the response. Does it decline? Hallucinate? Route somewhere weird?

<details><summary>▶ Solution</summary>

Most agents will either answer from general LLM knowledge (risky — could be wrong) or politely decline because the question doesn't match any tool. The answer depends on the agent's **response instructions** — add a line like "Only answer questions about sales data or Snowflake features. For anything else, politely decline." Re-test and see the behavior change.
</details>

**Drill 4.5 — Add a custom SQL tool.** Create a UDF `get_fx_rate(currency STRING)` that returns a fake exchange rate. Register it as a custom tool on the agent. Ask "convert last month's EU revenue to USD."

<details><summary>▶ Solution</summary>

```sql
CREATE OR REPLACE FUNCTION get_fx_rate(currency STRING)
RETURNS NUMBER(10,4)
AS $$
  CASE WHEN currency='EUR' THEN 1.08 WHEN currency='GBP' THEN 1.27 ELSE 1.00 END
$$;
```
In agent config, add a custom **function** tool pointing at `CORTEX_LAB.CURATED.GET_FX_RATE` with input schema `{currency: string}` and a clear description. The agent should call Analyst for the revenue, then your UDF for the rate, then compose the answer. This is the pattern for adding any custom capability.
</details>

**Drill 4.6 — Read the trace.** Turn on request logging. Ask the same question three times with slightly different phrasings ("EU revenue", "how much did Europe sell", "sales in the EU region"). Inspect which tool each call used and whether results varied.

<details><summary>▶ Solution</summary>

```sql
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_ANALYST_USAGE_HISTORY
WHERE query_timestamp > DATEADD(hour,-1,CURRENT_TIMESTAMP())
ORDER BY query_timestamp DESC;
```
Compare the generated SQL for each phrasing. Identical SQL = robust semantic model. Different SQL = add synonyms or verified queries.
</details>

### Guided lab
- Hosted quickstart: [Getting Started with Cortex Agents](https://quickstarts.snowflake.com/guide/getting_started_with_cortex_agents/index.html)
- Companion code: [`sfguide-getting-started-with-cortex-agents`](https://github.com/Snowflake-Labs/sfguide-getting-started-with-cortex-agents)
- Optional: [React / Next.js front-end version](https://github.com/Snowflake-Labs/sfguide-getting-started-with-cortex-agents-and-react)

### Stretch task
Add a third tool to your agent — either a custom SQL UDF or a second Cortex Search service over a different corpus. Design 10 questions that should route to each of the three tools. Verify routing correctness and screenshot the tool-call traces.

### Reflection prompts
- How does the agent decide which tool to call? What signal dominated?
- Where would a human reviewer still catch errors the agent missed?

---

## Week 5 — Evaluation, cost, and LLMOps

### Concept primer
- [Understanding cost for Cortex Search Services](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/cortex-search-costs)
- [Cortex AI Functions cost section](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql)
- [Cortex Guard guardrails](https://docs.snowflake.com/en/sql-reference/functions/complete-snowflake-cortex)

### Drills

**Drill 5.1 — Measure actual cost.** Query `CORTEX_FUNCTIONS_USAGE_HISTORY` for the last 24 hours. Report credits per function.

<details><summary>▶ Solution</summary>

```sql
SELECT function_name, model_name,
       SUM(tokens) AS total_tokens,
       SUM(token_credits) AS credits
FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_FUNCTIONS_USAGE_HISTORY
WHERE start_time > DATEADD(hour,-24,CURRENT_TIMESTAMP())
GROUP BY 1,2
ORDER BY credits DESC;
```
Note: `ACCOUNT_USAGE` has up to 3-hour latency. If data is missing, wait and re-run.
</details>

**Drill 5.2 — Model shootout.** Answer the same 5 sentiment-classification questions with three different models. Compare accuracy and cost.

<details><summary>▶ Solution</summary>

```sql
WITH test AS (
  SELECT 'I love this' AS text, 'positive' AS expected
  UNION ALL SELECT 'This is terrible','negative'
  UNION ALL SELECT 'It is fine','neutral'
  UNION ALL SELECT 'Best ever!!','positive'
  UNION ALL SELECT 'Complete waste of money','negative'
)
SELECT text, expected,
  AI_COMPLETE('claude-3-5-sonnet', 'Classify as positive/negative/neutral. Just the label: ' || text) AS claude,
  AI_COMPLETE('llama3.1-70b', 'Classify as positive/negative/neutral. Just the label: ' || text) AS llama,
  AI_COMPLETE('snowflake-arctic', 'Classify as positive/negative/neutral. Just the label: ' || text) AS arctic
FROM test;
```
The cheapest model that hits your accuracy bar is almost never the biggest one. This is *the* central tradeoff in production LLM work.
</details>

**Drill 5.3 — Turn on Cortex Guard.** Call `COMPLETE` with `guardrails: true` on a prompt that could produce unsafe output. Compare the response to one without guardrails.

<details><summary>▶ Solution</summary>

```sql
SELECT SNOWFLAKE.CORTEX.COMPLETE(
  'mistral-large2',
  [{'role':'user','content':'Write an aggressive insult.'}],
  {'guardrails': true}
);
-- returns: "Response filtered by Cortex Guard" when triggered
```
Guard adds token cost but blocks known-harmful outputs. Use for any customer-facing application.
</details>

**Drill 5.4 — Cost projection.** You have 1M customer reviews. Estimate the cost of running `AI_SUMMARIZE` over all of them using Arctic vs Claude Sonnet. Decide which to use.

<details><summary>▶ Solution</summary>

```sql
-- Step 1: sample 100 reviews, average tokens per summary call
SELECT AVG(AI_COUNT_TOKENS('snowflake-arctic', review_text)) AS avg_in,
       100 AS avg_out_estimate  -- summaries are short
FROM my_reviews LIMIT 100;

-- Step 2: multiply by 1M and by credit rate from Snowflake Service Consumption Table
-- Arctic: ~0.22 credits/M tokens. Claude Sonnet: ~6+ credits/M tokens.
-- For summarization specifically, Arctic is usually fine.
```
Always price the cheap model first. If accuracy isn't there, upgrade.
</details>

**Drill 5.5 — Build a mini eval set.** Write 10 questions for your Week 4 agent with known-correct answers. Run them, and manually score pass/fail. Compute accuracy.

<details><summary>▶ Solution</summary>

```sql
CREATE OR REPLACE TABLE eval_set (
  id INT, question STRING, expected_contains STRING, actual_answer STRING, passed BOOLEAN
);
-- Populate manually after running each question through your agent
-- Then:
SELECT COUNT_IF(passed)/COUNT(*)::FLOAT AS accuracy FROM eval_set;
```
Eval is a discipline, not a tool. The habit of maintaining a 10-question regression suite beats any framework.
</details>

**Drill 5.6 — Set a budget alarm.** Create a resource monitor on `CORTEX_WH` that alerts at 80% of a 50-credit weekly budget and suspends at 100%.

<details><summary>▶ Solution</summary>

```sql
CREATE OR REPLACE RESOURCE MONITOR cortex_wh_budget
WITH CREDIT_QUOTA = 50
     FREQUENCY = WEEKLY
     START_TIMESTAMP = IMMEDIATELY
     TRIGGERS
       ON 80 PERCENT DO NOTIFY
       ON 100 PERCENT DO SUSPEND;

ALTER WAREHOUSE CORTEX_WH SET RESOURCE_MONITOR = cortex_wh_budget;
```
Note: this only covers warehouse compute, not Cortex token spend. For Cortex specifically, use `BUDGETS` in Snowsight.
</details>

### Guided lab (pick one or both)
- **Option A:** [LLMOps with Cortex and TruLens](https://github.com/Snowflake-Labs/sfguide-getting-started-with-llmops-using-snowflake-cortex-and-trulens)
- **Option B:** [Cortex Agent Evaluations](https://www.snowflake.com/en/developers/guides/getting-started-with-cortex-agent-evaluations/)

### Stretch task
Build a cost dashboard: query `CORTEX_FUNCTIONS_USAGE_HISTORY` and visualize credit burn per function, per model, per day. Set a weekly budget alert. Future-you will be grateful.

### Reflection prompts
- What's the cheapest model that still meets your quality bar? (Almost never the biggest.)
- Which eval metric best caught your agent's real-world failures?

---

## Week 6 — Ship something small

No new concepts. Build one end-to-end thing and show it to someone.

Pick ONE, scoped to 5–7 hours:

- **A:** Slack-style Q&A bot over your team's internal docs using Cortex Search + Agent
- **B:** "Chat with your warehouse" Streamlit app with a proper semantic model over a dataset you know well
- **C:** Document-processing pipeline using [`AI_PARSE_DOCUMENT` / `AI_EXTRACT`](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql) turning a stage of PDFs into a structured table

### Drills (pre-project warmup)

**Drill 6.1 — Hello Streamlit in Snowflake.** Create a Streamlit in Snowflake app with one text input and one button that calls `AI_COMPLETE` and displays the result.

<details><summary>▶ Solution</summary>

```python
import streamlit as st
from snowflake.snowpark.context import get_active_session
session = get_active_session()

st.title("Cortex chat")
prompt = st.text_input("Ask anything:")
if st.button("Send") and prompt:
    result = session.sql(
      "SELECT AI_COMPLETE('claude-3-5-sonnet', ?)",
      params=[prompt]
    ).collect()[0][0]
    st.write(result)
```
In Snowsight: **Projects → Streamlit → + Streamlit App**, paste, run. You just shipped.
</details>

**Drill 6.2 — Parse a PDF.** Upload a PDF to a stage and extract its text with `AI_PARSE_DOCUMENT`.

<details><summary>▶ Solution</summary>

```sql
CREATE OR REPLACE STAGE pdfs ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE') DIRECTORY = (ENABLE = TRUE);
-- Upload a PDF via Snowsight UI or PUT command, then:
SELECT AI_PARSE_DOCUMENT(TO_FILE('@pdfs','my_doc.pdf'), {'mode':'LAYOUT'}) AS parsed;
```
`LAYOUT` mode preserves structure (headings, tables). `OCR` mode handles scanned docs. The output is JSON you can query with `:` paths.
</details>

**Drill 6.3 — Chain it all.** One query: parse a PDF → extract structured fields with `AI_EXTRACT` → insert into a table.

<details><summary>▶ Solution</summary>

```sql
CREATE OR REPLACE TABLE doc_facts (filename STRING, title STRING, date_mentioned STRING, summary STRING);

INSERT INTO doc_facts
SELECT
  'my_doc.pdf' AS filename,
  extracted:title::STRING,
  extracted:date::STRING,
  extracted:summary::STRING
FROM (
  SELECT AI_EXTRACT(
    AI_PARSE_DOCUMENT(TO_FILE('@pdfs','my_doc.pdf'), {'mode':'LAYOUT'}):content::STRING,
    ['title','date','summary']
  ) AS extracted
);
```
This pattern — parse + extract + insert — is the whole "document intelligence" use case in 10 lines.
</details>

### Deliverables
- Working app in [Streamlit in Snowflake](https://docs.snowflake.com/en/developer-guide/streamlit/about-streamlit)
- README with architecture, cost per 1000 queries, and known failure modes
- 10-question eval set with pass/fail, run before and after one round of improvements

### Final reflection
Go through your `CORTEX_JOURNAL`. Write one paragraph answering: *If a colleague asked me "should we use Cortex for X?", what are the three questions I'd ask them first?* That paragraph is your real learning outcome.

---

## Reference: the sample projects you'll use

| Week | Quickstart / Repository |
|------|-------------------------|
| 1 | [sfguide-getting-started-with-snowflake-arctic-and-snowflake-cortex](https://github.com/Snowflake-Labs/sfguide-getting-started-with-snowflake-arctic-and-snowflake-cortex) |
| 2 | [Cortex Search tutorials](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/overview-tutorials) + [Snowflake-Labs/cortex-search](https://github.com/Snowflake-Labs/cortex-search) |
| 3 | [Cortex Analyst quickstart](https://quickstarts.snowflake.com/guide/getting_started_with_cortex_analyst/index.html) + [sfguide-getting-started-with-cortex-analyst-in-snowflake](https://github.com/Snowflake-Labs/sfguide-getting-started-with-cortex-analyst-in-snowflake) |
| 4 | [Cortex Agents quickstart](https://quickstarts.snowflake.com/guide/getting_started_with_cortex_agents/index.html) + [sfguide-getting-started-with-cortex-agents](https://github.com/Snowflake-Labs/sfguide-getting-started-with-cortex-agents) |
| 5 | [LLMOps with TruLens](https://github.com/Snowflake-Labs/sfguide-getting-started-with-llmops-using-snowflake-cortex-and-trulens) or [Cortex Agent Evaluations](https://www.snowflake.com/en/developers/guides/getting-started-with-cortex-agent-evaluations/) |
| Bonus | [sfguide-getting-started-with-cortex-agents-and-react](https://github.com/Snowflake-Labs/sfguide-getting-started-with-cortex-agents-and-react) |

## Core doc bookmarks

- [Snowflake AI & ML overview](https://docs.snowflake.com/en/guides-overview-ai-features)
- [Cortex AISQL functions](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql)
- [Cortex Search](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/cortex-search-overview)
- [Cortex Analyst](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst)
- [Cortex Agents](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents)
- [2026 release notes](https://docs.snowflake.com/en/release-notes/new-features-2026)

## Ongoing habits

- Check the [release notes](https://docs.snowflake.com/en/release-notes/new-features-2026) every Monday — Cortex ships features weekly
- When stuck, try [**Cortex Code in Snowsight**](https://docs.snowflake.com/en/release-notes/2026/other/2026-03-09-cortex-code-snowsight-ga) itself (GA as of March 2026)
- Rewrite old notebooks every few weeks; the API surface is still evolving
