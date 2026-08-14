# SignalForge

**An autonomous research-to-newsletter pipeline built on LangGraph.** Ingests from RSS and the open web, deduplicates, classifies, enriches, drafts, self-verifies, and delivers — with human-in-the-loop approval gates and a full guardrail layer.

Built as a study in *production* agentic architecture: not a demo chain, but a stateful graph with explicit failure modes, fallbacks, and observability.

---

## Why this exists

Most "AI newsletter" projects are a single LLM call wrapped in a cron job. They break silently, hallucinate confidently, and duplicate content. SignalForge treats the pipeline as a **distributed system with an LLM in it** — every stage has a defined contract, a failure mode, and a fallback.

## Architecture

```
                          ┌──────────────────┐
   RSS / Web / PDF  ──────►   Ingestion      │  scheduled + event triggers
                          └────────┬─────────┘
                                   ▼
                          ┌──────────────────┐
                          │  Deduplication   │  embedding similarity + URL canonicalisation
                          └────────┬─────────┘
                                   ▼
                          ┌──────────────────┐
                          │    Filtering     │  relevance threshold, drops low-signal items
                          └────────┬─────────┘
                                   ▼
                          ┌──────────────────┐
                          │  Classification  │  topic routing → downstream specialist
                          │   and Routing    │
                          └────────┬─────────┘
                                   ▼
                          ┌──────────────────┐
                          │    Enrichment    │  full-text fetch, context gathering
                          └────────┬─────────┘
                                   ▼
                          ┌──────────────────┐
                          │    Generation    │  drafting agent
                          └────────┬─────────┘
                                   ▼
                          ┌──────────────────┐
                          │ Verification     │◄─┐  groundedness check
                          │      Loop        │  │  retry on failure
                          └────────┬─────────┘──┘
                                   ▼
                          ┌──────────────────┐
                          │ Human Oversight  │  approval gate before send
                          └────────┬─────────┘
                                   ▼
                          ┌──────────────────┐
                          │    Delivery      │  + interactive chatbot over archive
                          └──────────────────┘

        Cross-cutting: Guardrails · State Management · Observability · Fallbacks
```

## Agents

| Agent | Responsibility |
|-------|----------------|
| `data_search` | Sources candidate articles from RSS feeds and the open web |
| `dedup` | Rejects near-duplicates via embedding similarity before they cost tokens |
| `classifier` | Assigns topic categories and routes to the right downstream handler |
| `curator` | Selects and orders the final item set for the issue |
| `reviewer` | Verifies generated copy is grounded in source material; triggers retry |
| `chatbot` | Conversational retrieval over the published archive |

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Orchestration | **LangGraph** (stateful graph, conditional edges, retry loops) |
| LLM Framework | LangChain, LangChain Core |
| Models | **AWS Bedrock**, Google Gemini, OpenAI-compatible endpoints |
| Vector Store | ChromaDB |
| API | FastAPI + Uvicorn |
| Ingestion | feedparser, trafilatura, pypdf, requests |
| Tokenisation | tiktoken |

## Design Documentation

This repo ships **15 design documents** under [`docs/`](docs/) — written before and alongside the implementation:

| Doc | Covers |
|-----|--------|
| [`00-langgraph-architecture`](docs/00-langgraph-architecture) | Graph topology, node contracts, state shape |
| [`01-ingestion-and-triggering`](docs/01-ingestion-and-triggering) | Scheduled vs event-driven entry points |
| [`02-deduplication`](docs/02-deduplication) | Similarity thresholds, canonicalisation strategy |
| [`03-filtering`](docs/03-filtering) | Relevance scoring and cutoffs |
| [`04-classification-and-routing`](docs/04-classification-and-routing) | Topic taxonomy, conditional routing |
| [`05-enrichment`](docs/05-enrichment) | Full-text extraction, context assembly |
| [`06-generation`](docs/06-generation) | Prompt design, output structure |
| [`07-verification-loop`](docs/07-verification-loop) | Groundedness checks, retry semantics |
| [`08-human-oversight`](docs/08-human-oversight) | Approval gates, override paths |
| [`09-storage`](docs/09-storage) | Persistence model |
| [`10-delivery-and-interaction`](docs/10-delivery-and-interaction) | Send pipeline, archive chat |
| [`11-observability`](docs/11-observability) | Tracing, metrics, what gets logged |
| [`12-guardrails`](docs/12-guardrails) | Input/output safety controls |
| [`12-failure-points-and-fallbacks`](docs/12-failure-points-and-fallbacks) | Every failure mode and its fallback |
| [`13-state`](docs/13-state) | State transitions across the graph |

## Engineering Notes

- **Dedup runs before enrichment** — rejecting duplicates early avoids paying for full-text fetch and LLM tokens on content already covered.
- **The verification loop is a cycle, not a check** — the reviewer agent can send work back to generation with feedback, bounded by a retry ceiling to prevent infinite loops.
- **Human oversight is a graph node, not a wrapper** — approval state lives in the graph, so a rejected issue resumes at the right stage instead of restarting.
- **Every stage has a documented fallback** — see `12-failure-points-and-fallbacks`. Model unavailability, empty retrieval, and extraction failure all degrade rather than crash.

## Getting Started

```bash
cd ra_bend
pip install -r requirements.txt
cp .env.example .env      # add model + API credentials
uvicorn main:app --reload --port 8000
```

Model credentials required for whichever provider you enable (Bedrock, Gemini, or an OpenAI-compatible endpoint). See `config.py` for the full settings surface.

## Author

**NagaVenkatesh Arigala** — AI/GenAI Engineer
[LinkedIn](https://www.linkedin.com/in/nv-arigala0801/) · [GitHub](https://github.com/Venki0987)

## License

MIT — see [LICENSE](LICENSE).

---

## 📂 About this repository

This is a **documentation and architecture showcase**. It covers the problem, the system design, the agent topology, and the engineering decisions behind the project.

**The source code is held in a private repository.** I'm glad to walk through the implementation in a technical conversation or screen-share — reach out via the links below.

All rights reserved — see [LICENSE](LICENSE).