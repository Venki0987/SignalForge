# Guardrails — Where, Why, and How

---

## 1. Scout Agent — Input Sanitization Guardrails

**Where:** At the boundary between external APIs and your pipeline state.

**Why:** External sources (Twitter, RSS, web scrapes) can contain injection attacks, malformed data, XSS payloads in titles/snippets, or adversarial prompt injections embedded in article text (someone could write a blog post containing "Ignore previous instructions and...").

**Guardrails needed:**

- **Prompt injection filtering:** Before any scraped text enters the LangGraph state, run it through a sanitizer that strips or escapes common injection patterns. Look for phrases like "ignore previous instructions", "you are now", "system:", "###" at the start of lines. Don't pass raw scraped text directly into any LLM prompt without wrapping it in clearly delimited boundaries (e.g., XML tags `<source_text>...</source_text>`).
- **Content length limits:** Cap `snippet` at 500 chars, `title` at 200 chars. If a source returns a 50KB "snippet," it's malformed — truncate or discard.
- **Schema validation:** Every signal object must pass a Pydantic schema check before entering state. Missing required fields (url, title) → discard the signal, don't crash the pipeline.
- **URL validation:** Verify URLs are well-formed (valid scheme, domain). Block known malicious domains. Don't follow redirects more than 3 levels deep.

**Implementation:**
```python
from pydantic import BaseModel, HttpUrl, validator
import re

INJECTION_PATTERNS = [
    r"ignore\s+(all\s+)?previous\s+instructions",
    r"you\s+are\s+now",
    r"^system:",
    r"^###\s*(instruction|system|prompt)",
]

class SignalInput(BaseModel):
    source: str
    url: HttpUrl
    title: str  # max 200 chars enforced
    snippet: str  # max 500 chars enforced
    
    @validator("title", "snippet")
    def sanitize_text(cls, v):
        # Truncate
        v = v[:500]
        # Check for injection patterns
        for pattern in INJECTION_PATTERNS:
            if re.search(pattern, v, re.IGNORECASE):
                v = "[FILTERED: potential injection detected]"
                break
        return v
```

---

## 2. SLM Classifier — Output Constraint Guardrails

**Where:** After the classifier returns its response, before the value enters state.

**Why:** Even a well-prompted model can return `news_class: 7` or `news_class: "model release"` as a string instead of int, or malformed JSON. If bad data enters state, the conditional routing breaks silently.

**Guardrails needed:**

- **Strict output parsing:** Parse the classifier's response as JSON. If parsing fails, retry once with a "return ONLY valid JSON" reinforcement. If still fails, default to class 3 (most generic prompt) and flag for review.
- **Value range enforcement:** `news_class` must be integer 1-5. Anything else → rejection and retry.
- **Confidence floor:** If `confidence < 0.5`, don't trust the classification at all. Route to Curator for second opinion (as described in fallbacks).

**Implementation:**
```python
def validate_classification(raw_output: str) -> dict:
    try:
        parsed = json.loads(raw_output)
    except json.JSONDecodeError:
        return {"valid": False, "reason": "invalid_json"}
    
    if parsed.get("news_class") not in [1, 2, 3, 4, 5]:
        return {"valid": False, "reason": "class_out_of_range"}
    
    if not isinstance(parsed.get("confidence"), (int, float)):
        return {"valid": False, "reason": "missing_confidence"}
    
    if parsed["confidence"] < 0.0 or parsed["confidence"] > 1.0:
        return {"valid": False, "reason": "confidence_out_of_range"}
    
    return {"valid": True, "data": parsed}
```

---

## 3. Writer Agent — Content Quality Guardrails

**Where:** Post-generation, before the draft enters state and goes to the Reviewer.

**Why:** The Writer can produce content that's structurally wrong (no markdown, wrong format), dangerously opinionated, or contains leaked system prompt fragments.

**Guardrails needed:**

