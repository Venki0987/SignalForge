# LangGraph Workflow Architecture

## Overview

This document defines the complete LangGraph DAG (Directed Acyclic Graph) architecture — the state schema, node definitions, edge connections, routing logic, and workflow patterns used across the pipeline.

---

## Workflow Patterns Used

| Pattern | Where Used | Why |
|---------|-----------|-----|
| **Sequential** | CRON → Scout → Dedup → Curator → Router | Linear dependency — each node needs the previous node's output |
| **Parallel (Fan-out / Fan-in)** | Data Web Search ∥ Writer | Data Search runs async alongside Writer to avoid blocking |
| **Router (Conditional)** | SLM Classifier → class-specific prompt selection | Routes state to different prompt templates based on classification |
| **Evaluator-Optimizer Loop** | Writer ↔ Reviewer | Iterative refinement — Reviewer rejects, Writer rewrites, up to 3 cycles |
| **Map-Reduce (Parallel per story)** | After Curator, each story is processed independently | Multiple stories processed in parallel through Enrichment → Generation → Verification |

---

## Global State Schema

The LangGraph state is the single shared object that flows through the entire DAG. Every node reads from and writes to this state.

```python
from typing import TypedDict, Optional, Literal
from typing_extensions import Annotated
from langgraph.graph import add_messages
import operator

class Signal(TypedDict):
    """A single raw news signal from the Scout."""
    signal_id: str
    source: Literal["arxiv", "rss", "twitter", "youtube", "web_search"]
    url: str
    title: str
    snippet: str
    raw_metadata: dict
    fetched_at: str
    related_sources: list[dict]  # Populated by dedup layer


class StoryState(TypedDict):
    """State for a single story being processed through the pipeline.
    After the Curator, each story gets its own sub-state and is processed independently."""
    
    # Identity
    story_id: str
    run_id: str
    
    # From Scout + Dedup
    signal: Signal
    
    # From Curator
    relevance_score: float
    curator_reasoning: str
    
    # From SLM Router
    news_class: int  # 1-5
    classification_confidence: float
    
    # From Sourcer Agent
    distilled_context: Optional[dict]
    # {
    #   "news_text": str,
    #   "paper_abstract": Optional[str],
    #   "paper_conclusion": Optional[str],
    #   "paper_key_figures": Optional[str],
    #   "transcript_summary": Optional[str],
    #   "source_urls": list[str],
    #   "raw_metadata": dict
    # }
    
    # From Data Web Search Agent (Class 1 & 5 only)
    data_search_results: Optional[dict]
    data_search_complete: bool  # Flag to know if async search finished
    
    # From Writer Agent
    writer_draft: Optional[str]  # Markdown
    writer_system_prompt: str  # The class-specific prompt used
    
    # From Reviewer Agent
    reviewer_status: Optional[Literal["APPROVED", "REJECTED"]]
    reviewer_reasoning: Optional[str]
    hallucinations_found: list[str]
    formatting_errors: list[str]
    retry_count: int  # Max 3
    
    # From HITL
    human_decision: Optional[Literal["approved", "edited", "rejected"]]
    human_edit: Optional[str]  # If edited, the final version
    editor_id: Optional[str]
    
    # Pipeline control
    status: Literal[
        "scouted", "deduplicated", "curated", "classified",
        "enriched", "drafted", "reviewing", "approved",
        "escalated", "published", "rejected_by_human"
    ]
    error_log: list[str]


class PipelineState(TypedDict):
    """Top-level state for the entire weekly run."""
    
    # Run metadata
    run_id: str
    triggered_at: str
    trigger_type: Literal["cron", "manual"]
    
    # Layer 1 output
    raw_signals: list[Signal]
    
    # Layer 2 output
    deduplicated_signals: list[Signal]
    
    # Layer 3 output
    curated_stories: list[StoryState]
    
    # Layer 4+ (each story processed independently)
    stories_in_progress: list[StoryState]
    stories_completed: list[StoryState]
    stories_escalated: list[StoryState]
    
    # Run-level metrics
    total_signals_fetched: int
    duplicates_removed: int
    stories_filtered_out: int
    total_cost_usd: float
    run_status: Literal["running", "awaiting_review", "published", "failed"]
```

