# Snowflake Cortex: A 6-Week Learning Plan

**Tailored for:** Someone with solid Snowflake/SQL skills, new to AI/ML
**Weekly commitment:** 5–7 hours
**Goal:** Go from zero to shipping a working Cortex Agent that combines RAG, text-to-SQL, and LLM reasoning.

---

## How we'll work together

Each week follows the same rhythm so you build habits, not just knowledge:

1. **Concept primer (45–60 min)** — read docs, watch a short video, understand the *why*
2. **Guided lab (2–3 hrs)** — run an official Snowflake-Labs quickstart, typing (not copy-pasting) the code
3. **Stretch task (1–2 hrs)** — modify the lab with your own data or a twist I'll suggest
4. **Reflection (30 min)** — write 3–5 bullets in a running journal: what clicked, what didn't, one question

Keep a Snowflake Notebook called `CORTEX_JOURNAL` for this. You'll thank yourself in week 5.

---

## Week 0 — Setup (one evening, ~1 hour)

Do this before week 1 so you're not fighting environment issues later.

- Create a Snowflake trial account (or use an existing dev account) in a region that supports Cortex — **AWS us-west-2, us-east-1, or eu-central-1** are safest
- Grant yourself the `CORTEX_USER` database role on your user
- Create a dedicated database `CORTEX_LAB` with schemas `RAW`, `CURATED`, `APPS`
- Create a small XS warehouse `CORTEX_WH` with auto-suspend 60s
- Install the Snowflake CLI and verify `snow connection test` works
- Clone these two repos locally — you'll use them all six weeks:
  - `github.com/Snowflake-Labs/sf-samples`
  - `github.com/Snowflake-Labs/sfguides`

**Checkpoint:** Run `SELECT SNOWFLAKE.CORTEX.COMPLETE('claude-3-5-sonnet', 'Say hi');` in a worksheet. If you get a response, you're ready.

---

## Week 1 — Cortex AISQL & LLM Functions

**The mental model to build this week:** Cortex LLM functions are just SQL functions. `AI_COMPLETE`, `AI_SUMMARIZE`, `AI_CLASSIFY`, `AI_EXTRACT`, `AI_TRANSLATE`, `AI_EMBED` — each one takes text in and returns text, JSON, or a vector. No Python, no API keys, no separate service. This is the biggest "aha" moment for SQL-first people, so take time to let it sink in.

### Concept primer
- Read: *Snowflake Cortex AISQL* overview in the docs (the page listing all `AI_*` functions)
- Read: the `AI_COMPLETE` reference page end-to-end — pay attention to model options and the `options` argument (temperature, response_format)
- Skim: the model lifecycle doc so you understand "preview" vs "GA" models

### Guided lab
- **Quickstart:** *Getting Started with Snowflake Arctic and Snowflake Cortex* (Snowflake-Labs/sfguide-getting-started-with-snowflake-arctic-and-snowflake-cortex)
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
- Read: *Cortex Search overview* in the docs
- Read: the March 2026 update notes on **multi-index search services** and **custom vector embeddings** — these are recent and genuinely useful
- Understand the difference between `AI_EMBED` (one-shot embedding function) and a Cortex Search service (managed, refreshed, queryable)

### Guided lab
- Build a Cortex Search service over a documentation corpus. The Streamlit docs or Snowflake's own docs (available as a public dataset) are ideal.
- Query it from SQL using `SEARCH_PREVIEW`, then from Python using the Snowpark snowflake-ml `CortexSearchRetriever` class

### Stretch task
Build a minimal RAG pipeline in a Snowflake Notebook: user question → Cortex Search retrieves top 5 chunks → stuff into a prompt → `AI_COMPLETE` generates grounded answer. About 40 lines of Python. Test with 10 questions and note which ones fail and why (hallucination? retrieval miss? chunking problem?).

### Reflection prompts
- What's the failure mode when the answer isn't in your corpus?
- How would you evaluate retrieval quality without a human reviewer?

