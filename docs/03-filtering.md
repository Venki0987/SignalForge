# Layer 3: Filtering

## Component
- Curator Agent (DeepSeek V4)

---

## What it does
The quality gate. Decides which stories are actually worth writing about — filters out low-signal noise, hype, and irrelevant content. Only high-relevance stories survive this layer.

## Why DeepSeek V4
MoE (Mixture of Experts) architecture provides frontier-level reasoning for filtering at low batch-scoring costs. Since you're evaluating 50-100 items, MoE only activates relevant expert parameters per token, keeping cost manageable.

---

## Implementation Plan

### Input
The deduplicated signal list from Layer 2 (typically 50-100 unique signals).

### Filtering Criteria
The agent's system prompt defines what passes the quality gate:

1. **Genuine Breakthrough vs. Incremental Update** — Is this a real technical advancement or just a version bump?
2. **Enterprise/Engineering Impact** — Does this meaningfully affect how engineering teams build, deploy, or operate?
3. **Source Credibility** — Is it peer-reviewed, from an official blog, or from a reputable outlet? Random Medium posts don't pass.
4. **Timeliness** — Is this actually new, or a rehash of old news with a fresh headline?
5. **Signal Density** — Does the source contain enough technical substance to write a meaningful section about?

### Scoring System
The agent evaluates each signal and produces:
```json
{
  "story_id": "...",
  "relevance_score": 8.5,
  "curator_reasoning": "Novel architecture paper from Google DeepMind with SOTA results on multi-modal reasoning benchmarks. High enterprise relevance.",
  "passes_threshold": true
}
```

Threshold: `relevance_score >= 7` passes to the next layer.

### Story Clustering (Optional Enhancement)
The Curator can also recognize when 3 different signals are all about the same underlying event (e.g., multiple outlets covering the same model launch). It merges them into one "story cluster" with multiple source URLs, selecting the richest source as primary.

### Batch Processing Strategy
- Process all signals in a single batch prompt (list them all, ask for scores) OR process individually. Batch is faster but risks context window limits with 100 items.
- Recommended: Process in batches of 10-15 signals per prompt call. This balances speed with evaluation quality.

### Output
A curated shortlist written to LangGraph state:
- Typically 10-20 high-quality stories per weekly run
- Each has `relevance_score` and `curator_reasoning` attached
- Low-scoring items are logged (for observability) but not passed forward

### Key Goal
Be a ruthless editor. Only stories with genuine technical significance and enterprise relevance survive. The downstream pipeline is expensive (Writer, Reviewer, HITL) — every story that passes must justify that cost.


---

## Failures & Fallbacks

### Curator Agent — Over-Filtering or Under-Filtering

**What can fail:** DeepSeek V4 is too aggressive (kills good stories) or too lax (passes junk that wastes downstream tokens).

**Fallback strategy:**
- **Soft kill, not hard kill.** Stories scored 5-6 (below threshold of 7, but not garbage) go into a "borderline" bucket visible in the HITL dashboard. The editor can manually promote them.
- **Run-level sanity check:** If the Curator passes fewer than 3 stories OR more than 30, something is off. Fewer than 3 → lower the threshold to 6 for this run and re-evaluate. More than 30 → raise to 8 and re-evaluate.
- Never let the Curator output zero stories — that's a hard failure. If it happens, skip curation for this run and pass all deduplicated signals forward with a "UNFILTERED" flag so the HITL editor knows.

| Failure Point | Max Retries | Escalation Path | Final Fallback |
|---------------|-------------|-----------------|----------------|
| Curator (over/under filter) | 1 re-eval at adjusted threshold | Borderline bucket for HITL | Pass all if zero stories selected |