---

## Node Definitions

Each node is a Python function (or agent invocation) that takes the state and returns a state update.

### Node Map

| Node ID | Node Name | Type | Model / Logic | Input From State | Writes To State |
|---------|-----------|------|---------------|------------------|-----------------|
| `scout` | Scout/Sourcer Agent | LLM Agent (tool-calling) | Gemma 4 27B | `run_id`, `triggered_at` | `raw_signals` |
| `dedup` | Deduplication Layer | Python Function | FAISS + MiniLM-L6-v2 | `raw_signals` | `deduplicated_signals`, `duplicates_removed` |
| `curator` | Curator Agent | LLM Agent | DeepSeek V4 | `deduplicated_signals` | `curated_stories` |
| `classifier` | SLM Classifier Router | LLM Agent | Qwen 3.5-9B | `story.signal` | `story.news_class`, `story.classification_confidence` |
| `prompt_selector` | Prompt Selector | Python Function | None (lookup table) | `story.news_class` | `story.writer_system_prompt` |
| `sourcer` | Sourcer/Enrichment Agent | LLM Agent (tool-calling) | Gemma 4 27B | `story.signal`, `story.news_class` | `story.distilled_context` |
| `data_search` | Data Web Search Agent | LLM Agent (tool-calling) | Llama 3.3 70B | `story.signal`, `story.news_class` | `story.data_search_results` |
| `writer` | Writer Agent | LLM Agent | Claude 4.5 Sonnet / Qwen 3 | `story.distilled_context`, `story.data_search_results`, `story.writer_system_prompt` | `story.writer_draft` |
| `reviewer` | Reviewer Agent | LLM Agent | DeepSeek R1 | `story.writer_draft`, `story.distilled_context`, `story.data_search_results`, `story.news_class` | `story.reviewer_status`, `story.hallucinations_found`, `story.formatting_errors` |
| `dlq` | Dead Letter Queue | Python Function | None | Full `story` state | `stories_escalated` (+ DB write + notification) |
| `hitl_gate` | Human-in-the-Loop | External Wait | None (human input) | `story` (approved by reviewer) | `story.human_decision`, `story.human_edit` |
| `publish` | Publisher | Python Function | None | `stories_completed` | PostgreSQL write + Vector DB chunking + frontend push |

---

## Graph Structure (Edge Definitions)

