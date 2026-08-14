# Layer 6: Generation

## Component
- Writer Agent (Claude 4.5 Sonnet / Qwen 3)

---

## What it does
The creative engine. Takes all enriched context and produces polished, technically rigorous newsletter copy tailored to each news class.

## Model Selection Strategy
- **Claude 4.5 Sonnet** — For Class 1 and Class 2 (where prose quality and technical precision matter most)
- **Qwen 3** — For Class 3, 4, and 5 (where output is more structured bullet points — cheaper and fast enough)

---

## Implementation Plan

### Inputs (from LangGraph State)
The Writer receives three things:

1. **Class-specific system prompt** — dynamically selected based on `state["news_class"]`
2. **Distilled Context payload** — from the Sourcer Agent
3. **Data Web Search JSON** — if available from the parallel branch (may be `null`)

### System Prompts by Class

#### Class 1: Model & Foundation Releases
```
You are a Senior AI Systems Architect writing an internal briefing. The provided context 
contains news about a new foundation model. Do not use marketing fluff. You MUST output a 
Markdown comparison table contrasting this new model against current state-of-the-art baselines. 
Highlight token pricing, context window limits, and specific MMLU/HumanEval benchmark scores. 
End with a 2-sentence verdict on its enterprise viability.
```

#### Class 2: Academic Research & Papers
```
You are a Deep Learning Research Scientist explaining a breakthrough to an engineering team. 
The provided context includes a news summary and the abstract/conclusion of an ArXiv paper. 
Explain the core architectural or mathematical novelty. Do not oversimplify. Clearly define 
any new algorithms or datasets introduced. Explain why this approach improves upon previous methods.
```

#### Class 3, 4, 5: Product / Hardware / Policy
```
You are an Enterprise Strategy Lead. Summarize the provided news payload. Focus explicitly on 
how this development impacts daily engineering workflows, system infrastructure, or legal 
compliance. Use bullet points for key takeaways.
```

### User Message Construction
```
Here is the distilled context:
{distilled_context}

Here is the structured data (if available):
{data_search_results or "No structured data available for this story."}

Write the newsletter section following your system prompt instructions.
```

### Output Format
- Markdown-formatted newsletter section
- Stored in `state["writer_draft"]`
- Must follow the formatting rules of its class (comparison table for Class 1, technical explanation for Class 2, bullet points for Class 3/4/5)

### Rewrite Handling (from Reviewer Rejection)
When the Reviewer rejects a draft and loops back:
- The Writer receives the ORIGINAL inputs PLUS the Reviewer's corrections:
  ```
  Previous draft was rejected. Here are the issues:
  - Hallucinations: {hallucinations_found}
  - Formatting errors: {formatting_errors}
  - Reviewer reasoning: {reasoning}
  
  Rewrite the section addressing ALL of the above issues.
  ```
- The Writer must fix every flagged issue while maintaining quality
- `state["retry_count"]` is incremented

### Quality Constraints
- No marketing language ("revolutionary", "game-changing") — ever
- All claims must be traceable to the distilled context
- If data_search_results provides numbers, USE them — don't paraphrase or round
- If data is unavailable, don't invent it — state what's known

### Key Goal
Produce technically rigorous, class-appropriate newsletter copy that is strictly grounded in the provided source material. Every claim should be verifiable against the input context. Quality over speed — this is what the reader sees.


---

## Failures & Fallbacks

### Writer Agent — Iterative Failure (The Big One)

**What can fail:** Writer keeps getting rejected by the Reviewer. After 3 rounds the same mistakes persist — maybe the source material is contradictory, or the Writer is "stuck" in a pattern.

**Fallback strategy — Multi-tier escalation:**

**Tier 1 — Standard loop (retry ≤ 3):**
Same Writer model, same prompt, but with Reviewer's corrections injected. This handles 85% of cases (typos, missing table, rounded number).

**Tier 2 — Model swap (retry 4-5):**
If rejected 3 times, DON'T immediately DLQ. Switch to a different Writer model. If the primary was Claude 4.5 Sonnet, swap to Qwen 3 (or vice versa). Different models have different failure modes — a fresh model often resolves whatever the first one was stuck on. Give the new model 2 more attempts (retry 4 and 5).

**Tier 3 — Targeted regeneration (retry 6, final attempt):**
If the second Writer model also fails, look at WHAT is failing. Take the Reviewer's `hallucinations_found` list and extract just the failing claims. Use a targeted prompt:
```
The following claims could not be verified:
{hallucinations_list}

Remove these claims entirely from the draft. Do not replace them with alternative claims.
Simply delete the unverifiable sentences and smooth the remaining text.
```
This produces a "conservative" draft that's factually safe (omits rather than fabricates). Submit to Reviewer one final time.

**Tier 4 — DLQ (after 6 total attempts):**
If even the conservative draft fails, the story is genuinely problematic. Route to DLQ with the full trace of all 6 attempts. The human editor now has all drafts and all rejection reasons — they can manually write or decide to skip the story.

| Failure Point | Max Retries | Escalation Path | Final Fallback |
|---------------|-------------|-----------------|----------------|
| Writer (repeated rejection) | 3 same model + 2 alt model + 1 conservative | Model swap → targeted regen | DLQ after 6 total attempts |
