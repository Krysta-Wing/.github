# Krysta Wing

**Execution infrastructure for AI agents.**

Run agent-generated code safely. Stream output live. Validate before it propagates.

---

## What We Are

Krysta Wing is an independent engineering org building the execution and observability layer for AI agent systems. We are not an MLOps platform. We are not a monitoring dashboard. We are the infrastructure that sits between an AI agent and the damage it could do — making sure code runs safely, output streams in real time, and nothing broken propagates to the next step.

---

## The Problem We Solve

AI agents write and execute code. That code needs to run somewhere. Every existing option forces a tradeoff:

- **E2B / Modal** — request/response black boxes. You submit code, you wait, you get a dump. If the agent writes an infinite loop or a long script, your UI freezes. No visibility until it's over.
- **Raw Docker containers** — no isolation guarantees, no output validation, no trace of what happened.
- **Nothing** — most teams just let the agent execute code directly. One bad generation away from a disaster.

We solve the visibility and safety gap. Developers get live-streamed stdout as it happens, a full execution trace logged automatically, and a validation gate that checks the output before it moves downstream.

---

## The Three Primitives

Every product we build maps to one of three things:

```
Execute    →    Observe    →    Validate
  ↑                               ↓
claw-daemon          kwing-library + gate.py
```

**Execute** — run untrusted agent-generated code in a sandboxed environment, stream stdout live via SSE.

**Observe** — log every execution event (stdout lines, timing, memory, errors) as a structured trace tied to a jobId.

**Validate** — check the execution output against rules before passing it to the next agent step. Block if it fails.

---

## The Developer API

This is what a developer using Krysta Wing writes:

```python
from krysta import Claw

async with Claw.spawn(runtime="python", timeout=10) as sandbox:

    # live streaming — see output as it happens
    async for chunk in sandbox.execute(agent_generated_code):
        print(chunk.stdout)

    # validate before passing downstream
    result = sandbox.validate(rules=["valid_json", "no_network_calls"])

    # full execution trace
    trace = sandbox.trace()
```

Three methods. That is the entire surface area a developer needs to think about.

---

## Repository Map

We operate across two domains. Understand what each repo owns before touching it.

### `kwing-claw` — Execution Infrastructure
**Domain:** `kwing-claw` (private)
**Language:** Node.js
**Status:** Live

The core execution engine. This is the hardest and most critical piece of the stack. Do not modify without understanding the full event flow.

```
kwing-claw/
├── src/
│   ├── pages/
│   │   └── api/
│   │       ├── submit.js     # POST /execute — accepts payload, pushes to Kafka, returns jobId + 202
│   │       └── stream.js     # GET /stream — SSE pipe, subscribes to Redis logs:jobId channel
│   └── infrastructure/
│       └── claw-daemon.js    # Kafka consumer → child_process.spawn() → stdout piped to Redis pub/sub
├── docker-compose.yml        # spins up kwing-kafka + kwing-redis locally
└── package.json
```

**How it works end to end:**

```
POST /execute
    → Kafka (code-submissions topic, numPartitions: 3)
        → claw-daemon consumes job
            → child_process.spawn(runtime, code)
                → stdout/stderr piped to Redis logs:{jobId}
                    → GET /stream SSE connection reads Redis channel
                        → browser receives live chunks
```

**Running locally:**
```bash
docker compose up -d          # start Kafka + Redis
npm run daemon                # start worker pool (open 3 terminals for horizontal scaling)
npm run dev                   # start Next.js gateway at localhost:3000
```

**Horizontal scaling:** The Kafka topic runs 3 partitions. Spin up 3 separate daemon processes and Kafka will rebalance automatically, distributing jobs round-robin. If one daemon dies mid-execution, Kafka reroutes to surviving workers.

**Payload schema (POST /execute):**
```json
{
  "runtime": "python3",
  "code": "print('hello')",
  "timeout_ms": 5000
}
```

---

### `krysta-wing` (kwing-library) — Execution Tracer + Telemetry
**Domain:** `kwing-library` (PyPI: `pip install krysta`)
**Language:** Python
**Status:** Live

The observability layer. Originally built as an ML telemetry tracker. Now repurposed as the execution trace engine — every job that runs through claw gets logged here as a structured event stream.

```
krysta_eval/
├── __init__.py
├── schema.py              # frozen JSON report schema — do not modify without a team discussion
├── cli.py                 # terminal interface: krysta-eval run / krysta-eval diff
├── core/
│   ├── engine.py          # main evaluation lifecycle orchestrator
│   ├── baseline.py        # sliding-window μ ± 2σ statistical manager
│   └── gate.py            # threshold + regression pass/fail logic (exit code 1 on fail)
├── sdk/
│   └── client.py          # unified kwing-sdk abstraction wrapper
├── plugins/
│   ├── base.py            # abstract plugin interface — all plugins implement this
│   ├── rag_plugin.py      # RAG pipeline hook (structural, metrics incomplete)
│   └── rag_metrics.py     # 🛑 INCOMPLETE — math functions, tracked in GitHub Issue
└── runners/
    ├── base.py            # abstract runner interface
    └── torch_runner.py    # 🛑 INCOMPLETE — tracked in GitHub Issue
```

**Quick start:**
```python
from krysta_reporter import ModelReport

reporter = ModelReport(week=22, model_name="ResNet50-XAI", modality="hybrid-omni")
reporter.metrics = {
    "latency": 14.2,
    "vram": 3120.0,
    "loss": 0.042
}
reporter.compile()  # outputs structured JSON + markdown report
```