```
┌─────────────────────────────────────────────────────────────────────┐
│                         PIPELINE GRAPH                                │
│                                                                       │
│  START                                                                │
│    │                                                                  │
│    ▼                                                                  │
│  [scout] ─── Sequential ──→ [dedup] ─── Sequential ──→ [curator]    │
│                                                                       │
│    │ (curator outputs list of stories)                                │
│    │                                                                  │
│    ▼  MAP (Fan-out: each story processed in parallel)                │
│                                                                       │
│  ┌─────────────── PER-STORY SUBGRAPH ───────────────────┐           │
│  │                                                        │           │
│  │  [classifier] ─── Sequential ──→ [prompt_selector]    │           │
│  │       │                                │               │           │
│  │       │                                ▼               │           │
│  │       │                          [sourcer]             │           │
│  │       │                                │               │           │
│  │       ▼                                ▼               │           │
│  │  ┌─── Conditional ───┐          ┌─────────────┐      │           │
│  │  │ Class 1 or 5?     │          │             │      │           │
│  │  └──┬────────────┬───┘          │             │      │           │
│  │     │YES         │NO            │             │      │           │
│  │     ▼            │              │             │      │           │
│  │  [data_search]   │              │             │      │           │
│  │     │            │              │             │      │           │
│  │     │  (async)   │              │             │      │           │
│  │     └────────────┼──── Merge ───┼─────────────┘      │           │
│  │                   │              │                     │           │
│  │                   ▼              ▼                     │           │
│  │              [writer] ◄──────────┘                    │           │
│  │                   │                                    │           │
│  │                   ▼                                    │           │
│  │  ┌─────── Evaluator-Optimizer Loop ──────────┐       │           │
│  │  │                                            │       │           │
│  │  │  [reviewer]                                │       │           │
│  │  │       │                                    │       │           │
│  │  │       ▼                                    │       │           │
│  │  │  ┌─── Conditional ───────────────┐        │       │           │
│  │  │  │                               │        │       │           │
│  │  │  │ APPROVED    REJECTED          REJECTED  │       │           │
│  │  │  │             (retry<3)         (retry≥3) │       │           │
│  │  │  │                │                  │     │       │           │
│  │  │  │                ▼                  ▼     │       │           │
│  │  │  │          [writer] ◄─┐          [dlq]   │       │           │
│  │  │  │             │       │             │     │       │           │
│  │  │  │             ▼       │             │     │       │           │
│  │  │  │         [reviewer]──┘             │     │       │           │
│  │  │  │                                   │     │       │           │
│  │  │  └───────────────────────────────────┘     │       │           │
│  │  │                                            │       │           │
│  │  └────────────────────────────────────────────┘       │           │
│  │       │ (APPROVED)                                     │           │
│  │       ▼                                                │           │
│  │  [hitl_gate] ─── wait for human ──→ decision          │           │
│  │       │                                                │           │
│  │       ▼                                                │           │
│  │  ┌─── Conditional ───┐                                │           │
│  │  │ approved? edited? │                                │           │
│  │  │ rejected?         │                                │           │
│  │  └──┬──────┬─────┬──┘                                │           │
│  │     │      │     │                                    │           │
│  │     ▼      ▼     ▼                                    │           │
│  │  [done]  [done] [discard]                             │           │
│  │                                                        │           │
│  └────────────────────────────────────────────────────────┘           │
│                                                                       │
│    ▼  REDUCE (Fan-in: collect all completed stories)                 │
│                                                                       │
│  [publish] ─── Sequential ──→ END                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Edge Definitions (Code-Level)

```python
from langgraph.graph import StateGraph, END

# ═══════════════════════════════════════════
# MAIN PIPELINE GRAPH
# ═══════════════════════════════════════════

pipeline = StateGraph(PipelineState)

# --- Layer 1-3: Sequential ingestion ---
pipeline.add_node("scout", scout_agent)
pipeline.add_node("dedup", deduplication_function)
pipeline.add_node("curator", curator_agent)
pipeline.add_node("process_stories", process_stories_map)  # Fan-out
pipeline.add_node("publish", publish_function)

pipeline.add_edge("scout", "dedup")          # Sequential
pipeline.add_edge("dedup", "curator")         # Sequential
pipeline.add_edge("curator", "process_stories")  # Fan-out to per-story subgraph
pipeline.add_edge("process_stories", "publish")  # Fan-in after all stories done
pipeline.add_edge("publish", END)

pipeline.set_entry_point("scout")


# ═══════════════════════════════════════════
# PER-STORY SUBGRAPH (executed for each story)
# ═══════════════════════════════════════════

story_graph = StateGraph(StoryState)

# --- Layer 4: Classification ---
story_graph.add_node("classifier", slm_classifier)
story_graph.add_node("prompt_selector", select_prompt_by_class)

# --- Layer 5: Enrichment ---
story_graph.add_node("sourcer", sourcer_agent)
story_graph.add_node("data_search", data_web_search_agent)
story_graph.add_node("merge_enrichment", merge_enrichment_results)

# --- Layer 6: Generation ---
story_graph.add_node("writer", writer_agent)

# --- Layer 7: Verification ---
story_graph.add_node("reviewer", reviewer_agent)
story_graph.add_node("dlq", dead_letter_queue)

# --- Layer 8: Human Oversight ---
story_graph.add_node("hitl_gate", human_in_the_loop_wait)

# === EDGES ===

# Sequential: classify → select prompt
story_graph.add_edge("classifier", "prompt_selector")

# Sequential: prompt → sourcer
story_graph.add_edge("prompt_selector", "sourcer")

# Conditional: after sourcer, decide if data_search is needed
story_graph.add_conditional_edges(
    "sourcer",
    should_run_data_search,  # Returns "data_search" or "writer"
    {
        "data_search": "data_search",
        "writer": "writer"
    }
)

