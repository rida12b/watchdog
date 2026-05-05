# Watchdog — AI Agent Production Watchdog

> **Multi-agent system that monitors, investigates, and mitigates anomalies in production AI agentic systems — in real time.**

Version: 0.1.0
Status: Active development — MVP targeted Q3 2026

---

## 1. The problem

### 1.1 Who it's for

**Target user**: AI / Platform Engineering teams (3-15 people) running one or more AI systems in production — corporate RAG, customer support agents, code reviewers, document processors — on Azure, AWS, or GCP.

### 1.2 Why this exists (market signals 2026)

- Only **14%** of companies with an agentic AI pilot manage to scale it to production (vs 78% who have a pilot). **89% of failures** trace back to 5 gaps — chief among them: **lack of monitoring tooling**.
  *(Datadog State of AI Engineering 2026)*
- **67%** of executives admit they have already experienced a data leak through unsanctioned AI tools.
- **36%** have no agent supervision plan at all.
- **35%** admit they would not be able to *"unplug"* a rogue agent.
- **40%+** of agentic AI projects will fail by end of 2027 due to escalating costs, unclear value, and insufficient risk controls.
  *(Gartner)*

What this audience would buy: a system that **detects** anomalies on their production agents, **investigates** them automatically, **proposes** mitigations, and lets them **isolate** a compromised agent in seconds — without building this monitoring infrastructure themselves.

---

## 2. Core promise

> **In one sentence**: Watchdog detects anomalies on a production AI agent in less than 60 seconds, automatically investigates them with structured context, proposes an actionable mitigation, and supports immediate isolation (kill switch) when needed.

### Success conditions (ALL must be true to call Watchdog "delivered")

1. ✅ Automatic detection of at least 6 anomaly types (cost spike, latency, error rate, hallucination rate, semantic drift, prompt injection)
2. ✅ Time-to-detect under 60 seconds for an active anomaly
3. ✅ Structured automatic investigation (probable cause + traces + samples + similar history)
4. ✅ Actionable proposed mitigation (not just an "alert" — a real human-validatable action)
5. ✅ Working kill switch: isolation of a compromised agent in under 5 seconds
6. ✅ Automatic output quality evaluation (≥ 1 dimension, ideally RAGAS + custom LLM-as-judge)
7. ✅ Functional multi-tenant (1 customer = 1 isolated workspace, strictly separated data)
8. ✅ Watchdog self-monitors itself (Langfuse meta-tracing)
9. ✅ Searchable incident history (RAG over past incidents feeds future investigations)
10. ✅ MCP server (Watchdog can be exposed as a tool to other agents)

---

## 3. Glossary

| Term | Definition |
|---|---|
| **Monitored Agent** | An AI agent or RAG pipeline running in production that sends its telemetry to Watchdog. *Not* the Watchdog itself. |
| **Telemetry Event** | A trace unit: an LLM call with its metadata (input prompt, output, latency, cost, tokens, model, agent_id, tenant_id, timestamp). |
| **Anomaly** | An event (or pattern of events) flagged by the Watcher as statistically or semantically abnormal — based on rules, thresholds, and models. |
| **Incident** | An anomaly that has been escalated — requires investigation and potentially mitigation. |
| **Investigation** | The process by which the Investigator collects context (traces, logs, samples), forms a probable-cause hypothesis, and structures the data for the next step. |
| **Mitigation Action** | A concrete proposed action to resolve the incident (isolate agent, switch fallback model, block an input pattern, add a filtering rule, etc.). May be suggested for human review or executed automatically depending on severity. |
| **Kill Switch** | Immediate isolation mechanism for a Monitored Agent. Blocks all its future calls in under 5 seconds. |
| **Watchdog Agent** | One of the 4 agents that compose the Watchdog system (Watcher, Investigator, Mitigator, Alerter). Not to be confused with Monitored Agent. |
| **Tenant** | A customer / organization using Watchdog. Tenant data is strictly isolated from others. |
| **Eval Output** | An automatic quality score assigned to a Telemetry Event or sample (e.g., RAGAS faithfulness, LLM-as-judge relevance). |
| **Recursive Eval** | A differentiating innovation: Watchdog retroactively evaluates its own detections (a false positive, once confirmed by humans, feeds the incident RAG to improve future detections). |

---

