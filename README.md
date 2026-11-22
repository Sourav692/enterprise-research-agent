# Enterprise AI Research Agent

A multi-agent research assistant for a (fictional) fintech company, **LedgerLeaf Payments**. It answers natural-language questions from **internal company documents (RAG)** and the **web**, cites every claim, flags **conflicts** between internal and external sources, **asks for clarification** when a question is ambiguous, remembers **conversation context**, and pauses for **human approval** on sensitive operations.

Built with **LangGraph** (orchestration) · **LangChain** (OpenAI-compatible, Tavily, Chroma, HuggingFace integrations) · **FastAPI** · **ChromaDB** · **Pydantic** · **SQLite** · LLMs via **OpenRouter** (default: Claude Sonnet 5.5 + Haiku 5.5; any model with structured outputs works).

> **Runs without API keys.** With no keys, the system uses an offline heuristic LLM and a labelled fictional web corpus, so the API, demo, tests and CI all work out of the box. Add `OPENROUTER_API_KEY` and `TAVILY_API_KEY` for the real thing.

![architecture](docs/architecture.png)

*(If the image is missing, see the Mermaid diagrams in [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md), which render on GitHub.)*

---

## How it works

```
question ─► contextualize ─► classify ─┬─► clarify ⏸ ──► (back to classify)
                                       ├─► direct reply
                                       └─► privacy guard ⏸ ─┬─► internal RAG ─┐   (parallel)
                                                            └─► web search ───┴─► validate ─┬─► reformulate ─► (retry)
                                                                                            ├─► detect conflicts ─┐
                                                                                            └─────────────────────┴─► synthesize ⇄ groundedness check
                                                                                                                          └─► approval gate ⏸ ─► finalize
```

⏸ = LangGraph `interrupt()`: the API returns `needs_clarification` / `pending_approval`, and the conversation resumes from its SQLite checkpoint.

| Agent                          | Job                                                                                                  |
| ------------------------------ | ---------------------------------------------------------------------------------------------------- |
| **Contextualizer**       | rewrites follow-ups into standalone questions; summarises long histories                             |
| **Router/Classifier**    | `internal` / `web` / `both` / `clarify` / `direct`, with separate internal and web queries |
| **Privacy guard**        | stops confidential terms from leaking into external web queries without approval                     |
| **Internal RAG agent**   | ChromaDB semantic search over chunked docs with classification and date metadata                     |
| **Web research agent**   | Tavily search                                                                                        |
| **Validator**            | grades relevance and drops noise; triggers query reformulation when nothing is relevant              |
| **Conflict detector**    | structured internal vs external disagreements, with dates and likely resolution                      |
| **Synthesizer**          | evidence-only answer with `[S#]` citations and an explicit claim list                              |
| **Groundedness checker** | verifies each claim against its cited source; forces regeneration on hallucinations                  |
| **Approval gate**        | human review for confidential sources or low-confidence answers                                      |

Full details are in **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)**.

---

## Quickstart

