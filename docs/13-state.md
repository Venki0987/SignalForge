# State Schema

## Overview

The pipeline uses a single `NewsItem` state that represents one news item's full lifecycle, and a `PipelineState` that holds run metadata plus a list of all `NewsItem` objects. Every node reads and writes to fields within `NewsItem` — the schema stays constant, values evolve as the item flows through the graph.

---

## NewsItem State

```python
from typing import TypedDict, Optional, Literal

class NewsItem(TypedDict):
    """Single news item — its full lifecycle state."""
    
    # Identity & Status
    item_id: str
    status: Literal[
        "fetched",
        "deduplicated",
        "curated",
        "classified",
        "enriched",
        "drafted",
        "reviewing",
        "approved",
        "escalated",
        "published",
        "rejected_by_human"
    ]
    
    # ─── Scout Output ───
    source: str  # "rss_techcrunch" | "rss_openai" | "rss_anthropic" | "reddit"
    url: str
    title: str
    snippet: str  # First 200-500 chars
    raw_metadata: dict
    # {
    #   "authors": [],
    #   "published_date": str,
    #   "subreddit": Optional[str],
    #   "score": Optional[int],
    #   "num_comments": Optional[int]
    # }
    fetched_at: str  # ISO timestamp
    
    # ─── Dedup Output ───
    related_sources: list[dict]  # Other URLs covering the same story
    
    # ─── Curator Output ───
    relevance_score: Optional[float]  # 1-10
    curator_reasoning: Optional[str]
    
    # ─── Classifier Output ───
    news_class: Optional[int]  # 1-5
    classification_confidence: Optional[float]  # 0.0-1.0
    
    # ─── Enrichment Output ───
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
    data_search_results: Optional[dict]
    # {
    #   "model_name": Optional[str],
    #   "parameters": Optional[str],
    #   "context_window": Optional[str],
    #   "mmlu_score": Optional[str],
    #   "gsm8k_score": Optional[str],
    #   "humaneval_score": Optional[str],
    #   "api_cost_per_1m_tokens": Optional[str],
    #   "source_url": Optional[str],
    #   "data_confidence": "verified" | "unverified" | "partial"
    # }
    data_search_complete: bool
    
    # ─── Writer Output ───
    writer_system_prompt: Optional[str]  # The class-specific prompt used
    writer_draft: Optional[str]  # Markdown output
    retry_count: int  # Starts at 0, max 6
    
    # ─── Reviewer Output ───
    reviewer_status: Optional[Literal["APPROVED", "REJECTED"]]
    reviewer_reasoning: Optional[str]
    hallucinations_found: list[str]
    formatting_errors: list[str]
    
    # ─── HITL Output ───
    human_decision: Optional[Literal["approved", "edited", "rejected"]]
    human_edit: Optional[str]  # If edited, the final version
    editor_id: Optional[str]
    
    # ─── Pipeline Control ───
    error_log: list[str]  # Any errors encountered during processing
    cumulative_tokens: int  # Total tokens spent on this item (budget tracking)
```

---

## PipelineState

```python
class PipelineState(TypedDict):
    """Top-level state — run metadata + the list of all news items."""
    
    # Run Metadata
    run_id: str
    triggered_at: str  # ISO timestamp
    trigger_type: Literal["cron", "manual"]
    
    # The List
    items: list[NewsItem]
    
    # Run-Level Metrics (accumulated during execution)
    total_signals_fetched: int
    duplicates_removed: int
    items_filtered_out: int
    total_cost_usd: float
    run_status: Literal["running", "awaiting_review", "published", "failed"]
```

---

## How State Evolves Through the Pipeline

| Layer | Fields Populated |
|-------|-----------------|
| Scout | `item_id`, `source`, `url`, `title`, `snippet`, `raw_metadata`, `fetched_at`, `status="fetched"` |
| Dedup | `related_sources`, `status="deduplicated"` |
| Curator | `relevance_score`, `curator_reasoning`, `status="curated"` |
| Classifier | `news_class`, `classification_confidence`, `writer_system_prompt`, `status="classified"` |
| Enrichment | `distilled_context`, `data_search_results`, `data_search_complete`, `status="enriched"` |
| Writer | `writer_draft`, `retry_count`, `status="drafted"` |
| Reviewer | `reviewer_status`, `reviewer_reasoning`, `hallucinations_found`, `formatting_errors`, `status="reviewing"` or `"approved"` or `"escalated"` |
| HITL | `human_decision`, `human_edit`, `editor_id`, `status="published"` or `"rejected_by_human"` |

---

## Design Decisions

1. **Single schema, not two** — Every node sees the same `NewsItem` fields. No translation between schemas at the fan-out point.
2. **Status field as lifecycle tracker** — Instead of separate lists (in_progress, completed, escalated), each item carries its own status. Filter the list to get items at any stage.
3. **Optional fields for later stages** — Fields populated by downstream nodes start as `None`. Nodes only read fields they need and only write fields they own.
4. **Immutable history** — Once a field is written, it stays. The Reviewer doesn't overwrite the Writer's draft — it adds its own verdict alongside it. This preserves the full trace for debugging.