**Config (krysta_config.yaml):**
```yaml
workspace_root: "production_reports"
thresholds:
  token_confidence: 0.85
  latency_limit_ms: 50.0
```

---

## Architecture: Full System Map

```
┌─────────────────────────────────────────────────────────┐
│                    CONSUMER LAYER                       │
│   AI Agent (generates code)    Developer / CI pipeline  │
│                    kwing-sdk (pip install krysta)         │
└──────────────────────┬──────────────────────────────────┘
                       │ POST /execute
┌──────────────────────▼──────────────────────────────────┐
│                    GATEWAY LAYER                        │
│         api-gateway — Next.js deployed on Vercel        │
│   POST /execute → jobId + 202  |  GET /stream → SSE     │
└──────────┬──────────────────────────────────────────────┘
           │                              ▲ SSE live stdout
┌──────────▼──────────┐    ┌─────────────┴──────────────┐
│    Apache Kafka      │    │      Redis pub/sub          │
│  immutable job queue │    │   logs:jobId channel        │
│  backpressure buffer │    │   zero-lag streaming        │
└──────────┬──────────┘    └─────────────▲──────────────┘
           │                             │ publish stdout
┌──────────▼─────────────────────────────┴──────────────┐
│               EXECUTION LAYER (claw-daemon)             │
│   consumes Kafka → child_process.spawn() → pipe Redis   │
│   worker pool, 3 partitions, hot-failover rebalance     │
└──────────────────────┬──────────────────────────────────┘
                       │ every event logged
┌──────────────────────▼──────────────────────────────────┐
│              OBSERVE LAYER (kwing-library)               │
│   execution tracer — logs stdout, timing, memory, errors │
│   baseline manager — μ ± 2σ sliding window profiles     │
└──────────────────────┬──────────────────────────────────┘
                       │ report JSON
┌──────────────────────▼──────────────────────────────────┐
│              VALIDATE LAYER (kwing-eval)                 │
│   gate.py — threshold + regression checks               │
│   sandbox.validate() — rule engine (no_network, etc.)   │
│   PASS → exit 0 → next agent step                       │
│   FAIL → exit 1 → block + structured report             │
└─────────────────────────────────────────────────────────┘
```

---

## What Is Being Built Next

In priority order:

**1. Python SDK (`kwing/claw.py`)**
The public-facing wrapper. Exposes `Claw.spawn()`, `sandbox.execute()`, `sandbox.validate()`, `sandbox.trace()`. Hits the Vercel gateway internally. This is what external developers install.

**2. Execution tracer bridge**
Connect kwing-library's `ModelReport.compile()` output directly to the claw-daemon job lifecycle. Every `jobId` gets a trace file automatically.

**3. `sandbox.validate()` rule engine**
Build on top of existing `gate.py`. Accepts a list of named rules (`valid_json`, `no_network_calls`, `output_under_limit`) and returns pass/fail with a structured reason.

**4. `torch_runner.py` and `rag_metrics.py`**
Both tracked as open GitHub Issues. Contributors welcome on these — see labels below.

**5. First public technical report**
Run kwing against a real agent workflow, publish findings on the Krysta Wing site. This is our credibility moment.

---

## GitHub Labels and Contribution Guide

We split work across three access tiers:

| Label | Meaning | Who |
|---|---|---|
| `🧱 layer:core` | Engine, schema, baseline logic | Core team only |
| `🏃 layer:runner` | Model/runtime framework connectors | Contributors welcome |
| `🔌 layer:plugin` | Validators, formatters, metric plugins | Contributors welcome |
| `⚠️ status:blocker` | Breaking the execution loop | Prioritize immediately |
| `🎯 scope:mvp` | Required before first public release | Must ship before anything else |

**Before opening a PR:**
- Do not modify `schema.py` without a team discussion — it is the data contract everything else depends on
- Do not modify `claw-daemon.js` without understanding the full Kafka → Redis → SSE flow
- All new runners must extend `runners/base.py`
- All new plugins must extend `plugins/base.py`

---

## Open Issues (Unblocked, Contributor-Ready)

**Issue #1 — `torch_runner.py`** (`🏃 layer:runner` `🎯 scope:mvp`)
Implement the PyTorch model runner extending `runners/base.py`. Must implement `load()`, `infer()`, and `teardown()`. See `runners/base.py` for the interface contract.

**Issue #2 — `rag_metrics.py`** (`🔌 layer:plugin` `🎯 scope:mvp`)
Implement `context_relevance()`, `faithfulness()`, and `hallucination_rate()` math functions. Use `sentence-transformers` for cosine similarity. See `plugins/rag_plugin.py` for how these will be called.

---

## Identity and Positioning

**Krysta Wing** is an independent engineering collective. We build infrastructure, not wrappers. We publish original technical findings, not paper summaries.

We are not:
- An MLOps dashboard
- A model fine-tuning service
- A consulting firm
- An AI wrapper

We are the execution layer that makes agent-generated code safe to run in production.

---

## Contact and Links

- PyPI: `pip install krysta`
- Domains: `kwing-claw` · `kwing-library`
- GitHub org: Krysta Wing (private repos — request access from core team)

---

*Internal document — Krysta Wing Engineering. Last updated June 2026.*
