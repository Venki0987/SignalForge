# Layer 11: Observability & Tracing

## Component
- Observability Layer (LangSmith / OpenTelemetry)

---

## What it does
Monitors the health, cost, performance, and quality of every single node in the pipeline. Provides full DAG execution traces for debugging failures and optimizing performance.

## Why This Layer is Critical
With 6+ LLM agents, multiple external APIs, and complex routing logic, debugging production failures without observability is pure guesswork. This layer answers: "What went wrong, where, and why?"

---

## Implementation Plan

### Integration Approach
- **LangSmith:** LangGraph natively supports LangSmith tracing. Every node execution automatically logs inputs, outputs, and metadata.
- **OpenTelemetry (alternative/complement):** For custom metrics, latency histograms, and integration with existing infrastructure monitoring (Grafana, Datadog).

### What Gets Tracked (Per Node)

| Metric | Description |
|--------|-------------|
| Token Usage | Input + output tokens per LLM call |
| Latency (ms) | Time from node entry to node exit |
| Cost ($) | Calculated from token usage × model pricing |
| Success/Failure | Did the node complete without error? |
| Full Prompt + Completion | For debugging (stored, not displayed in dashboards) |
| Retry Count | How many retries were needed for external API calls |
| Model Used | Which model handled this specific invocation |

### Aggregate Metrics (Per Run)

| Metric | What It Tells You |
|--------|-------------------|
| Rejection Rate by Class | Are Class 2 stories always getting rejected? → Writer prompt needs tuning |
| Average Retry Count | How many rewrites before approval? (Target: < 1.5) |
| DLQ Frequency | How many stories hit Dead Letter Queue per run? (Target: < 2) |
| End-to-End Duration | Total time from CRON trigger to publishing |
| Cost Per Story | Average $ spent per newsletter section |
| Cost Per Run | Total weekly pipeline cost |
| API Failure Rates by Source | Is ArXiv unreliable on Mondays? Is Tavily rate-limiting us? |
| Dedup Hit Rate | What % of signals are duplicates? (Validates the dedup threshold) |

### Alerting Rules

| Condition | Action |
|-----------|--------|
| Rejection rate > 60% in a single run | Slack/email alert — likely a systemic prompt issue |
| DLQ receives > 3 stories in one run | Alert — something is fundamentally wrong with source quality or Writer |
| Any single node latency > 120s | Alert — possible API timeout or model overload |
| Total run cost exceeds budget threshold | Alert — unexpected token burn |
| External API failure rate > 50% | Alert — source is likely down |

### Dashboard Views

**1. Run Overview**
- Timeline visualization of the current/last run
- Status of each story: in-progress, approved, rejected, escalated
- Total cost and duration

**2. Node Performance**
- Per-node latency histograms
- Token usage trends over time
- Error rates by node

**3. Quality Metrics**
- Reviewer rejection reasons (aggregated)
- Most common hallucination types
- HITL edit frequency (how often do humans change the AI output?)

**4. Cost Analysis**
- Cost breakdown by model
- Cost per class (which types of stories are most expensive to produce?)
- Trend over weeks

### Implementation Details
- LangSmith: Set `LANGCHAIN_TRACING_V2=true` and `LANGCHAIN_API_KEY` in environment. LangGraph automatically traces.
- Custom spans: Wrap non-LLM nodes (dedup layer, FAISS operations) in OpenTelemetry spans manually.
- Storage: LangSmith handles trace storage. For OpenTelemetry, export to Jaeger or Grafana Tempo.

### Key Goal
Full visibility into every pipeline execution. When something breaks, you should be able to trace the exact failure point in under 2 minutes. When costs spike, you should know which model or which class of stories is responsible. This layer doesn't touch the main pipeline — it's purely observational, reading state at each node transition.
