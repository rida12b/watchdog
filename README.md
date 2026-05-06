# Watchdog

> **Multi-agent system that monitors, investigates, and mitigates anomalies in production AI agentic systems — in real time.**

**Status**: Phase 0 — initial setup complete, MVP in active development. Targeted MVP delivery: end of Q3 2026.
**Version**: 0.1.0
**License**: Proprietary

---

## The problem

Only **14%** of companies with an agentic AI pilot manage to scale it to production (vs 78% with a pilot). **89% of those failures** trace back to a single root cause: **lack of monitoring and observability tooling for AI agents in production**. *(Source: Datadog State of AI Engineering 2026.)*

Existing observability stacks (Datadog, Langfuse, LangSmith, Helicone) **stop at logging** — they record what happened, but don't detect, investigate, or mitigate. When an AI agent in production starts hallucinating, leaking data, or burning your budget, you typically find out from the monthly bill or from a customer complaint. **Watchdog closes that gap.**

---

## What Watchdog does

Watchdog plugs into your production AI agents (RAG pipelines, customer support agents, code reviewers, etc.) and provides **autonomous, real-time supervision**:

- **Detects** anomalies in under 60 seconds — cost spikes, latency spikes, error rate, hallucination rate, semantic drift, prompt injection
- **Investigates** automatically — a multi-agent system reads traces, identifies probable cause, structures the context
- **Proposes mitigations** — actionable suggestions: throttle a user, switch fallback model, isolate an agent, block input pattern
- **Kill switch** — isolates a compromised agent in under 5 seconds
- **Self-monitors** — meta-tracing of Watchdog's own decisions, with a recursive evaluation loop that learns from human feedback

---

## Quick start

```bash
git clone https://github.com/rida12b/watchdog.git
cd watchdog
pip install -e ".[dev]"
cp .env.example .env       # edit .env with your config
uvicorn watchdog_ai.main:app --reload --port 8000
```

Once running, send a test telemetry event:

```bash
curl -X POST http://localhost:8000/api/v1/events \
  -H "X-Tenant-ID: demo" \
  -H "Content-Type: application/json" \
  -d '{"agent_id": "test-agent", "model": "gpt-4-turbo", "prompt": "hello", "output": "world", "prompt_tokens": 1, "completion_tokens": 1, "total_tokens": 2, "cost_usd": 0.0001, "status": "success"}'
```

---

## Architecture

Watchdog is structured in 8 layers:

1. **Ingestion** — telemetry receive (FastAPI)
2. **Storage** — PostgreSQL + TimescaleDB + pgvector (single instance)
3. **Multi-agent core** — LangGraph chaining Watcher → Investigator → Mitigator → Alerter
4. **Eval engine** — RAGAS + custom LLM-as-judge
5. **API + dashboard** — REST API + minimal admin UI + MCP server
6. **Auth + multi-tenant** — JWT + RBAC + strict tenant isolation
7. **Mitigation outbound** — kill switch + webhooks + rule injection
8. **Meta-observability** — Langfuse self-tracing + recursive eval loop

Stack: Python 3.13, FastAPI, Pydantic v2, PostgreSQL 16 + TimescaleDB + pgvector, LangGraph, Azure OpenAI (primary) + Mistral (fallback), Langfuse self-hosted, Docker → Azure Container Apps.

📖 **Full architecture, scenarios, and roadmap** → [VISION.md](VISION.md)

---

## Project status

| Phase | Scope | Target |
|---|---|---|
| **Phase 0** — Setup | pyproject, package skeleton, public docs | ✅ Complete |
| **Phase 1** — MVP M1 | Telemetry ingestion + cost-spike detection + multi-tenant auth (S1, S2, S6) | 🔨 In progress |
| **Phase 2** — MVP M2 | Multi-agent investigation + mitigation + alerting (S3, S4, S5) | ⏳ Planned |
| **Phase 3** — Full | Eval, kill switch, drift, prompt injection, RAG history, MCP, meta-tracing (S7-S13) | ⏳ Planned |
| **Phase 4** — Public demo | Azure Container Apps deployment + live dashboard | ⏳ Planned |

---

## Documentation

- [VISION.md](VISION.md) — full project vision, architecture, scenarios, roadmap, market analysis
- `LICENSE` — proprietary, see file for terms

---

## Contact

Author: Rida Boualam
GitHub: [@rida12b](https://github.com/rida12b)
