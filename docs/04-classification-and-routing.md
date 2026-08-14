# Layer 4: Classification & Routing

## Component
- SLM Classifier Router (Qwen 3.5-9B)

---

## What it does
Tags each surviving story with a `news_class` (1-5), which determines everything downstream — which system prompt the Writer receives, whether the Data Web Search agent activates, and how the Reviewer evaluates the draft.

## Why Qwen 3.5-9B
Sub-second semantic classification at negligible inference cost. A lightweight SLM is perfect for a clear-cut 5-class categorization task — no need to waste a frontier model on routing decisions.

---

## Implementation Plan

### The Five Classes

| Class | Label | Downstream Effect |
|-------|-------|-------------------|
| 1 | Model & Foundation Releases | Architect prompt + Data Web Search agent activates |
| 2 | Academic Research & Papers | Research Scientist prompt + full paper fetch |
| 3 | Product & Application Launches | Enterprise Strategy prompt |
| 4 | Hardware & Infrastructure | Enterprise Strategy prompt |
| 5 | Business, Policy & Geopolitics | Enterprise Strategy prompt + Data Web Search agent activates |

### Classification Prompt
```
Classify this AI news into exactly one class:
1. Model & Foundation Releases (new LLMs/VLMs, benchmarks, API pricing)
2. Academic Research & Papers (ArXiv papers, novel architectures, training methods)
3. Product & Application Launches (new tools, coding assistants, frameworks)
4. Hardware & Infrastructure (GPUs, edge AI, data center innovations)
5. Business, Policy & Geopolitics (funding, regulations, export controls)

Input: {title} — {snippet}

Return JSON: {"news_class": <int>, "confidence": <float>}
```

### Input
Each story's `title + snippet + curator_reasoning` from the Curator's output.

### Output
The `news_class` integer is injected into the LangGraph state:
```python
state["news_class"] = 2
state["classification_confidence"] = 0.94
```

### Routing Logic (LangGraph Conditional Edge)
After classification, a `conditional_edge` function reads `state["news_class"]` and determines:

1. **Which system prompt template** the Writer Agent will receive (5 different prompts, one per class)
2. **Whether the Data Web Search Agent activates** (only for Class 1 and Class 5)
3. **Which evaluation criteria** the Reviewer Agent will apply

### Edge Cases
- If `confidence < 0.7`: Route to a general-purpose prompt template OR flag for human classification in the HITL dashboard.
- In practice, with 5 well-defined classes and a 9B model, expect 95%+ accuracy.
- If a story genuinely spans two classes (e.g., a new model release + regulatory implications), classify by the PRIMARY focus. The Writer can still touch on secondary aspects.

### Key Goal
Fast, cheap, deterministic routing. The SLM Router is the "traffic controller" — it doesn't produce content, it just ensures every story gets sent to the right processing pipeline with the right instructions. Sub-second latency per classification.


---

## Failures & Fallbacks

### SLM Classifier — Misclassification

**What can fail:** Qwen 3.5-9B tags a model release paper as "Academic Research" or a regulatory piece as "Product Launch." Wrong class = wrong Writer prompt = wrong output format.

**Fallback strategy:**
- If `confidence < 0.7`, don't guess. Route to a **fallback classification step** using the Curator agent (DeepSeek V4) as a second opinion — it already has context about the story from Layer 3.
- If both models disagree, use the Curator's classification (it's the stronger model).
- **Downstream safety net:** The Reviewer will catch formatting violations (e.g., Writer used bullet points when a comparison table was required). A misclassification gets caught at review time and triggers a correction — the Reviewer's rejection reason will implicitly reveal the class mismatch.

| Failure Point | Max Retries | Escalation Path | Final Fallback |
|---------------|-------------|-----------------|----------------|
| Classifier (low confidence) | 1 (second opinion from Curator) | Use Curator's classification | Reviewer catches downstream |