---

## Week 3 — Cortex Analyst (text-to-SQL)

**The mental model:** Cortex Analyst is *not* "an LLM writing SQL against your tables." It's an LLM writing SQL against a **semantic model** — a YAML file where you define the business meaning of tables, columns, metrics, and synonyms. The quality of your YAML determines the quality of your answers. This week is 30% LLM, 70% data modeling discipline.

### Concept primer
- Read: *Cortex Analyst overview* and the *semantic model spec* docs end-to-end. This is the week you cannot skim.
- Read the March 2026 preview note about **relationship paths in semantic views** — it changes how you model joins

### Guided lab
- **Quickstart:** *Getting Started with Cortex Analyst* (Snowflake-Labs/sfguide-getting-started-with-cortex-analyst-in-snowflake)
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
- Read: *Cortex Agents* docs and the agent spec
- Review the March 2026 Cortex Search tool update — agents can now target a specific search service by description rather than fanning out

### Guided lab
- **Quickstart:** *Getting Started with Cortex Agents* (Snowflake-Labs/sfguide-getting-started-with-cortex-agents)
- This is the capstone lab: you'll reuse the Cortex Search service from week 2 and the semantic model from week 3, wire them to a single agent, and chat with it via Snowflake Intelligence and a Streamlit app

### Stretch task
Add a third tool to your agent — either a custom SQL function you write yourself or a second Cortex Search service over a different corpus. Ask questions that should route to each of the three tools and verify routing behaves correctly. Screenshot the tool-call traces.

### Reflection prompts
- How does the agent decide which tool to call? What signal dominated?
- Where would a human reviewer still catch errors the agent missed?

---

## Week 5 — Evaluation, cost, and LLMOps

Now that you can build, learn what "good" means.

### Concept primer
- Read: *Cortex credits and cost* docs — understand per-token pricing for different models and how to estimate cost before shipping
- Read: the Cortex Guardrails feature docs

### Guided lab
- **Quickstart:** *Getting Started with LLMOps using Snowflake Cortex and TruLens* (Snowflake-Labs/sfguide-getting-started-with-llmops-using-snowflake-cortex-and-trulens)
- You'll instrument your week-2 RAG app with TruLens and compute groundedness, context relevance, and answer relevance scores automatically

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
- **C:** A document-processing pipeline using `AI_PARSE_DOCUMENT` + `AI_EXTRACT` that turns a stage of PDFs into a structured table

### Deliverables
- Working app in Streamlit in Snowflake
- README with the architecture, cost per 1000 queries, and known failure modes
- A 10-question eval set with pass/fail, run before and after one round of improvements

### Final reflection
Go back through your `CORTEX_JOURNAL`. Write one paragraph answering: *If a colleague asked me "should we use Cortex for X?", what are the three questions I'd ask them first?* That paragraph is your real learning outcome.

---

## Reference: the sample projects you'll use

| Week | Repository |
|------|-----------|
| 1 | `Snowflake-Labs/sfguide-getting-started-with-snowflake-arctic-and-snowflake-cortex` |
| 2 | Cortex Search sections of `Snowflake-Labs/sf-samples` |
| 3 | `Snowflake-Labs/sfguide-getting-started-with-cortex-analyst-in-snowflake` |
| 4 | `Snowflake-Labs/sfguide-getting-started-with-cortex-agents` |
| 5 | `Snowflake-Labs/sfguide-getting-started-with-llmops-using-snowflake-cortex-and-trulens` |
| Bonus | `Snowflake-Labs/sfguide-getting-started-with-cortex-agents-and-react` (Next.js front-end) |

## Ongoing habits

- Check the Snowflake release notes every Monday — Cortex ships features weekly and you'll miss useful things otherwise
- When stuck, try **Cortex Code in Snowsight** itself (GA as of March 2026) — it's an agentic assistant trained on Snowflake docs
- Rewrite old notebooks every few weeks; the API surface is still evolving