- **Format compliance check (programmatic, no LLM):**
  - Class 1: Must contain a markdown table (`|` characters + `---` separator). If not → auto-reject before even hitting the Reviewer.
  - Class 2: Must be > 200 words (a proper technical explanation can't be shorter).
  - Class 3/4/5: Must contain bullet points (`- ` or `* `).
  - All classes: Must be valid Markdown (no unclosed code blocks, no broken links).

- **Banned phrase filter:**
  ```python
  BANNED_PHRASES = [
      "revolutionary", "game-changing", "groundbreaking", "unprecedented",
      "best ever", "unbelievable", "incredible", "amazing",
      "I think", "In my opinion",  # No editorializing
      "As an AI", "As a language model",  # Prompt leakage
  ]
  ```
  If any banned phrase appears → strip it and flag for the Reviewer with a note, OR auto-reject and retry with an explicit "do not use marketing language" reinforcement.

- **Length guardrails:**
  - Minimum: 150 words (anything shorter isn't substantive)
  - Maximum: 800 words (beyond this, it's bloated for a newsletter section)
  - If over max → ask Writer to condense (don't just truncate)

- **PII detection:** Scan the draft for email addresses, phone numbers, or names that aren't public figures. If found → strip them. Newsletter shouldn't leak anyone's personal info from source material.

**Implementation:**
```python
import re

def writer_guardrail_check(draft: str, news_class: int) -> dict:
    issues = []
    
    # Format checks
    if news_class == 1 and "|" not in draft:
        issues.append("MISSING_TABLE")
    if news_class == 2 and len(draft.split()) < 200:
        issues.append("TOO_SHORT_FOR_RESEARCH")
    if news_class in [3, 4, 5] and not re.search(r"^[\-\*] ", draft, re.MULTILINE):
        issues.append("MISSING_BULLET_POINTS")
    
    # Banned phrases
    for phrase in BANNED_PHRASES:
        if phrase.lower() in draft.lower():
            issues.append(f"BANNED_PHRASE: {phrase}")
    
    # Length
    word_count = len(draft.split())
    if word_count < 150:
        issues.append("BELOW_MINIMUM_LENGTH")
    if word_count > 800:
        issues.append("EXCEEDS_MAXIMUM_LENGTH")
    
    # PII detection (basic)
    if re.search(r'\b[\w.+-]+@[\w-]+\.[\w.-]+\b', draft):
        issues.append("CONTAINS_EMAIL")
    if re.search(r'\b\d{3}[-.]?\d{3}[-.]?\d{4}\b', draft):
        issues.append("CONTAINS_PHONE")
    
    return {"passed": len(issues) == 0, "issues": issues}
```

---

## 4. Reviewer Agent — Output Integrity Guardrails

**Where:** After the Reviewer returns its verdict, before routing decisions are made.

**Why:** The Reviewer itself can hallucinate issues that don't exist (false positives), or return malformed JSON that breaks the routing logic.

**Guardrails needed:**

- **Strict JSON schema validation:** The Reviewer's output MUST parse as valid JSON with exactly the fields: `status`, `reasoning`, `hallucinations_found`, `formatting_errors`. If it doesn't → retry once, then default to "APPROVED" with a flag for HITL (safer to let human decide than to loop forever on a broken Reviewer).

- **Logical consistency check:**
  - If `hallucinations_found` is non-empty but `status` is "APPROVED" → override to "REJECTED" (the model contradicted itself).
  - If `status` is "REJECTED" but both `hallucinations_found` and `formatting_errors` are empty → suspicious. Retry once — the model rejected without evidence.

- **Hallucination specificity requirement:** Each item in `hallucinations_found` must reference a specific claim (not vague things like "the tone is off"). If a flagged hallucination is fewer than 10 characters, it's probably garbage — ignore it.

**Implementation:**
```python
def validate_reviewer_output(raw_output: str) -> dict:
    try:
        parsed = json.loads(raw_output)
    except json.JSONDecodeError:
        return {"valid": False, "reason": "invalid_json", "action": "retry"}
    
    required_fields = ["status", "reasoning", "hallucinations_found", "formatting_errors"]
    for field in required_fields:
        if field not in parsed:
            return {"valid": False, "reason": f"missing_{field}", "action": "retry"}
    
    if parsed["status"] not in ["APPROVED", "REJECTED"]:
        return {"valid": False, "reason": "invalid_status", "action": "retry"}
    
    # Logical consistency
    has_issues = (len(parsed["hallucinations_found"]) > 0 or 
                  len(parsed["formatting_errors"]) > 0)
    
    if has_issues and parsed["status"] == "APPROVED":
        parsed["status"] = "REJECTED"  # Override inconsistency
    
    if not has_issues and parsed["status"] == "REJECTED":
        return {"valid": False, "reason": "rejection_without_evidence", "action": "retry"}
    
    # Filter out vague hallucination flags
    parsed["hallucinations_found"] = [
        h for h in parsed["hallucinations_found"] if len(h) >= 10
    ]
    
    return {"valid": True, "data": parsed}
```

---

## 5. Data Web Search Agent — Numerical Integrity Guardrails

**Where:** After the search agent returns structured data, before it merges into state.

**Why:** The agent might return numbers for the WRONG model (searched for GPT-5, found GPT-4 stats), hallucinate plausible-looking but fake benchmark scores, or return data in wrong units.

**Guardrails needed:**

- **Model name cross-check:** The `model_name` field in the returned JSON must fuzzy-match the model name from the original story signal. If they don't match (edit distance > 3) → discard the entire result.
- **Sanity range checks for known metrics:**
  ```
  MMLU: 0-100 (percentage)
  GSM8k: 0-100 (percentage)
  HumanEval: 0-100 (percentage)
  Context window: 1,000 - 10,000,000 (tokens)
  API cost per 1M tokens: $0.01 - $500
  Parameter count: must contain 'B' or 'T' or 'M' suffix
  ```
  Anything outside these ranges → set to `null` (clearly wrong data).
- **Source URL verification:** If `source_url` is provided, it must be a real URL (not a hallucinated one). Optionally, do a HEAD request to verify it returns 200. If not → mark data as `"unverified"`.

**Implementation:**
```python
from difflib import SequenceMatcher

def validate_search_data(data: dict, expected_model: str) -> dict:
    # Model name match
    if data.get("model_name"):
        similarity = SequenceMatcher(None, 
            data["model_name"].lower(), 
            expected_model.lower()
        ).ratio()
        if similarity < 0.6:
            return {"valid": False, "reason": "model_name_mismatch"}
    
    # Range checks
    range_checks = {
        "mmlu_score": (0, 100),
        "gsm8k_score": (0, 100),
        "humaneval_score": (0, 100),
        "context_window": (1000, 10_000_000),
    }
    
    for field, (min_val, max_val) in range_checks.items():
        val = data.get(field)
        if val is not None:
            try:
                numeric = float(str(val).replace("%", ""))
                if numeric < min_val or numeric > max_val:
                    data[field] = None  # Discard out-of-range
            except ValueError:
                data[field] = None  # Not a number
    
    return {"valid": True, "data": data}
```

---

## 6. RAG Chatbot — Grounding & Safety Guardrails

**Where:** At two points: (a) before the retrieved chunks are sent to the model, and (b) after the model generates a response.

**Why:** The chatbot might answer from general knowledge (ignoring the RAG context), generate harmful content if a user asks adversarial questions, or leak system prompt details.

**Guardrails needed:**

### Pre-generation (input side):

- **User query sanitization:** Strip injection attempts from user messages. Wrap user input in clear delimiters:
  ```
  <user_question>{sanitized_query}</user_question>
  ```
- **Query relevance check:** If the user asks something completely unrelated to AI/tech (e.g., "how do I make a bomb"), reject immediately with a canned response. Use a lightweight classifier or keyword blocklist.
- **Chunk relevance threshold:** If the top re-ranked chunk scores below 0.3, don't inject any chunks. Respond from summary only, or decline.

### Post-generation (output side):

- **Grounding verification (lightweight):** Check if the key claims in the chatbot's response appear (even partially) in the provided chunks or summary. This is a string-matching heuristic, not an LLM call:
  - Extract sentences from the response that contain numbers or specific claims
  - Check if those numbers/claims appear in the source context
  - If > 30% of claims have no source match → regenerate at temperature 0.1

- **System prompt leakage detection:** If the response contains fragments of the system prompt (e.g., "You are a technical assistant", "Answer strictly based on"), filter them out before sending to the user.

- **Response length cap:** Max 1500 words per response. If the model runs long (deep dive mode can get verbose), truncate gracefully at the last complete paragraph and append "Would you like me to continue?"

- **Refusal for out-of-scope:** If the chatbot detects it cannot answer from context, it MUST use the predefined refusal: "I don't have enough information from the source material to answer that. You can check the full paper here: {link}." It should NEVER say "Based on my training data..." or "Generally speaking..."

**Implementation:**
```python
# Post-generation guardrail
def chatbot_output_guardrail(response: str, context_chunks: list[str], summary: str) -> dict:
    issues = []
    
    # System prompt leakage
    LEAKAGE_PATTERNS = [
        "you are a technical assistant",
        "answer strictly based on",
        "provided context",
        "as an AI language model",
    ]
    for pattern in LEAKAGE_PATTERNS:
        if pattern.lower() in response.lower():
            response = response.replace(pattern, "")
            issues.append("PROMPT_LEAKAGE_STRIPPED")
    
    # Grounding check — extract sentences with numbers
    import re
    numeric_sentences = [s for s in response.split(". ") if re.search(r'\d+', s)]
    combined_context = " ".join(context_chunks) + " " + summary
    
    ungrounded = 0
    for sentence in numeric_sentences:
        # Extract the number and check if it exists in context
        numbers = re.findall(r'\d+\.?\d*', sentence)
        if numbers and not any(n in combined_context for n in numbers):
            ungrounded += 1
    
    if numeric_sentences and ungrounded / len(numeric_sentences) > 0.3:
        issues.append("HIGH_UNGROUNDED_RATIO")
    
    # Length cap
    words = response.split()
    if len(words) > 1500:
        # Truncate at last paragraph break before 1500 words
        truncated = " ".join(words[:1500])
        last_para = truncated.rfind("\n\n")
        if last_para > 0:
            response = truncated[:last_para] + "\n\nWould you like me to continue?"
        issues.append("TRUNCATED")
    
    return {
        "response": response,
        "issues": issues,
        "needs_regen": "HIGH_UNGROUNDED_RATIO" in issues
    }
```

---

## 7. HITL Dashboard — Access Control & Integrity Guardrails

**Where:** API layer (FastAPI endpoints) serving the human review dashboard.

**Why:** Unauthorized access could allow someone to approve fabricated content, inject malicious edits, or delete stories from the queue.

**Guardrails needed:**

- **Authentication:** JWT-based auth. Only authenticated editors can access `/admin/*` endpoints.
- **Role-based access:** Two roles:
  - `editor`: Can approve, edit, reject stories
  - `admin`: Can publish runs, manage users, access observability
- **Edit validation:** When an editor submits an "edited" story, the edited content goes through the same Writer guardrail checks (banned phrases, PII detection, length limits). Humans can also introduce errors.
- **Audit immutability:** Once an audit log entry is written, it cannot be modified or deleted via the API. Append-only.
- **Rate limiting:** Max 50 review actions per hour per editor. Prevents automated abuse if credentials are compromised.
- **CSRF protection:** Standard CSRF tokens on all state-changing endpoints.

---

## 8. Pipeline-Level — Token Budget Guardrails

**Where:** Wrapped around every LLM call across the entire pipeline.

**Why:** A single runaway story (e.g., a 200-page PDF that somehow gets through) could burn through thousands of dollars in tokens. A misbehaving agent in a loop could drain your API budget.

**Guardrails needed:**

- **Per-node token budget:**

  | Node | Max Input Tokens | Max Output Tokens |
  |------|-----------------|-------------------|
  | Scout | 2,000 | 4,000 |
  | Curator (per batch of 10) | 8,000 | 3,000 |
  | Classifier | 500 | 100 |
  | Sourcer | 2,000 | 1,000 |
  | Data Search | 1,000 | 500 |
  | Writer | 12,000 | 2,000 |
  | Reviewer | 15,000 | 1,000 |
  | Chatbot | 8,000 | 2,000 |

- **Per-story budget:** Total tokens spent on a single story (across all retries) must not exceed 100K tokens. If it does → force DLQ.
- **Per-run budget:** Total cost for a weekly run must not exceed a configurable threshold (e.g., $50). If exceeded mid-run → pause pipeline, alert admin, wait for approval to continue.
- **Input truncation:** Before passing distilled context to Writer or Reviewer, truncate to the max input budget. Use smart truncation (keep abstract + conclusion, trim middle sections) rather than hard character cut.

**Implementation:**
```python
from functools import wraps

class TokenBudgetExceeded(Exception):
    pass

def enforce_token_budget(max_input: int, max_output: int):
    def decorator(func):
        @wraps(func)
        async def wrapper(state, *args, **kwargs):
            # Estimate input tokens (rough: 1 token ≈ 4 chars)
            input_text = json.dumps(state)
            estimated_input = len(input_text) // 4
            
            if estimated_input > max_input:
                # Smart truncation
                state = truncate_context(state, max_input)
            
            result = await func(state, *args, **kwargs)
            
            # Track cumulative spend
            state["_cumulative_tokens"] = state.get("_cumulative_tokens", 0) + result.get("tokens_used", 0)
            
            if state["_cumulative_tokens"] > 100_000:  # Per-story limit
                raise TokenBudgetExceeded(f"Story {state['story_id']} exceeded 100K token budget")
            
            return result
        return wrapper
    return decorator
```

---

## 9. Vector DB — Embedding Injection Guardrails

**Where:** Before text is embedded and stored in Qdrant/Pinecone.

**Why:** If adversarial text is embedded (e.g., someone publishes an ArXiv paper with hidden instructions in the PDF), it could poison your RAG retrieval — returning manipulated chunks when users query the chatbot.

**Guardrails needed:**

- **Pre-embedding sanitization:** Same injection pattern detection as the Scout layer. Strip suspicious content before embedding.
- **Chunk metadata integrity:** Every chunk MUST have a valid `story_id` that maps to a published story in PostgreSQL. Orphan chunks (no valid parent story) should never exist.
- **Retrieval-time filtering:** When retrieving chunks for the chatbot, ALWAYS filter by `story_id` (scoped to the story the user is asking about). Never do an unscoped vector search across all content — that's how cross-story contamination happens.
- **Periodic hygiene:** Run a weekly job that checks for orphan chunks, duplicate embeddings, or chunks whose parent story was later rejected by HITL. Delete them.

---

## 10. Observability Layer — Alert Fatigue Guardrails

**Where:** The alerting logic within the observability system.

**Why:** If you alert on everything, the team ignores everything. Alerts must be actionable and rare.

**Guardrails needed:**

- **Alert deduplication:** Same alert condition within a 1-hour window → send once, not 15 times.
- **Severity tiers:**
  - `CRITICAL` (pager): Pipeline completely stuck, DB down, budget exceeded
  - `WARNING` (Slack): High rejection rate, single source down, DLQ entry
  - `INFO` (dashboard only): Borderline dedup decisions, partial data search results
- **Auto-resolve:** If a WARNING condition (e.g., "ArXiv API down") resolves on the next check → auto-close the alert, don't leave it dangling.
- **Cooldown:** After a CRITICAL alert is sent, suppress duplicate CRITICAL alerts for 30 minutes. Gives the team time to respond without getting bombarded.

---

## Summary Table

| Layer | Guardrail Type | Mechanism | Failure Mode Prevented |
|-------|---------------|-----------|----------------------|
| Scout | Input sanitization | Regex + Pydantic schema | Prompt injection, malformed data |
| Classifier | Output constraint | JSON schema + range check | Bad routing from invalid classification |
| Writer | Content quality | Programmatic format check + banned phrases | Marketing fluff, wrong format, PII leaks |
| Reviewer | Output integrity | Schema validation + logical consistency | Broken routing from malformed verdicts |
| Data Search | Numerical integrity | Range checks + name matching | Wrong model's stats, fake numbers |
| Chatbot (input) | Query sanitization | Injection filter + relevance threshold | Adversarial prompts, off-topic abuse |
| Chatbot (output) | Grounding + safety | String matching + leakage detection | Hallucination, prompt leakage |
| HITL | Access control | JWT + RBAC + rate limiting | Unauthorized edits, credential abuse |
| Pipeline-wide | Token budget | Per-node + per-story + per-run caps | Runaway costs, infinite loops |
| Vector DB | Embedding integrity | Sanitization + scoped retrieval | RAG poisoning, cross-contamination |
| Observability | Alert hygiene | Dedup + severity tiers + cooldown | Alert fatigue → ignored real issues |

---

## Core Philosophy

**Validate at every boundary** (external → internal, model output → state, user input → model). Keep validation fast and deterministic (no LLM calls for guardrails except as a last resort). Log everything that gets filtered so you can tune thresholds over time.
