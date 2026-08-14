# Layer 7: Verification Loop

## Components
- Reviewer Agent (DeepSeek R1)
- Dead Letter Queue / Human Escalation

---

## Reviewer Agent (DeepSeek R1)

### What it does
The fact-checker. Performs entailment-based claim verification — mapping every claim in the Writer's draft back to the source material. Catches hallucinations, numerical errors, and formatting violations.

### Why DeepSeek R1
Explicitly trained via Reinforcement Learning for logical deduction, self-verification, and multi-step reasoning. Natively outputs a `<think>` block before its final response, which we exploit to force claim-by-claim verification.

---

### Implementation Plan

#### Inputs (from LangGraph State)
1. `state["distilled_context"]` — the source truth
2. `state["data_search_results"]` — the numerical truth (if available)
3. `state["writer_draft"]` — what needs verification
4. `state["news_class"]` — determines evaluation criteria

#### The Core Verification Pattern

1. **Input Phase:** Agent receives the Distilled Context + Data Search JSON + Writer Draft
2. **`<think>` Phase:** DeepSeek R1 isolates every technical claim in the draft and searches the source material for evidence
3. **Output Phase:** Returns structured JSON with verdict
4. **Routing Phase:** LangGraph conditional edge routes based on status

#### System Prompt
```
You are an uncompromising Lead Technical Editor. Your job is to prevent AI hallucinations.

You will receive two inputs:
1. [SOURCE_FACTS]: The raw, distilled context.
2. [DRAFT]: The newsletter copy written by the Writer Agent.

Your task is to verify the [DRAFT] against the [SOURCE_FACTS] based on the current news class: {news_class}.

EVALUATION RULES:
1. Extract every quantitative metric, architectural claim, or factual statement from the [DRAFT].
2. Attempt to find exact supporting evidence in the [SOURCE_FACTS].
3. If a claim is in the [DRAFT] but NOT in the [SOURCE_FACTS], it is a hallucination.

OUTPUT FORMAT:
You must output a valid JSON object.
Before outputting the JSON, use your <think> tags to map each claim one by one.

{
  "status": "APPROVED" | "REJECTED",
  "reasoning": "A concise explanation of your decision.",
  "hallucinations_found": ["List of unsupported claims, or empty if none"],
  "formatting_errors": ["List of missing tables/bullet points based on class rules"]
}

If 'hallucinations_found' or 'formatting_errors' contains any items, you MUST set "status" to "REJECTED".
```

#### Class-Specific Evaluation Criteria

| Class | Focus | Rejection Triggers |
|-------|-------|-------------------|
| 1 - Model Releases | Quantitative precision & benchmark integrity | Missing metrics, benchmark scores off by even 0.1%, missing comparison table, unit confusion (per-million vs per-thousand) |
| 2 - Research Papers | Mathematical & architectural faithfulness | Oversimplified math, wrong attribution of capabilities, confusing "future work" with current results |
| 3 - Product Launches | Practical impact & feature accuracy | Hallucinated integrations, marketing fluff instead of concrete features, fabricated pricing |
| 4 - Hardware | System specs & constraints | Confused hardware metrics (bandwidth vs compute), omitted infrastructure requirements |
| 5 - Business/Policy | Nuance & objective reporting | Misstated timelines, confused proposed vs enacted policy, injected unverified opinions |

#### Critical: Cross-Reference Data Web Search Output
For Class 1 and 5, the Reviewer ALSO validates the draft's numbers against `state["data_search_results"]`. This closes the blind spot where the Writer could hallucinate a number that isn't in the news text but IS in the search results.

#### Output Schema
```json
{
  "status": "REJECTED",
  "reasoning": "Draft claims model scored 91.2% on MMLU but source data shows 88.4%.",
  "hallucinations_found": [
    "MMLU score stated as 91.2% — actual is 88.4%",
    "Claims 'native tool calling' but paper says 'future work'"
  ],
  "formatting_errors": [
    "Missing required comparison table for Class 1 story"
  ]
}
```

---

### Routing Logic (LangGraph Conditional Edge)

```
IF status == "APPROVED":
    → Route to Layer 8 (Human-in-the-Loop)

IF status == "REJECTED" AND retry_count < 3:
    → Route BACK to Writer Agent with corrections payload
    → Increment state["retry_count"]

IF status == "REJECTED" AND retry_count >= 3:
    → Route to Dead Letter Queue
```

---

## Dead Letter Queue / Human Escalation

### What it does
Catches stories that the Writer simply cannot get right after 3 attempts. Prevents infinite loops and unbounded token burn.

### Implementation Plan
- A terminal node in the DAG for that specific story.
- Logs the full state into PostgreSQL:
  - All 3 drafts
  - All 3 rejection reasons with hallucination lists
  - The source context
  - Status: `"ESCALATED"`
- Sends a notification (email or Slack webhook) to the editorial team:
  - "Story X failed verification 3 times — needs manual writing/review"
  - Includes a link to the HITL dashboard where the full context is viewable
- The story is effectively "parked" — it doesn't block other stories in the same run from proceeding.

### Why 3 Attempts?
- 1 rejection is normal — Writers often miss a formatting requirement or round a number
- 2 rejections suggest the source material may be ambiguous
- 3 rejections mean something systemic is wrong — either the source is contradictory, the prompt is unclear, or the story is genuinely too complex for automated synthesis

### Key Goal
Guarantee factual accuracy with zero tolerance for hallucinations. Every claim in the final newsletter must be traceable to the source material. The `<think>` trace forces DeepSeek R1 to do the actual work of verification rather than pattern-matching "this looks about right." The DLQ ensures the system never loops infinitely or publishes unverified content.


---

## Failures & Fallbacks

### Reviewer Agent — False Rejections or Passes

**What can fail:** DeepSeek R1 is TOO strict (rejects drafts for non-issues, like nitpicking phrasing that isn't actually a factual claim) or TOO lenient (approves hallucinations).

**Fallback strategy:**
- **Too strict (high rejection rate):** Track rejection rate per class. If Class 3 (Product Launches) has a rejection rate > 70%, the evaluation criteria for that class are probably miscalibrated. Alert the team to review and relax the Class 3 Reviewer prompt.
- **Too lenient (hallucinations reach HITL):** Track how often human editors reject stories that the Reviewer approved. If this exceeds 20%, the Reviewer prompt needs tightening. This is a feedback loop — log the human's rejection reason and use it to refine the Reviewer's criteria.
- **Second judge pattern:** For high-stakes stories (Class 1 model releases with benchmark claims, Class 5 regulatory pieces), add an optional second Reviewer pass using a DIFFERENT model (e.g., Claude 4.5 Sonnet as a second judge). Both must approve. Only enable this for stories where factual errors have high consequences — don't double-review every story (too expensive).

| Failure Point | Max Retries | Escalation Path | Final Fallback |
|---------------|-------------|-----------------|----------------|
| Reviewer (miscalibrated) | N/A | Track rejection rates | Add second judge for high-stakes |
