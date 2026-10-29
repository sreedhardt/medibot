# MediBot

A role-based-access RAG assistant for a hospital network, and the evaluation
and guardrail pipeline that wraps it.

| Directory | What it is |
|---|---|
| [`app/`](app) | **MediBot**: hybrid RAG (dense + BM25 + cross-encoder rerank) and SQL RAG over hospital documents and a claims/maintenance database. Access control is enforced inside the vector search, so a role never retrieves a chunk it isn't allowed to see. FastAPI backend, Next.js chat UI. |
| [`evalguard/`](evalguard) | **EvalGuard**: input and output guardrails with fail-closed verdicts, LangSmith tracing of the full request path, and a repeatable evaluation run (RAGAS, an LLM judge, deterministic heuristics and planted controls) that produces a pass/fail report. |

## How they fit together

EvalGuard imports MediBot's backend in-process as a library and calls its
routing, retrieval, reranking, generation and SQL RAG functions directly. That
lets every step appear as its own trace span, and gives evaluation the retrieved
chunk text that MediBot's HTTP response doesn't carry. MediBot runs on its own
and doesn't depend on EvalGuard.

EvalGuard finds MediBot through `MEDIBOT_ROOT`, which defaults to `../app`.

The two projects have separate `pyproject.toml` files and environments, because
their dependencies differ (EvalGuard adds RAGAS, OpenEvals and LangSmith). The
[EvalGuard findings](evalguard/README.md#findings) include a MediBot bug the
evaluation run caught, which was then fixed in `app/`.

## Quick start

Requires Python 3.12, [uv](https://docs.astral.sh/uv/), Docker and a
[Groq API key](https://console.groq.com/keys). EvalGuard also uses a
[LangSmith API key](https://smith.langchain.com/) for tracing.

```bash
# MediBot: start Qdrant and index the corpus (once)
cd app
cp .env.example .env            # add GROQ_API_KEY and a JWT_SECRET
docker compose up -d
uv sync && (cd backend && uv run python -m ingest.run_ingest)

# EvalGuard
cd ../evalguard
cp .env.example .env            # add GROQ_API_KEY and LANGSMITH_API_KEY
uv sync
uv run pytest
```

See [`app/README.md`](app/README.md) for the UI, demo credentials and RBAC
evidence, and [`evalguard/README.md`](evalguard/README.md) for running the
guardrail gateway and the evaluation.
