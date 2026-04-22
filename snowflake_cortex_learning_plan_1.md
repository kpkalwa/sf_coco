# Snowflake Cortex: A 6-Week Learning Plan

**Tailored for:** Someone with solid Snowflake/SQL skills, new to AI/ML
**Weekly commitment:** 5–7 hours
**Goal:** Go from zero to shipping a working Cortex Agent that combines RAG, text-to-SQL, and LLM reasoning.

---

## How we'll work together

Each week follows the same rhythm so you build habits, not just knowledge:

1. **Concept primer (45–60 min)** — read docs, understand the *why*
2. **Guided lab (2–3 hrs)** — run an official Snowflake-Labs quickstart, typing (not copy-pasting) the code
3. **Stretch task (1–2 hrs)** — modify the lab with your own data or a twist I'll suggest
4. **Reflection (30 min)** — write 3–5 bullets in a running journal: what clicked, what didn't, one question

Keep a Snowflake Notebook called `CORTEX_JOURNAL` for this. You'll thank yourself in week 5.

---

## Week 0 — Setup (one evening, ~1 hour)

Do this before week 1 so you're not fighting environment issues later.

- Create a Snowflake trial account (or use an existing dev account) in a region that supports Cortex — **AWS us-west-2, us-east-1, or eu-central-1** are safest
- Grant yourself the `CORTEX_USER` database role on your user ([privileges docs](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql))
- Create a dedicated database `CORTEX_LAB` with schemas `RAW`, `CURATED`, `APPS`
- Create a small XS warehouse `CORTEX_WH` with auto-suspend 60s
- Install the [Snowflake CLI](https://docs.snowflake.com/en/developer-guide/snowflake-cli/index) and verify `snow connection test` works
- Clone these two repos locally — you'll use them all six weeks:
  - [`Snowflake-Labs/sf-samples`](https://github.com/Snowflake-Labs/sf-samples)
  - [`Snowflake-Labs/sfguides`](https://github.com/Snowflake-Labs/sfguides)

**Checkpoint:** Run `SELECT SNOWFLAKE.CORTEX.COMPLETE('claude-3-5-sonnet', 'Say hi');` in a worksheet. If you get a response, you're ready.

**Reference docs for this week:**
- [Snowflake AI and ML overview](https://docs.snowflake.com/en/guides-overview-ai-features)

---

## Week 1 — Cortex AISQL & LLM Functions

**The mental model to build this week:** Cortex LLM functions are just SQL functions. `AI_COMPLETE`, `AI_SUMMARIZE`, `AI_CLASSIFY`, `AI_EXTRACT`, `AI_TRANSLATE`, `AI_EMBED` — each one takes text in and returns text, JSON, or a vector. No Python, no API keys, no separate service. This is the biggest "aha" moment for SQL-first people, so take time to let it sink in.

### Concept primer
- Read: [Snowflake Cortex AISQL overview](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql) — the page listing all `AI_*` functions
- Read: [`AI_COMPLETE` / `COMPLETE` reference](https://docs.snowflake.com/en/sql-reference/functions/complete-snowflake-cortex) end-to-end — pay attention to model options and the `options` argument (temperature, response_format)
- Read: [Cortex AISQL with images / multimodal](https://docs.snowflake.com/user-guide/snowflake-cortex/ai-images) — it's more capable than most people expect
- Bookmark: [`TRY_COMPLETE`](https://docs.snowflake.com/en/sql-reference/functions/try_complete-snowflake-cortex) — you'll want this for batch work

### Guided lab
- **Quickstart:** [Getting Started with Snowflake Arctic and Snowflake Cortex](https://github.com/Snowflake-Labs/sfguide-getting-started-with-snowflake-arctic-and-snowflake-cortex)
- You'll load a CSV of customer reviews and run sentiment, summarization, and translation across them in pure SQL

### Stretch task
Pick any text dataset you care about — your own emails exported as CSV, a public Kaggle reviews set, scraped forum posts — and build a single SQL view that produces, for each row: a 1-sentence summary, a sentiment score, an extracted list of topics (JSON via `AI_EXTRACT`), and a translation to a second language. One view, one warehouse, no Python.

### Reflection prompts
- Which function surprised you? Why?
- Where did token limits or latency bite?
- What would you never use an LLM function for?

---

## Week 2 — Embeddings & Cortex Search (RAG foundations)

**The mental model:** Cortex Search is a managed service that takes a table of text, embeds it, stores the vectors, keeps them fresh, and gives you a hybrid (vector + keyword) search endpoint. You do not manage a vector database. You write one `CREATE CORTEX SEARCH SERVICE` statement and Snowflake handles the rest.

### Concept primer
- Read: [Cortex Search overview](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/cortex-search-overview)
- Read: [`CREATE CORTEX SEARCH SERVICE` reference](https://docs.snowflake.com/en/sql-reference/sql/create-cortex-search)
- Read: [Querying a Cortex Search Service](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/query-cortex-search-service) — covers SQL, REST, and Python APIs
- Skim: [March 2026 updates](https://docs.snowflake.com/en/release-notes/2026/other/2026-03-12-recent-cortex-search) — multi-index services and custom vector embeddings are both recent and genuinely useful

### Guided lab
- **Official tutorials:** [Cortex Search tutorials index](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/overview-tutorials) — do tutorials 1 and 2 back to back
  - Tutorial 1: simple search app on AirBnb reviews
  - Tutorial 2: chatbot on TED Talk transcripts (RAG end-to-end)
- **Sample code repo:** [`Snowflake-Labs/cortex-search`](https://github.com/Snowflake-Labs/cortex-search) — use examples `05_streamlit_ai_search_app` and `06_streamlit_chatbot_app` as references

### Stretch task
Build a minimal RAG pipeline in a Snowflake Notebook: user question → Cortex Search retrieves top 5 chunks → stuff into a prompt → `AI_COMPLETE` generates grounded answer. About 40 lines of Python. Test with 10 questions and note which ones fail and why (hallucination? retrieval miss? chunking problem?).

### Reflection prompts
- What's the failure mode when the answer isn't in your corpus?
- How would you evaluate retrieval quality without a human reviewer?

---

## Week 3 — Cortex Analyst (text-to-SQL)

**The mental model:** Cortex Analyst is *not* "an LLM writing SQL against your tables." It's an LLM writing SQL against a **semantic model** — a YAML file where you define the business meaning of tables, columns, metrics, and synonyms. The quality of your YAML determines the quality of your answers. This week is 30% LLM, 70% data modeling discipline.

### Concept primer
- Read: [Cortex Analyst overview](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst)
- Read: [Semantic model specification](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst/semantic-model-spec) end-to-end. **This is the week you cannot skim.**
- Read: [Cortex Analyst REST API reference](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst/rest-api)
- Skim: [Semantic views overview](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst/semantic-model-spec) — note the March 2026 preview for relationship paths

### Guided lab
- **Quickstart (hosted):** [Getting Started with Cortex Analyst: Augment BI with AI](https://quickstarts.snowflake.com/guide/getting_started_with_cortex_analyst/index.html)
- **Companion code:** [`Snowflake-Labs/sfguide-getting-started-with-cortex-analyst-in-snowflake`](https://github.com/Snowflake-Labs/sfguide-getting-started-with-cortex-analyst-in-snowflake)
- You'll load revenue data, author a semantic YAML by hand, and query it conversationally via the REST API and a Streamlit front-end

### Stretch task
Take the semantic model you built and **deliberately break it** in three ways: remove a synonym, misname a metric, delete a column description. For each, ask the same question five times and log where the model goes wrong. This is the single most valuable exercise of the whole plan — it teaches you that text-to-SQL accuracy is a modeling problem, not a model problem.

### Reflection prompts
- Which YAML field mattered most for accuracy?
- What's the smallest change that produced the biggest quality improvement?

---

## Week 4 — Cortex Agents (putting it all together)

**The mental model:** A Cortex Agent is an orchestrator. It receives a user question, decides whether to route it to Cortex Search (for unstructured answers), Cortex Analyst (for structured/numeric answers), or a general LLM, calls one or more of them, and composes a final response. You configure the agent once; it picks the right tool per turn.

### Concept primer
- Read: [Cortex Agents overview](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents)
- Read: [Cortex Agents REST API](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-rest-api)
- Read: [Cortex Agents Run API](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-run) — understand the event stream (thinking, tool_use, content delta)
- Read: [Configure and interact with Agents in Snowsight](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-manage)

### Guided lab
- **Quickstart (hosted):** [Getting Started with Cortex Agents](https://quickstarts.snowflake.com/guide/getting_started_with_cortex_agents/index.html)
- **Companion code:** [`Snowflake-Labs/sfguide-getting-started-with-cortex-agents`](https://github.com/Snowflake-Labs/sfguide-getting-started-with-cortex-agents)
- This is the capstone lab: you'll reuse the Cortex Search service from week 2 and the semantic model from week 3, wire them to a single agent, and chat with it via Snowflake Intelligence and a Streamlit app
- **Optional extension:** [Cortex Agents with Next.js / React front-end](https://github.com/Snowflake-Labs/sfguide-getting-started-with-cortex-agents-and-react) if you want a "real app" feel

### Stretch task
Add a third tool to your agent — either a custom SQL function you write yourself or a second Cortex Search service over a different corpus. Ask questions that should route to each of the three tools and verify routing behaves correctly. Screenshot the tool-call traces.

### Reflection prompts
- How does the agent decide which tool to call? What signal dominated?
- Where would a human reviewer still catch errors the agent missed?

---

## Week 5 — Evaluation, cost, and LLMOps

Now that you can build, learn what "good" means.

### Concept primer
- Read: [Understanding cost for Cortex Search Services](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/cortex-search-costs)
- Read: the [Cortex AI Functions cost section](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql) — per-token pricing varies a lot by model
- Read: the [Cortex Guard guardrails section](https://docs.snowflake.com/en/sql-reference/functions/complete-snowflake-cortex) in the `COMPLETE` doc

### Guided lab — pick one or do both
- **Option A — LLMOps:** [Getting Started with LLMOps using Snowflake Cortex and TruLens](https://github.com/Snowflake-Labs/sfguide-getting-started-with-llmops-using-snowflake-cortex-and-trulens). Instrument your week-2 RAG app with TruLens and compute groundedness, context relevance, and answer relevance scores automatically
- **Option B — Agent Evaluations:** [Getting Started with Cortex Agent Evaluations](https://www.snowflake.com/en/developers/guides/getting-started-with-cortex-agent-evaluations/). Uses the built-in evaluation framework to score your week-4 agent on groundedness, tool selection, and execution efficiency

### Stretch task
Build a cost dashboard: query `SNOWFLAKE.ACCOUNT_USAGE.CORTEX_FUNCTIONS_USAGE_HISTORY` and visualize credit burn per function, per model, per day. Set a weekly budget alert on your warehouse. Future-you will be grateful.

### Reflection prompts
- What's the cheapest model that still meets your quality bar on your use case? (This is almost never the biggest one.)
- Which eval metric best caught your agent's real-world failures?

---

## Week 6 — Ship something small

No new concepts. Build one thing end-to-end and show it to someone.

Pick ONE of these, scoped to be finish-able in 5–7 hours:

- **A:** A Slack-style Q&A bot over your team's internal docs using Cortex Search + Agent
- **B:** A "chat with your warehouse" Streamlit app with a proper semantic model over a dataset you know well
- **C:** A document-processing pipeline using [`AI_PARSE_DOCUMENT` / `AI_EXTRACT`](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql) that turns a stage of PDFs into a structured table

### Deliverables
- Working app in [Streamlit in Snowflake](https://docs.snowflake.com/en/developer-guide/streamlit/about-streamlit)
- README with the architecture, cost per 1000 queries, and known failure modes
- A 10-question eval set with pass/fail, run before and after one round of improvements

### Final reflection
Go back through your `CORTEX_JOURNAL`. Write one paragraph answering: *If a colleague asked me "should we use Cortex for X?", what are the three questions I'd ask them first?* That paragraph is your real learning outcome.

---

## Reference: the sample projects you'll use

| Week | Quickstart / Repository |
|------|-------------------------|
| 1 | [sfguide-getting-started-with-snowflake-arctic-and-snowflake-cortex](https://github.com/Snowflake-Labs/sfguide-getting-started-with-snowflake-arctic-and-snowflake-cortex) |
| 2 | [Cortex Search tutorials](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/overview-tutorials) + [Snowflake-Labs/cortex-search](https://github.com/Snowflake-Labs/cortex-search) |
| 3 | [Cortex Analyst quickstart](https://quickstarts.snowflake.com/guide/getting_started_with_cortex_analyst/index.html) + [sfguide-getting-started-with-cortex-analyst-in-snowflake](https://github.com/Snowflake-Labs/sfguide-getting-started-with-cortex-analyst-in-snowflake) |
| 4 | [Cortex Agents quickstart](https://quickstarts.snowflake.com/guide/getting_started_with_cortex_agents/index.html) + [sfguide-getting-started-with-cortex-agents](https://github.com/Snowflake-Labs/sfguide-getting-started-with-cortex-agents) |
| 5 | [sfguide-getting-started-with-llmops-using-snowflake-cortex-and-trulens](https://github.com/Snowflake-Labs/sfguide-getting-started-with-llmops-using-snowflake-cortex-and-trulens) or [Cortex Agent Evaluations](https://www.snowflake.com/en/developers/guides/getting-started-with-cortex-agent-evaluations/) |
| Bonus | [sfguide-getting-started-with-cortex-agents-and-react](https://github.com/Snowflake-Labs/sfguide-getting-started-with-cortex-agents-and-react) (Next.js front-end) |

## Core doc bookmarks

- [Snowflake AI & ML overview](https://docs.snowflake.com/en/guides-overview-ai-features)
- [Cortex AISQL functions](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql)
- [Cortex Search](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/cortex-search-overview)
- [Cortex Analyst](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst)
- [Cortex Agents](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents)
- [Snowflake 2026 release notes](https://docs.snowflake.com/en/release-notes/new-features-2026)

## Ongoing habits

- Check the [Snowflake release notes](https://docs.snowflake.com/en/release-notes/new-features-2026) every Monday — Cortex ships features weekly and you'll miss useful things otherwise
- When stuck, try [**Cortex Code in Snowsight**](https://docs.snowflake.com/en/release-notes/2026/other/2026-03-09-cortex-code-snowsight-ga) itself (GA as of March 2026) — it's an agentic assistant trained on Snowflake docs
- Rewrite old notebooks every few weeks; the API surface is still evolving
