# 🔭 AgentLens

> Real-time observability for LangGraph agents. Drop-in tracing, cost attribution, and a visual dashboard — in under 5 minutes.

[![PyPI version](https://badge.fury.io/py/agentlens.svg)](https://badge.fury.io/py/agentlens)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.9+](https://img.shields.io/badge/python-3.9+-blue.svg)](https://www.python.org/downloads/)

---

## Why AgentLens?

Debugging a LangGraph agent in production looks like this:

```
Task failed. Reason: unknown. Cost: ??? Tokens used: ???
```

AgentLens makes it look like this:

```
✓ research_agent          127ms   $0.0043   4 tool calls
  ├─ web_search            34ms   $0.0012   1 call
  ├─ summarize             61ms   $0.0024   1 call  ← most expensive
  └─ format_output         32ms   $0.0007   2 calls
❌ write_report             --     $0.0091   FAILED at node: citation_check
   └─ Error: context_length_exceeded after 3,847 tokens
```

No vendor lock-in. No data leaving your machine. No config files.

---

## Install

```bash
pip install agentlens
```

---

## Quick Start (3 lines)

```python
from agentlens import trace
from langgraph.graph import StateGraph

# Wrap your existing graph — nothing else changes
graph = trace(StateGraph(MyState))

# ...build your graph as normal...
app = graph.compile()
result = app.invoke({"query": "What is the capital of France?"})

# That's it. Open http://localhost:7842 to see the trace.
```

---

## What You Get

### 1. Run Timeline
Every node, every tool call, every LLM invocation — in the order they happened, with timing and token counts.

```
Run #42 | research_pipeline | 1.3s | $0.018 | ✓ SUCCESS
├─ [0ms]  retrieve_context     45ms   512 tokens in   │ ✓
├─ [45ms] classify_intent      23ms   128 tokens in   │ ✓
├─ [68ms] web_search (tool)    312ms  —               │ ✓  → 3 results
├─ [380ms] synthesize          510ms  1024 tokens in  │ ✓  2048 tokens out
└─ [890ms] format_response     410ms  256 tokens in   │ ✓
```

### 2. Cost Attribution
Understand exactly where your money is going — per node, per model, per workflow.

| Node | Model | Tokens In | Tokens Out | Cost |
|------|-------|-----------|------------|------|
| synthesize | gpt-4o | 1,024 | 2,048 | $0.0123 |
| classify_intent | gpt-4o-mini | 128 | 32 | $0.0002 |
| format_response | gpt-4o-mini | 256 | 512 | $0.0004 |
| **Total** | | **1,408** | **2,592** | **$0.0129** |

### 3. Error Forensics
When a node fails, AgentLens captures the full state at the point of failure — not just the exception.

```python
# State snapshot at failure:
{
  "node": "citation_check",
  "error": "context_length_exceeded",
  "state_at_failure": {
    "messages": [...],          # full message history
    "accumulated_context": 3847, # tokens at time of failure
    "last_tool_result": {...}
  },
  "retry_count": 2
}
```

### 4. Visual Dashboard

```bash
agentlens dashboard
# → http://localhost:7842
```

Real-time run list, interactive trace viewer, cost-over-time charts, and error heatmaps. Built with FastAPI + React.

---

## Advanced Usage

### Multi-Agent Pipelines

AgentLens automatically traces cross-agent calls when you use it across your subgraphs:

```python
from agentlens import trace, AgentLensConfig

config = AgentLensConfig(
    project="logistics-pipeline",
    track_subgraphs=True,   # trace calls across agents
    cost_alerts={"per_run": 0.10}  # alert if a run exceeds $0.10
)

research_agent = trace(research_graph, config=config)
routing_agent  = trace(routing_graph, config=config)
```

All agents share a session ID, so you can visualize the full multi-agent interaction in one trace.

### Custom Metadata

Tag your runs with business context for filtering in the dashboard:

```python
app.invoke(
    {"query": "..."},
    config={"metadata": {
        "customer_id": "acme-corp",
        "workflow": "invoice-processing",
        "env": "production"
    }}
)
```

### Exporting Traces

```python
from agentlens import export_runs

# Export last 100 runs as JSON (for fine-tuning datasets, audits, etc.)
export_runs(limit=100, format="jsonl", output="traces.jsonl")
```

---

## Architecture

```
Your LangGraph App
       │
       ▼
  AgentLens Tracer  ◄── Callback handler wrapping LangChain's BaseTracer
       │
       ▼
  Event Store  ◄──────── SQLite (local) or Postgres (team/prod)
       │
       ▼
  FastAPI Backend  ◄──── REST API serving the dashboard
       │
       ▼
  React Dashboard  ◄──── Real-time UI via SSE (no WebSocket needed)
```

**Key design decisions:**
- **Zero-overhead tracing**: Callbacks are async and non-blocking. Your agent doesn't wait for AgentLens.
- **Local-first**: All data stays on your machine by default. SQLite works out of the box.
- **Framework-native**: Extends `BaseTracer` — works with anything LangChain supports.
- **No telemetry**: AgentLens never phones home. Ever.

---

## Roadmap

- [ ] LangGraph callback tracer
- [ ] SQLite event store
- [ ] FastAPI backend
- [ ] React dashboard (run list, timeline, cost breakdown)
- [ ] Postgres backend for teams
- [ ] Alert rules (cost, latency, error rate)
- [ ] Multi-agent session linking
- [ ] OpenTelemetry export
- [ ] AutoGen / CrewAI adapters

---

## Contributing

Pull requests welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for setup instructions.

```bash
git clone https://github.com/yourusername/agentlens
cd agentlens
pip install -e ".[dev]"
pytest
```

---

## License

MIT — use it, fork it, ship it.

---

*Built by a developer who got tired of `print(state)` debugging.*