Prerequisites: Python 3.11–3.13 and [uv](https://docs.astral.sh/uv/) (`pip install uv`).

```bash
uv sync --extra local-embeddings     # install (includes sentence-transformers for local embeddings)
cp .env.example .env                 # optional: add OPENROUTER_API_KEY / TAVILY_API_KEY
uv run research-agent ingest --reset # build the vector index (downloads bge-small, ~130MB, once)
uv run research-agent serve          # API on http://127.0.0.1:8000, Swagger UI at /docs
```

Other entry points:

```bash
uv run research-agent chat                          # interactive CLI chat (in-process)
uv run research-agent chat --url http://127.0.0.1:8000   # CLI against a running server
uv run python scripts/demo.py                       # run the 10 demo scenarios -> docs/DEMO.md
uv run python -m research_agent.evaluation          # evaluation -> reports/eval_report.md
uv run pytest                                       # 55 offline unit + integration tests
uv run pytest -m live                               # end-to-end tests with real APIs (needs keys)
```

A `Makefile` wraps these (`make install ingest serve chat demo eval test`).

**Docker:** `docker compose up --build` (builds the index into the image; conversations persist in a volume).

**Windows note:** the commands above work in PowerShell; use `uv run ...` directly if `make` is not installed.

### Using real providers

| Variable                                           | Effect                                                                                                                                                                                                                            |
| -------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OPENROUTER_API_KEY`                             | switches the LLM from the offline mock to OpenRouter (`SMART_MODEL=anthropic/claude-sonnet-5.5`, `FAST_MODEL=anthropic/claude-haiku-5.5` by default; change them to any OpenRouter model id that supports structured outputs) |
| `OPENROUTER_FALLBACK_MODELS`                     | optional JSON list of models OpenRouter falls back to if the primary fails                                                                                                                                                        |
| `TAVILY_API_KEY`                                 | switches web search from fixtures to Tavily (free tier at tavily.com)                                                                                                                                                             |
| `LANGSMITH_TRACING=true` + `LANGSMITH_API_KEY` | full LangGraph traces in LangSmith                                                                                                                                                                                                |

`GET /health` shows which providers are active. See `.env.example` for every setting (HITL toggles, thresholds, retry limits...).

---

## Demo scenarios

`scripts/demo.py` drives the real HTTP API and writes the transcript to **[docs/DEMO.md](docs/DEMO.md)**:

| #  | Scenario                                                                     | Shows                                                                                                       |
| -- | ---------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| 1  | "What is our parental leave policy and how many PTO days...?"                | internal RAG with citations                                                                                 |
| 2  | "Latest trends in real-time payments adoption?"                              | web-only routing                                                                                            |
| 3  | "How do PayNova's current fees compare to our pricing?"                      | parallel internal + web retrieval, **conflict detected** (2025 doc says 2.4%, 2026 article says 1.9%) |
| 4  | "Is our treasury Fed funds rate assumption still valid?"                     | stale internal assumption vs current data                                                                   |
| 5  | "What's the limit?" → "The API rate limit"                                  | **clarification** interrupt and resume                                                                |
| 6  | "What uptime does our SLA guarantee?" → "And what credits...if we miss it?" | **multi-turn context**                                                                                |
| 7  | "What is the status of Project Falcon?"                                      | **HITL**: confidential sources need approval                                                          |
| 8  | "Search the web for news about Project Falcon"                               | **HITL privacy guard**: reviewer blocks the leak, so internal-only                                    |
| 9  | "Policy on cryptocurrency staking rewards?"                                  | hallucination guard: abstains                                                                               |
| 10 | `POST /documents` → approve → ask again                                  | **HITL knowledge-base write**, immediately searchable                                                 |

The knowledge base (`data/docs/`, 21 docs) includes deliberately **stale** documents (2024 regulatory watch, 2024 treasury assumptions, 2025 competitor analysis) and **confidential** ones (acquisition plan, salary bands, board minutes) to exercise conflict detection and approvals.

---

## Requirements checklist

| Requirement                                              | Where                                                                                                                         |
| -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| Natural-language queries                                 | `POST /chat`, CLI `chat`                                                                                                  |
| Query classification and routing (internal / web / both) | `agents/router.py`, `graph/builder.py::route_after_classify, fan_out`                                                     |
| RAG over internal KB                                     | `rag/` (front-matter loader, header-aware chunking, ChromaDB, bge-small embeddings)                                         |
| Web search for current info                              | `tools/web_search.py` (Tavily via `langchain-tavily`)                                                                     |
| Multiple specialised agents that process and validate    | 10 agent nodes in `agents/` (validator, conflict detector, groundedness checker...)                                         |
| Synthesis into one answer                                | `agents/synthesizer.py`                                                                                                     |
| Citations for every claim                                | `[S#]` markers, claim→source mapping, citation sanitising, `citations[]` in API                                          |
| Conflicting information                                  | `agents/conflicts.py`: structured `conflicts[]` + answer section                                                          |
| Clarification on ambiguity                               | `clarify` node with `interrupt()` + `/resume`                                                                           |
| Conversation context                                     | SQLite checkpointer per `thread_id`, contextualizer, history summarisation                                                  |
| Conditional routing                                      | 5 conditional edges, including loops (clarify, reformulate, regenerate) and a parallel fan-out                                |
| Retries for failed tool calls                            | `resilience.py` (tenacity, exponential backoff + jitter, non-retryable classification)                                      |
| Structured state                                         | `graph/state.py` (Pydantic `ResearchState`, custom reducers) and Pydantic outputs for every agent                         |
| Human-in-the-loop                                        | 5 approval types (`docs/ARCHITECTURE.md#human-in-the-loop`) + `/approvals` queue                                          |
| Logging and observability                                | structlog with correlation IDs, per-node/LLM/tool metrics, `/metrics`, `/audit`, optional LangSmith                       |
| Evaluation                                               | `evaluation.py`: retrieval hit@k/recall@k/MRR, routing accuracy, LLM-as-judge faithfulness/relevance/citations/completeness |
| API failures, empty retrieval, hallucination             | per-agent fallbacks + warnings, reformulation then abstention, groundedness check + regeneration                              |
| Persistence                                              | SQLite (checkpoints + app DB)                                                                                                 |
| API docs, tests, diagram, README                         | `docs/API.md` + Swagger, `tests/`, `docs/ARCHITECTURE.md`, this file                                                    |

---

## Project layout

```
src/research_agent/
  agents/        one module per agent + prompts.py + deps.py
  graph/         state.py (typed state, reducers) · builder.py (nodes, routing)
  llm/           base interface · openrouter_llm.py · mock_llm.py (offline heuristics)
  rag/           loader.py · embeddings.py · vectorstore.py
  tools/         web_search.py
  api/           app.py (FastAPI)
  observability/ logging.py · tracing.py (@traced_node)
  persistence/   db.py (threads, approvals, audit, metrics)
  service.py     composition root + interrupt/approval handling
  evaluation.py  evaluation harness
  resilience.py  retry helpers
  cli.py         typer CLI
data/docs/        internal knowledge base (markdown + front-matter)
data/mock_web/    offline web fixtures (clearly marked [mock])
eval/             golden_set.json (30 labelled questions)
tests/            unit/ · integration/ · live/
docs/             ARCHITECTURE.md · API.md · DEMO.md · *.mmd diagrams
```

## Evaluation results

`reports/eval_report.md` is regenerated by `make eval`. In offline mock mode (keyword router, extractive synthesis, lexical judge) with local bge-small embeddings:

| metric                                       | value            |
| -------------------------------------------- | ---------------- |
| retrieval hit@5 / recall@5 / MRR             | 1.0 / 1.0 / 0.98 |
| routing accuracy                             | 0.90             |
| conflict detection rate (3 seeded conflicts) | 1.0              |
| abstention on unanswerable question          | 1.0              |

Answer-quality judge scores in mock mode are heuristic. Run with an OpenRouter key for real LLM-as-judge numbers.

## Limitations and next steps

- Mock mode is for development and CI only; its answers are extractive and its routing is keyword-based.
- No authentication: `user_id`/`reviewer` are trusted fields. Add API keys or OAuth and role-based approvers before real use.
- Document-level access control (filtering confidential docs by user role *before* retrieval) would complement the approval gate.
- Single-process SQLite; switch to `PostgresSaver` + Postgres for multi-instance deployments.
