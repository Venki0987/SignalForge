# Layer 5: Enrichment

## Components
- Sourcer Agent (runs in parallel where needed)
- Data Web Search Agent (Llama 3.3 70B) — Parallel Branch, Class 1 & 5 Only

---

## Sourcer Agent

### What it does
Takes the classified story and fetches the full source material — the raw content that the Writer will synthesize into newsletter copy.

### Implementation Plan

#### Fetching Strategy by Class

| Class | What to Fetch | Method |
|-------|---------------|--------|
| Class 1 (Model Releases) | Official blog post full text, model cards, technical reports | Web scraper (Trafilatura) |
| Class 2 (Research Papers) | Full ArXiv PDF text + YouTube video transcript (if exists) | PyMuPDF/pdfplumber for PDF, youtube-transcript-api for video |
| Class 3 (Product Launches) | Full article text, product documentation pages | Web scraper (BeautifulSoup/Trafilatura) |
| Class 4 (Hardware) | Full article text, spec sheets | Web scraper |
| Class 5 (Business/Policy) | Full article text, regulatory documents | Web scraper |

#### The "Distilled Context" Concept
Rather than flooding the Writer with 40 pages of PDF, we build a curated payload containing only the most information-dense sections:

```json
{
  "news_text": "The original news snippet/article full text",
  "paper_abstract": "If academic paper — the abstract section",
  "paper_conclusion": "If academic paper — the conclusion section",
  "paper_key_figures": "Captions of key figures/tables if extractable",
  "transcript_summary": "If YouTube video exists — a condensed summary of the transcript",
  "source_urls": ["list of all source URLs for attribution"],
  "raw_metadata": {
    "authors": [],
    "institution": "",
    "published_date": "",
    "arxiv_id": ""
  }
}
```

#### Why Distilled (Not Full)
- Prevents "lost in the middle" hallucinations from the Writer
- Keeps token usage manageable and cost-efficient
- Forces the Writer to produce a cohesive narrative grounded in the technical abstract
- The FULL text still goes to the Vector DB (Layer 9) for the chatbot to query later

#### Error Handling
- Retry with exponential backoff on all HTTP fetches (3 retries: 2s → 4s → 8s)
- If PDF extraction fails, fall back to the abstract-only from ArXiv API metadata
- If YouTube transcript is unavailable (private video, no captions), skip it and note in metadata

---

## Data Web Search Agent (Llama 3.3 70B)

### What it does
Hunts for hard numbers — benchmark scores, pricing, parameter counts — that may not be in the original news article itself.

### Why Llama 3.3 70B
Exceptional at navigating and extracting structured data from unstructured web pages. Can parse complex tables and pricing pages reliably.

### Activation Condition
ONLY activates when `news_class` is:
- **Class 1** (Model Releases) — searches for benchmarks, pricing, specs
- **Class 5** (Business/Policy) — searches for funding amounts, regulatory specifics, timelines

### Async Parallel Execution
This agent runs ASYNCHRONOUSLY in parallel with the Writer Agent:
- In LangGraph, implemented as a parallel branch (fork) from the enrichment layer
- The Writer begins drafting from the distilled context immediately
- If the Data Web Search finishes before the Reviewer runs → its data is available for verification
- If it's slow → the Writer's draft proceeds without it, and the Reviewer may flag missing numbers (triggering a second Writer pass once data arrives)

### Search Strategy
Uses Tavily or Exa.ai tool calls to find specific structured data:

For Class 1:
- "{model_name} MMLU benchmark score"
- "{model_name} API pricing per million tokens"
- "{model_name} context window size"
- "{model_name} parameter count"
- "{model_name} HumanEval score"

For Class 5:
- "{company} funding round amount date"
- "{regulation_name} enforcement date"
- "{policy} affected companies list"

### Output Schema (Strict JSON)
```json
{
  "model_name": "GPT-5",
  "parameters": "1.8T (rumored)",
  "context_window": "256k tokens",
  "mmlu_score": "92.1%",
  "gsm8k_score": "96.4%",
  "humaneval_score": "89.7%",
  "api_cost_per_1m_tokens": "$3.00 input / $15.00 output",
  "source_url": "https://...",
  "data_confidence": "verified | unverified | partial"
}
```

If a value cannot be found → return `null` for that field. NEVER guess or hallucinate numbers.

### LangGraph State Merge
The async results get merged into state via:
```python
state["data_search_results"] = { ... }  # or None if not applicable
```

### Key Goal
Provide the Writer and Reviewer with hard, verifiable numbers that the original news source may not have included. Run in parallel to avoid adding latency to the main pipeline. Never fabricate data — `null` is always better than a guess.


---

## Failures & Fallbacks

### Sourcer Agent — PDF Extraction or Transcript Failures

**What can fail:** ArXiv PDF is corrupted or in a scan-only format. YouTube video has no captions. Blog post is behind a paywall.

**Fallback strategy:**
- **PDF fails:** Fall back to abstract + conclusion from ArXiv API metadata (always available as plain text). Flag the story as "partial context" so the Writer knows not to reference specific methodology sections.
- **Transcript unavailable:** Skip it. Note in `distilled_context` that no video context was available. The Writer works with what it has.
- **Paywall/403:** Try a different source URL from the `related_sources` list (populated by dedup layer). If all sources are paywalled, the story proceeds with just the snippet. If even the snippet is too thin, route to DLQ — can't write a quality section without source material.
- **Minimum viable context:** The `distilled_context` MUST contain at least `news_text` (200+ chars). If it doesn't, the story is unprocessable — send to DLQ.

### Data Web Search Agent — Returns Garbage or Nulls

**What can fail:** Tavily returns wrong numbers (finds stats for a different model), all fields come back null, or the agent hallucinates structured data.

**Fallback strategy:**
- **All nulls:** Acceptable. Writer proceeds without numerical data. The newsletter section just won't have a comparison table row for that metric. Better to omit than fabricate.
- **Suspicious data:** If the agent returns data but `source_url` is missing or doesn't match the model name, treat it as unverified. Add a `"data_confidence": "unverified"` flag. The Writer MUST prefix any unverified numbers with "reportedly" or "according to unconfirmed sources."
- **Timeout (takes > 30s):** Kill it. Writer proceeds without search data. This is why it runs async — the main pipeline isn't blocked.

| Failure Point | Max Retries | Escalation Path | Final Fallback |
|---------------|-------------|-----------------|----------------|
| Sourcer (PDF/transcript fail) | 3 per fetch | Fall back to metadata-only | DLQ if < 200 chars context |
| Data Web Search (bad data) | Timeout at 30s | Proceed without | Writer omits numbers |