## 4. Architecture overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    MONITORED AGENTS (customers)                  │
│  RAG corp │ Customer support agent │ Code reviewer │ etc.       │
│                                                                  │
│  Each agent sends its telemetry → Watchdog                      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           │ HTTPS POST telemetry events
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                          WATCHDOG                                │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ LAYER 1 — INGESTION                                         │ │
│  │ FastAPI receive endpoint │ tenant auth │ validation         │ │
│  │ Queue (in-memory MVP, Redis full)                           │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                   │
│  ┌──────────────────────────▼─────────────────────────────────┐ │
│  │ LAYER 2 — STORAGE                                           │ │
│  │ PostgreSQL: events, incidents, agents, tenants              │ │
│  │ TimescaleDB: time-series metrics (cost, latency)            │ │
│  │ pgvector: incident embeddings (same PG instance, no extra)  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                   │
│  ┌──────────────────────────▼─────────────────────────────────┐ │
│  │ LAYER 3 — WATCHDOG MULTI-AGENT (LangGraph)                  │ │
│  │                                                             │ │
│  │  ┌─────────┐    ┌──────────────┐                           │ │
│  │  │ WATCHER │ -> │ INVESTIGATOR │                           │ │
│  │  └─────────┘    └──────┬───────┘                           │ │
│  │                        │                                    │ │
│  │       ┌────────────────┴────────────────┐                  │ │
│  │       ▼                                 ▼                  │ │
│  │  ┌─────────┐                      ┌──────────┐             │ │
│  │  │MITIGATOR│ ─── alerts/actions ──│ ALERTER  │             │ │
│  │  └─────────┘                      └──────────┘             │ │
│  │                                                             │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ LAYER 4 — EVAL ENGINE                                       │ │
│  │ RAGAS │ custom LLM-as-judge │ drift detection               │ │
│  │ Sample selection: 10% traffic + 100% suspicious             │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ LAYER 5 — API + DASHBOARD                                   │ │
│  │ REST API (incidents, agents, configs)                       │ │
│  │ Live dashboard (minimal admin UI)                           │ │
│  │ MCP Server (expose Watchdog as a tool)                      │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ LAYER 6 — AUTH + MULTI-TENANT                               │ │
│  │ JWT │ RBAC │ strict tenant isolation                        │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ LAYER 7 — MITIGATION OUTBOUND                               │ │
│  │ Kill switch (callback to monitored agent for isolation)     │ │
│  │ Webhooks (Slack, email, Teams)                              │ │
│  │ Rule injection (block patterns, throttle, etc.)             │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ LAYER 8 — META-OBSERVABILITY                                │ │
│  │ Langfuse self-tracing (Watchdog traces itself)              │ │
│  │ Recursive eval feedback loop                                │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Code structure

```
watchdog_ai/
├── ingestion/        # Layer 1 — telemetry receive
│   ├── api.py
│   ├── validation.py
│   └── queue.py
│
├── storage/          # Layer 2 — persistence
│   ├── models.py     # SQLAlchemy: Event, Incident, Agent, Tenant
│   ├── repositories.py
│   └── migrations/
│
├── agents/           # Layer 3 — multi-agent LangGraph
│   ├── watcher.py
│   ├── investigator.py
│   ├── mitigator.py
│   ├── alerter.py
│   ├── orchestrator.py    # The LangGraph chaining them
│   └── prompts/
│
├── eval/             # Layer 4 — evaluation engine
│   ├── ragas_eval.py
│   ├── llm_judge.py
│   ├── drift_detector.py
│   └── sampler.py
│
├── api/              # Layer 5 — API + dashboard
│   ├── routes/
│   ├── dashboard/    # static minimal admin UI
│   └── mcp_server.py
│
├── auth/             # Layer 6 — security
│   ├── jwt.py
│   ├── rbac.py
│   └── tenant_isolation.py
│
├── mitigation/       # Layer 7 — outbound actions
│   ├── kill_switch.py
│   ├── webhooks.py
│   └── rules.py
│
├── meta/             # Layer 8 — auto-observability
│   ├── self_tracing.py
│   └── recursive_eval.py
│
├── tests/            # Per-layer tests
├── infra/            # Docker, compose, CI/CD, Azure deploy
└── docs/             # ADRs, C4, RFCs
```

**Conventions**:
- 1 file = 1 responsibility, ~200 lines max
- Type hints everywhere (Python, pyright strict mode)
- No obvious comments — code documents itself through naming
- Conventional commits

---

## 6. Scenarios (implementation order)

### MVP — 3 months

#### S1 — Connect a first agent and receive its telemetry
- **Trigger**: an agent in production makes an LLM call
- **Steps**: agent → POST /api/v1/events → validation → PostgreSQL persistence
- **Done when**: an agent can send 1000+ events/hour, all persisted without loss
- **Out of scope S1**: auth, anomaly detection, multi-tenant

#### S2 — Detect a simple anomaly (rule-based cost spike)
- **Trigger**: cost event > 3× rolling 1h average
- **Steps**: Watcher reads metrics → computes moving average → compares threshold → creates Incident
- **Done when**: a cost spike anomaly detected and stored as an Incident in DB in under 60s
- **Out of scope S2**: other anomaly types, investigation

