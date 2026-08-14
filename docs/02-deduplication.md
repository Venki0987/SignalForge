# Layer 2: Deduplication

## Component
- Deduplication Layer (Programmatic — no LLM)

---

## What it does
Filters out duplicate stories that are the same news reported by multiple outlets. Prevents redundant processing of echoed stories downstream.

## Why This Layer Exists
AI news gets echoed across dozens of outlets. Without dedup, the Curator will waste cycles evaluating 15 versions of the same story, and worse, the Writer might produce multiple newsletter sections about the same event.

---

## Implementation Plan

### Core Mechanism
This is NOT an LLM node — it's a pure programmatic layer (Python function within the LangGraph DAG).

### Step-by-Step Flow

1. Take every signal object's `title + snippet` text from the Scout's output.
2. Generate an embedding using a lightweight local model:
   - Model: `all-MiniLM-L6-v2` (from sentence-transformers)
   - Why: Fast, runs locally, no API cost, good enough for similarity detection
3. Maintain an in-memory FAISS index for the current run.
4. For each new signal:
   - Embed it
   - Search the FAISS index for nearest neighbor
   - If **cosine similarity > 0.92** → it's a duplicate → discard it (or attach it as a "related source" to the original signal)
   - If **unique** → add it to the FAISS index and pass it forward

### Cross-Run Persistence
- Optionally persist the FAISS index to disk across weekly runs so stories from last week that are still circulating don't get re-processed.
- On each new run, load the previous index and check new signals against it.

### Duplicate Handling Strategy
When a duplicate is found, don't just delete it. Attach it as an additional source URL to the original signal:
```json
{
  "primary_signal": { "url": "...", "title": "..." },
  "related_sources": [
    { "url": "...", "source": "techcrunch" },
    { "url": "...", "source": "theverge" }
  ]
}
```
This gives the Writer multiple reference points without creating duplicate newsletter entries.

### Expected Impact
- Typically cuts 30-50% of raw signals.
- Saves significant LLM inference cost downstream (Curator, Router, Writer all process fewer items).

### Key Goal
Ensure every story that passes to the Curator represents a unique news event. Merge related coverage into single clusters. Keep it fast and cheap — no LLM calls, pure vector math.


---

## Failures & Fallbacks

### Dedup Layer — Threshold Miscalibration

**What can fail:** Not a crash, but a silent quality issue. If cosine threshold (0.92) is too aggressive, you merge stories that are actually different. If too lenient, duplicates slip through.

**Fallback strategy:**
- Log every dedup decision (pair similarity score + which signals were merged vs. kept).
- After each run, surface the "borderline" merges (similarity 0.88–0.94) in the observability dashboard for human review.
- **Self-tuning:** Track how often the Curator later filters out items that the dedup layer passed. If it's high, the threshold is too lenient. Adjust quarterly based on data.

| Failure Point | Max Retries | Escalation Path | Final Fallback |
|---------------|-------------|-----------------|----------------|
| Dedup (miscalibration) | N/A | Log borderlines for review | Quarterly threshold tuning |
