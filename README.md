
# Pavan Sai Bellapu

**AI and backend engineer.** Eight years of async Python and distributed systems, the last few spent putting LLMs into production: retrieval, model routing, caching, evals, and the backend underneath all of it.

Bengaluru · [GitHub](https://github.com/devnevo) · [LinkedIn](https://www.linkedin.com/in/pavan-sb/) · [Writing](https://medium.com/@pbellapu09) · [pbellapu09@gmail.com](mailto:pbellapu09@gmail.com)

```text
$ whoami
backend + applied AI engineer · 8 yrs in production · Bengaluru

$ cat ~/scale
6M+ users · 2,000+ schools · 50K+ LLM interactions/month · 100+ req/s under 500 ms

$ cat ~/focus
agents         langgraph · mcp tool servers · tool calling · structured outputs · guardrails
correctness    commit-time checks for agent writes · idempotency · receipts · crash tests
retrieval      parent-child chunking · per-tenant indexes · citations · grounded refusals
evals          golden sets · ragas · llm-as-judge · promptfoo · regressions fail the build
observability  opentelemetry · langfuse · prometheus + grafana · alerts with runbooks
inference      model routing · context compression · response caching · cost per request
backend        async python · fastapi / tornado · sharded mongodb · redis · gcp cloud run
```

## Shipped in production

**Cloudaccel, 2018–2025.** Multi-tenant parent–teacher communication SaaS on GCP. 6M+ users, 1M+ monthly active, 2,000+ schools.

| | |
|---:|---|
| **$0.003** per LLM call | Production RAG cost after gating calls below a minimum input volume, deduplicating batches (~30% of spend was redundant), routing between Gemini Flash and Pro by task, and cutting token budgets from 8K to 1.2K. 62% below launch cost, ~85% below a GPT-4-class baseline. |
| **50K+** AI calls / month | RAG pipeline (FAISS, transformer embeddings, Gemini via LangChain) generating newsletters and behaviour reports for 2,000+ schools. |
| **<10 ms** per-school index load | School Answers AI: a parent Q&A assistant grounded in each school's own documents (LangChain loaders, Vertex AI embeddings, one FAISS index per school, Gemini). Answers cite sources, stay scoped to the parent's school, and decline when the documents don't cover the question. Caching each school's index cut load time from 200–500 ms to under 10 ms. |
| **99.8%** cache hit rate | Three-level cache, process-local → Redis → MongoDB, with pub/sub invalidation across async instances. Tenant-config reads down 99%, average query latency 45 ms → 18 ms at 100+ req/s. |
| **5,531** queries in one request | Found by an AST-based N+1 detector I wrote to scan 3,712 Python files. The audit catalogued 105+ bottlenecks, including a polling loop behind 80% of autoscaler scale-ups. Shipped the caching fix behind feature flags: ~$20K/yr saved, sub-500 ms P95 on critical endpoints. |
| **500K+** rows / day | MongoDB → BigQuery ETL for message-status records. Protobuf streaming inserts, p99 under 500 ms, inside a 45-minute window. |
| **8** content types | Dialogflow CX agent with multi-turn intent extraction and webhook tool calls. Replaced 8–10-click admin workflows with one sentence. |
| **−40%** number cost | Twilio and Vonage messaging at platform scale: HMAC webhook validation, 10DLC registration, GSM-7 enforcement, number pooling. SMS segment cost also down 15%. |

Also owned: SSO and identity (Google, Microsoft, Clever, ClassLink, SAML), and the sharded newsletter and forms system with translation into 150+ languages.

## Building in the open

Since January 2026 I have been retooling on purpose: agents, evals, observability, fine-tuning. Everything below is public and ships in weekly milestones.

### [fairwitness-db](https://github.com/devnevo/fairwitness-db) · commit-time checks for database changes proposed by AI agents

Agents read, reason, then write, and the data can change in between. fairwitness-db re-checks the facts an agent relied on *inside* the writing transaction, then commits or rejects with a recorded reason.

```text
agent reads a snapshot  →  proposes a typed operation + the facts it observed
fairwitness-db          →  open txn · reload authoritative state · check preconditions
                        →  commit + receipt   |   reject + receipt (reason, which facts drifted)
```

- **The demo.** Reproduces "the database committed the wrong refund": a naive write refunds a disputed order, and a review step that only sees the agent's original data makes the same mistake. fairwitness-db rejects it with `order_disputed` and lists the facts that changed.
- **Measured, not argued.** 200 seeded runs, 30 agents racing disputes and a policy change, 6,000 decisions. The naive baseline broke an invariant in every run (2,222 violations). fairwitness-db had none, and made exactly the same decisions as a correct hand-written transaction.
- **A trade-off with a number on it.** Strict mode (reject on *any* change to observed facts) committed about 40% fewer refunds in this simulation, for no gain on the rules that were written down. That is the price of guarding rules you didn't encode.
- **Crash- and race-tested.** Receipts commit in the same transaction as the effect, retries replay idempotently by operation ID, a simulated crash at every commit stage leaves nothing behind, and 16 threads race one shared budget. 32 tests on Python 3.10–3.13, zero runtime dependencies.
- **Honest about scope.** v0.1.0 is an alpha on SQLite, and the demo agent is deterministic Python, not an LLM. The idea has 2026 prior art (He & Yu, *When AI Agents Commit*; Sun, Wang & Li, *Agentic Transaction*), cited in the repo. This is a small, reproducible implementation with measurements.
- **Next:** PostgreSQL under `SERIALIZABLE` with overhead benchmarked · outbox and reconciliation for external payment providers · policy and authority checks.

### [school-answers-lab](SCHOOL_ANSWERS_REPO) · a production RAG rebuilt so I can break it on purpose

I built the original School Answers RAG at Cloudaccel: parents ask questions and get answers only from their own school's documents, with citations. This is a from-scratch rebuild with the model, search and parents simulated, so every failure can be injected and every signal checked against it.

```text
parent question (school + grade from login, never from the question)
  → input guard → router → answer cache → read-only SQL | per-school retrieval
  → model gateway (Flash/Pro, timeouts, retries, fallback)
  → output guard (citation required, cited school must match, every number must appear in a source)
```

Built in phases, each one checked against the faults before moving on:

1. **Retrieval: parent-child chunking.** Small child chunks (~400 characters) are matched for precision; the model receives the parent section (~2,000 characters), so a rule and its exception arrive together ("early pickup is allowed" *with* "a signed note by 10 am"). Parents are deduplicated and capped at four to bound context. One index per school is the tenant boundary.
2. **Tracing: OpenTelemetry → Langfuse.** One trace per question, a span per stage (guard, route, retrieve, generate, check), carrying school, prompt version, retrieved chunk IDs, tokens and cost.
3. **Metrics and alerts: Prometheus + Grafana.** Simulated parents at ~4 requests/s across three schools. Dashboards for latency per stage, cost per answer, cache hit rate, not-found rate and feedback; alert rules, each with a runbook.
4. **Quality: Ragas.** A sampled job scores faithfulness, answer relevancy and context precision and writes the scores back to the trace. A 57-question golden set gates every release in CI.
5. **Fault drills.** Eight switches inject real incidents: slow or failing provider, cold index cache, bad retriever, hallucinating prompt, cross-school leak, prompt bloat, router regression. Each fired its expected alert on live simulated traffic; seven of the eight also fail the release gate.

What the drills showed:

- Simulated parents' thumbs-up stayed flat while hallucinations rose. Only the scorer caught them.
- A bad retriever and a bad router looked identical on the dashboard. Only the trace told them apart.
- Releasing a new prompt version roughly doubled cost per answer, because it emptied the answer cache.
- The judge agreed with my hand labels only about two-thirds of the time, which is why it informs releases instead of blocking them.

### In progress

- **QLoRA fine-tuning** of Llama 3.1 8B and Qwen2.5 on a single GPU (PEFT, TRL, Unsloth) for structured-output generation. The eval set is written before training; every checkpoint is gated on schema validity and LLM-as-judge against the base model. W&B tracking, DPO next.
- **MLOps pipeline:** MLflow tracking and registry, Airflow orchestration, Evidently drift monitoring into Prometheus and Grafana.

<!--
## Upstream
- vllm-project/vllm #____ : one line on what it fixed
- langchain-ai/langgraph #____ : one line on what it fixed
-->

## Writing

- **[Your LLM eval has the same bug OpenAI's did](https://medium.com/@pbellapu09/your-llm-eval-has-the-same-bug-openais-did-and-why-this-may-be-one-of-the-few-times-we-could-44b451380ccb?sharedUserId=pbellapu09)**: graders that check the outcome and never the method, why that is the default state of CI-gated LLM-as-judge suites, and a six-point checklist for catching it.
- **[Notes from Running MongoDB in Production: What the Internals Actually Look Like](https://medium.com/@pbellapu09/notes-from-running-mongodb-in-production-what-the-internals-actually-look-like-06d7b04bae10?sharedUserId=pbellapu09)**
- **[RAG and the Bitter Lesson: Why AI Scaffolding Has an Expiry Date](https://medium.com/@pbellapu09/rag-and-the-bitter-lesson-why-ai-scaffolding-has-an-expiry-date-9eeafb00d8c9?sharedUserId=pbellapu09)**
- **[How Claude Code Manages Infinite Sessions Without Blowing the Context Window](https://medium.com/@pbellapu09/how-claude-code-manages-infinite-sessions-without-blowing-the-context-window-8bccb35387fe?sharedUserId=pbellapu09)**

## Toolbox

```text
languages   python (asyncio) · typescript · sql
backend     fastapi · tornado · pydantic · rest · protobuf · pub/sub · webhooks (hmac)
llm         langgraph · langchain · mcp · vertex ai (gemini) · aws bedrock · dialogflow cx
retrieval   faiss · qdrant · bm25 · embeddings · parent-child chunking · semantic search
ml          pytorch · transformers · peft · trl · unsloth · w&b · mlflow
evals       ragas · promptfoo · llm-as-judge · golden sets · judge calibration
obs         opentelemetry · langfuse · langsmith · prometheus · grafana · alerting + runbooks
data        mongodb (sharding, aggregation) · redis · postgres · sqlite · bigquery · airflow
infra       gcp (cloud run, cloud tasks, pub/sub) · docker · terraform · jenkins
learning    spark · kafka · dbt · kubernetes
```

## Now
- Shipping fairwitness-db v0.2 and the agentic RAG system, one milestone a week.
- Open to senior AI and backend roles. Bengaluru or remote, can join immediately.