#### S3 — Automatically investigate an anomaly
- **Trigger**: new Incident created
- **Steps**: Investigator (LangGraph) → reads agent traces → identifies pattern → forms cause hypothesis → enriches Incident
- **Done when**: every Incident has an `investigation` field with probable cause + 3-5 sample events + structured summary
- **Out of scope S3**: RAG over past incidents (comes in S11)

#### S4 — Propose a mitigation
- **Trigger**: investigated Incident
- **Steps**: Mitigator → analyzes the investigation → proposes 1-3 actions (`isolate`, `fallback_model`, `block_pattern`, `throttle`) with rationale
- **Done when**: every investigated Incident has an actionable `proposed_mitigations[]` field
- **Out of scope S4**: automatic execution (proposal-only at this stage)

#### S5 — Alert humans with structured context
- **Trigger**: mitigation proposed
- **Steps**: Alerter → formats structured JSON alert → sends Slack/email webhook
- **Done when**: a human receives a Slack message with: agent_id, anomaly type, probable cause, proposed mitigations, dashboard link
- **Out of scope S5**: rich dashboard

#### S6 — Multi-tenant + auth
- **Trigger**: feature gate
- **Steps**: JWT auth → tenant_id on every event/incident → query isolation
- **Done when**: 2 tenants can send telemetry, never see each other, full audit trail

---

### Full Version — months 4-9

#### S7 — Automatic output quality evaluation
- Sample selection: 10% traffic + 100% suspicious events
- RAGAS metrics (faithfulness, answer relevance, context precision)
- Custom LLM-as-judge (relevance, safety, factuality, coherence)
- Score stored per event, aggregated per agent

#### S8 — Kill switch
- API endpoint `/api/v1/agents/{id}/isolate`
- Callback registered by Monitored Agents → when `isolate` is called, the monitored agent must refuse all new calls
- Strict audit log (who triggered, when, why)

#### S9 — Semantic drift detection
- Embeddings on inputs/outputs over rolling windows
- Distribution shift detection
- Alert on significant drift (KS test on embeddings)

#### S10 — Prompt injection detection
- Classifier (LLM-as-judge) on incoming inputs
- Pattern matching against known templates (jailbreak prompts)
- Risk score per event

#### S11 — RAG over incident history (learning)
- pgvector embeddings of past incidents stored alongside relational data in PostgreSQL (same instance, single source of truth)
- The Investigator reads the K most similar before forming its hypothesis
- The Mitigator looks at what worked historically
- Hybrid query in single SQL: tenant filter + time window + vector similarity, atomic

#### S12 — MCP server
- Watchdog exposes itself as an MCP server
- Exposed tools: `query_incidents`, `get_agent_status`, `request_eval`, `trigger_kill_switch`
- Other agents can query it

#### S13 — Meta-tracing (Watchdog self-monitors)
- Langfuse integrated for Watchdog itself
- Recursive eval: human confirmation of incidents → ground truth → re-evaluation of past detections
- Metrics: detection precision/recall, false positives/negatives

---

## 7. Tech stack

| Layer | Tech | Justification |
|---|---|---|
| **Backend API** | FastAPI + Pydantic v2 | 2026 standard, async, auto-OpenAPI, type-safe |
| **Relational DB** | PostgreSQL 16 | ACID, mature, JSONB for event payloads |
| **Time-series DB** | TimescaleDB extension on PG | Native time-series on PG, no separate infra needed |
| **Vector search** | pgvector (Postgres extension) | Same Postgres instance — zero extra infrastructure. HNSW index handles ≤ 1M vectors comfortably. Hybrid queries (tenant filter + time window + vector similarity) become atomic single-SQL statements |
| **Multi-agent** | LangGraph | 2026 production standard, first-class state, MCP-ready |
| **LLM provider** | Azure OpenAI (primary) + Mistral (fallback) | EU data sovereignty alignment |
| **Embeddings** | text-embedding-3-large (Azure) | Standard, multilingual support |
| **Eval frameworks** | RAGAS + custom LLM-as-judge | RAGAS for RAG, custom for open-ended outputs |
| **Self-observability** | Langfuse self-hosted | Open-source, full data ownership |
| **Auth** | JWT + RBAC | Standard, enterprise SSO-ready |
| **Queue** | In-memory (MVP) → Redis (Full) | Simple first, scale later |
| **Containerization** | Docker | Mandatory for portability |
| **Orchestration** | Docker Compose (dev) → Azure Container Apps (prod) | No K8s — overkill at this scale |
| **CI/CD** | GitHub Actions | Standard |
| **Hosting** | Azure | Cost-aligned with target customers |
| **MCP** | mcp-python SDK | Official Anthropic standard |

---

## 8. Roadmap

### MVP — Months 1-3

- **Month 1**: S1 + S2 — ingestion + rule-based cost spike detection
- **Month 2**: S3 + S4 + S5 — multi-agent investigation/mitigation/alerting
- **Month 3**: S6 — multi-tenant auth + **public live demo**