# Data search → merge → writer
story_graph.add_edge("data_search", "merge_enrichment")
story_graph.add_edge("merge_enrichment", "writer")

# Writer → Reviewer (always)
story_graph.add_edge("writer", "reviewer")

# Reviewer → Conditional routing
story_graph.add_conditional_edges(
    "reviewer",
    review_decision_router,  # Returns "approved", "retry", or "escalate"
    {
        "approved": "hitl_gate",
        "retry": "writer",       # Loop back with corrections
        "escalate": "dlq"        # Dead letter queue
    }
)

# DLQ → END (story is parked)
story_graph.add_edge("dlq", END)

# HITL → END (story is done regardless of human decision)
story_graph.add_edge("hitl_gate", END)

story_graph.set_entry_point("classifier")
```

---

## Conditional Edge Functions

```python
def should_run_data_search(state: StoryState) -> str:
    """Decides whether to activate the Data Web Search agent."""
    if state["news_class"] in [1, 5]:
        return "data_search"
    else:
        # Skip data search, go directly to writer
        # Set data_search_results to None
        return "writer"


def review_decision_router(state: StoryState) -> str:
    """Routes based on Reviewer output and retry count."""
    if state["reviewer_status"] == "APPROVED":
        return "approved"
    elif state["retry_count"] >= 3:
        return "escalate"
    else:
        return "retry"
```

---

## Parallel Execution Strategy

### Map-Reduce: Per-Story Processing
After the Curator produces a list of 10-20 stories, each story is processed independently through the per-story subgraph. LangGraph's `map` functionality (or manual parallel invocation) handles this:

```python
async def process_stories_map(state: PipelineState) -> PipelineState:
    """Fan-out: Process each curated story through the story subgraph in parallel."""
    tasks = []
    for story in state["curated_stories"]:
        tasks.append(story_graph.ainvoke(story))
    
    results = await asyncio.gather(*tasks)
    
    completed = [r for r in results if r["status"] == "published"]
    escalated = [r for r in results if r["status"] == "escalated"]
    
    return {
        "stories_completed": completed,
        "stories_escalated": escalated
    }
```

### Parallel Branch: Data Search ∥ Writer
For Class 1 & 5 stories, ideally the Data Web Search runs in parallel with the Writer's first draft. Implementation options:

**Option A (Simpler — Sequential with timeout):**
Data Search runs first with a timeout. If it completes in time, results go to Writer. If not, Writer proceeds without it.

**Option B (True Parallel — Recommended):**
Use LangGraph's parallel node execution or a custom fork:
```python
# Fork after sourcer for Class 1 & 5:
# Branch A: data_search → waits
# Branch B: writer (starts immediately with distilled_context only)
# Merge: Before reviewer, inject data_search results into state
```

For V1, Option A is simpler and still effective. The Reviewer catches any missing data.

---

## Human-in-the-Loop Implementation

The HITL node is special — it pauses the graph execution and waits for external human input.

```python
from langgraph.checkpoint import MemorySaver  # or PostgresSaver for persistence

# The HITL node uses LangGraph's interrupt mechanism
def human_in_the_loop_wait(state: StoryState) -> StoryState:
    """
    This node uses LangGraph's 'interrupt' to pause execution.
    The graph state is persisted. When the human submits their decision
    via the API, the graph resumes from this point.
    """
    # In LangGraph, this is handled via:
    # 1. Graph reaches this node and pauses (checkpoint saved)
    # 2. Frontend polls/subscribes for pending reviews
    # 3. Human submits decision via FastAPI endpoint
    # 4. Backend resumes the graph with the human's input injected into state
    
    # The actual implementation uses LangGraph's interrupt_before or interrupt_after
    return {
        "status": "awaiting_review"
    }
```

Resume after human decision:
```python
# FastAPI endpoint
@app.put("/admin/review/{story_id}")
async def submit_review(story_id: str, decision: ReviewDecision):
    # Resume the paused graph with human input
    await graph.aupdate_state(
        thread_id=story_id,
        values={
            "human_decision": decision.action,
            "human_edit": decision.edited_content,
            "editor_id": decision.editor_id
        }
    )