**MVP success criterion**: a demo where a fictional agent (simulating LLM calls with a cost spike) is connected, Watchdog detects in under 60s, investigates, proposes mitigation, alerts Slack. Publicly accessible.

### Full Version — Months 4-9

- **Month 4**: S7 — automatic RAGAS + LLM-as-judge evaluation
- **Month 5**: S8 + S10 — kill switch + prompt injection detection
- **Month 6**: S9 — semantic drift
- **Month 7**: S11 — incident RAG
- **Month 8**: S12 — MCP server
- **Month 9**: S13 — meta-tracing + finalization (C4 / ADRs / RFCs / docs)

---

## 9. Non-goals

| Non-goal | Why |
|---|---|
| Not a generic APM | Not Datadog/New Relic. AI-only focus. |
| Not an LLM Gateway | No routing, no caching. Only observability + safety. |
| Not a standalone eval framework | Eval is a module *of* Watchdog, not a separate product. |
| No support for non-API models (embedded device monitoring) | Out of scope, requires specialized infrastructure. |
| No traditional ML training monitoring | Focus on LLM/agents in inference, not training. |
| Not a multi-million-user SaaS at MVP | Multi-tenant architecture yes, but not optimized for > 1000 tenants at this stage. |
| No support for non-LangGraph agents at MVP | Phase 2+ if demand. MVP: LangGraph agents + RAG via official SDK only. |
| No rich custom UI | Minimal dashboard suffices. Rich interfaces come later if needed. |
| No mobile app | Not relevant. |

---

## 10. Technical metrics

| Metric | MVP target | Full target |
|---|---|---|
| Time-to-detect anomaly | < 60s | < 30s |
| Time-to-investigate | < 5min | < 2min |
| False positives | < 30% | < 10% (via recursive eval) |
| False negatives | < 50% | < 15% |
| Eval cost / monitored agent cost | < 30% | < 15% |
| Kill switch latency | < 10s | < 5s |
| Self-trace coverage | 50% | 100% |

---

## 11. 90-second demo

**Setup**: 1 live dashboard open, 1 simulated monitored agent running, Watchdog in the background.

**Script (90s)**:
1. *(15s)* "Here's an AI agent in production automating customer support. It makes 50K LLM calls/day. Without Watchdog, if the agent drifts, we only notice on the monthly bill or via a customer complaint."
2. *(30s)* "I trigger a cost-spike simulation (an input that forces long outputs). Watch the dashboard."
3. *(15s)* "The Watcher detected it in 45s. The Investigator read the traces, identified the probable cause: a user spamming long prompts. The Mitigator proposes: throttle this user + temporary isolation."
4. *(15s)* "I click 'approve' on the mitigation. The Alerter notifies Slack. The agent keeps serving other users normally."
5. *(15s)* "All of this with a multi-agent LangGraph system, automatic RAGAS eval on outputs, RAG over past incidents, MCP server for integration with other agents, deployed on Azure. Happy to walk through any module in detail."

---

## 12. Why now (market signals 2026)

Fresh sources (April 2026):
- **Datadog State of AI Engineering 2026**: 89% of agentic scaling failures = monitoring gap
- **Gartner**: 40%+ of agentic AI projects will fail by end of 2027 (escalating costs, unclear value, insufficient risk controls)
- **650-company survey (March 2026)**: 78% have agentic pilots, **only 14% scale to production**
- **Confident AI / Galileo / Maxim**: explicit gap identified — all observability tools "stop at logging", none do quality eval
- **35% of executives admit they cannot unplug a rogue agent**
- **MCP donated to Linux Foundation by Anthropic** (Dec 2025) — agent interoperability is becoming standard
- **Anthropic + OpenAI + Block co-founded the Agentic AI Foundation** — agents = officially recognized paradigm

→ Watchdog targets **the most structural pain point of 2026 and beyond**. Relevance guaranteed for 5+ years.

---

## 13. Differentiation vs existing tools

| Competitor | What they do | Gap Watchdog fills |
|---|---|---|
| Langfuse | Tracing + prompt management | No real-time eval, no automatic mitigation, no kill switch |
| LangSmith | Best-in-class tracing for LangChain/LangGraph | Same as Langfuse; no multi-agent investigator |
| Helicone | Gateway proxy + cost tracking | No eval, no investigation, no safety |
| Datadog APM + LLM Observability | Infra monitoring + LLM logs | No quality eval, no mitigation, no multi-agent investigation |
| Galileo / Confident AI | Eval framework + monitoring | Not autonomous (humans must investigate + mitigate), no kill switch |
| **Watchdog** | **Autonomous multi-agent that watches + investigates + mitigates + safety** | The differentiator — closing the gap. |

---

## License

Proprietary — see [LICENSE](LICENSE) (or contact author).