```

---

## State Transitions Summary

```
Signal birthed by Scout
    → gains `related_sources` after Dedup
    → gains `relevance_score` + `curator_reasoning` after Curator
    → gains `news_class` + `confidence` after Classifier
    → gains `writer_system_prompt` after Prompt Selector
    → gains `distilled_context` after Sourcer
    → gains `data_search_results` after Data Search (Class 1 & 5)
    → gains `writer_draft` after Writer
    → gains `reviewer_status` + `hallucinations` + `formatting_errors` after Reviewer
    → gains `human_decision` + `human_edit` after HITL
    → finalized and written to DB after Publish
```

---

## Error Handling & Resilience

### Retry Strategy (External APIs)
```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=2, min=2, max=8))
async def call_external_api(url: str) -> dict:
    """All external API calls use this wrapper."""
    ...
```

### Node-Level Error Handling
If any node throws an unrecoverable error:
1. Error is logged to `state["error_log"]`
2. Story status is set to `"failed"`
3. Story is routed to the DLQ with error context
4. Other stories in the same run are NOT affected (isolated per-story processing)

### Checkpointing
LangGraph checkpointer (PostgresSaver) persists state after every node transition:
- Enables resume-from-failure (if the server crashes mid-run, restart from last checkpoint)
- Enables the HITL pause/resume pattern
- Provides full audit trail of state evolution

```python
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver(connection_string="postgresql://...")
app = pipeline.compile(checkpointer=checkpointer)
```

---

## Execution Flow Summary

```
CRON fires
  → Scout fetches 100+ raw signals
  → Dedup removes ~40 duplicates (60 unique remain)
  → Curator scores all 60, passes ~15 (relevance ≥ 7)
  → 15 stories fan-out to parallel per-story subgraphs:
      Each story:
        → Classified (sub-second)
        → Prompt selected (instant, lookup)
        → Enriched (Sourcer fetches full content)
        → [If Class 1/5] Data Search runs
        → Writer produces draft
        → Reviewer verifies
          → If rejected (up to 3x): Writer rewrites
          → If approved: HITL gate (pauses)
          → If 3x rejected: DLQ (escalated)
  → Human reviews all approved stories
  → Publish: DB write + Vector embed + Frontend push
  → Run complete
```

---

## Technology Requirements

| Component | Technology |
|-----------|-----------|
| Graph Framework | LangGraph (Python) |
| Checkpointer | PostgresSaver (persistent state) |
| Async Runtime | asyncio (for parallel story processing) |
| State Store | PostgreSQL |
| Vector Store | Qdrant / Pinecone |
| API Layer | FastAPI (serves frontend, manages HITL, resumes graphs) |
| Tracing | LangSmith (native LangGraph integration) |


---

## Failures & Fallbacks

### Entire Pipeline — Catastrophic Failure Mid-Run

**What can fail:** Server crashes, memory overflow, LangGraph process killed mid-execution.

**Fallback strategy:**
- **Checkpointing (already planned):** LangGraph with PostgresSaver persists state after every node. On restart, the pipeline resumes from the last successful node, not from scratch.
- **Idempotency:** Each node should be idempotent — running the same node with the same input twice produces the same output without side effects. This makes replay safe.
- **Timeout per run:** If the entire pipeline hasn't completed within 4 hours (generous for a weekly batch), kill it and alert. Something is badly stuck.
- **Zombie detection:** If a story has been in "reviewing" state for > 30 minutes, it's likely stuck in a loop that the retry counter didn't catch. Force-route to DLQ.

| Failure Point | Max Retries | Escalation Path | Final Fallback |
|---------------|-------------|-----------------|----------------|
| Pipeline crash | Resume from checkpoint | — | Abort + alert if > 4h |

### Core Philosophy
**Degrade gracefully, never crash silently, always produce something rather than nothing.** Every failure has a defined path — retry → escalate → fallback → DLQ/human. No infinite loops, no silent data loss.
