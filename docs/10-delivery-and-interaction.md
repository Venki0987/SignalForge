# Layer 10: Delivery & Interaction

## Components
- React + Tailwind Frontend (Newsletter Dashboard)
- RAG Chatbot Agent (Qwen 3 8B / GLM-5.1 32B)

---

## React + Tailwind Frontend

### What it does
The user-facing interface. Displays the curated weekly newsletter and embeds the live chatbot for interactive Q&A.

### Implementation Plan

#### Two Main Views

**1. Newsletter View**
- Displays the current week's curated stories, rendered from Markdown
- Filterable by news_class tags (chips/badges for each class)
- Each story card shows:
  - Title
  - Class badge (color-coded)
  - Summary preview (first 2-3 sentences)
  - "Read More" expand to full content
  - Source attribution links
  - "Ask about this" button → opens chatbot pre-scoped to this story
- Archive: Browse previous weeks' newsletters

**2. Chat View**
- Embedded chatbot interface (side panel or dedicated page)
- WebSocket connection for real-time streaming responses
- Conversation history persisted per session
- Context indicator: Shows which story the chatbot is currently grounded in
- Mode toggle: "Fast" (default) vs "Deep Dive" (for in-depth paper queries)

#### API Integration
- `GET /newsletters/latest` — current week's published newsletter
- `GET /newsletters/{run_id}` — specific week
- `GET /newsletters?class={1-5}` — filter by category
- `WebSocket /ws/chat` — real-time chat connection
- `POST /chat/session` — create new chat session scoped to a story

#### Tech Stack
- React (with hooks, no class components)
- Tailwind CSS for styling
- WebSocket client for chat streaming
- Markdown renderer (react-markdown) for story display
- Mobile responsive design

#### Optional: Email Delivery
When a newsletter is published, also distribute via:
- SendGrid or AWS SES for HTML email
- Templated email with story summaries and "Read more on dashboard" links

---

## RAG Chatbot Agent

### What it does
Allows employees to ask follow-up questions about any story, grounded strictly in the actual source material. Prevents the Writer's summary from being the only source of truth — users can dig into the raw paper/transcript.

### Dual-Model Strategy

| Mode | Model | When to Use | Context Budget |
|------|-------|-------------|----------------|
| Fast (default) | Qwen 3 8B | Standard questions about a story | Summary + 3 chunks (~4-5K tokens) |
| Deep Dive | GLM-5.1 32B (200K context) | User explicitly requests deep analysis | Summary + large paper sections (~50-100K tokens) |

### Query Flow (Step by Step)

```
1. User sends question via WebSocket
        ↓
2. Backend identifies which story (from session context or explicit selection)
        ↓
3. Pull finalized summary from PostgreSQL → inject as system prompt "Global Anchor"
        ↓
4. Embed the user's question
        ↓
5. Hybrid retrieval from Vector DB:
   - Dense vector search → top 10 candidates
   - BM25 keyword search → top 10 candidates
   - Merge and deduplicate
        ↓
6. Cohere Re-ranker: top 10 merged → re-ranked → take top 3 chunks
        ↓
7. Construct final prompt:
   - System: summary anchor + retrieved chunks
   - User: original question
        ↓
8. Stream response back via WebSocket
```

### Prompt Construction
```
System: You are a technical assistant helping an engineer understand an AI research topic.

Here is the high-level summary of this topic:
{summary_from_postgres}

Here are relevant excerpts from the source material:
---
[Chunk 1: {section_name}]
{chunk_1_content}
---
[Chunk 2: {section_name}]
{chunk_2_content}
---
[Chunk 3: {section_name}]
{chunk_3_content}
---

Answer the user's question strictly based on the provided context.
If the answer is not in the context, say "I don't have enough information from the source material to answer that."
Do not speculate or add information beyond what's provided.

User: {user_query}
```

### Grounding Guardrails
- The chatbot is strictly grounded — if the answer isn't in the retrieved chunks or the summary, it says so explicitly
- No freestyling, no general knowledge answers
- If the user asks something outside the scope of the stored material, suggest they check the original source links

### Deep Dive Mode
- Triggered by user toggle or explicit request ("explain the full methodology")
- Instead of top 3 chunks, loads entire paper sections (Methodology + Experiments + Results)
- Uses GLM-5.1 32B which can hold 200K tokens without degradation
- Higher latency but much more comprehensive answers

### Re-ranker Integration (Cohere)
- Purpose: Raw vector search returns "relevant-ish" chunks. The re-ranker applies cross-attention between the query and each chunk to find the MOST relevant ones.
- Input: User query + 10 candidate chunks
- Output: Re-ranked scores → select top 3
- This dramatically improves answer quality vs. using raw cosine similarity alone

### Key Goal
Give employees instant, accurate answers about any newsletter topic without context-switching to read full papers. The dual-context approach (PostgreSQL summary as anchor + Vector DB chunks for depth) ensures the chatbot has both breadth and precision. Never hallucinate — if it's not in the sources, say so.


---

## Failures & Fallbacks

### RAG Chatbot — Bad Retrieval or Hallucination

**What can fail:** Vector search returns irrelevant chunks. Re-ranker misjudges relevance. Model answers from general knowledge instead of the provided context.

**Fallback strategy:**
- **Retrieval quality check:** After re-ranking, if the top chunk's relevance score (from Cohere) is below 0.3, the system should respond: "I couldn't find specific information about that in the source material. Here's what the summary says: {summary excerpt}." Don't force an answer from bad chunks.
- **Grounding violation detection:** Add a lightweight post-generation check — does the chatbot's response contain claims NOT present in the provided chunks or summary? If so, regenerate with a stricter temperature (0.1) and a reinforced grounding instruction.
- **Fallback to summary-only mode:** If the Vector DB is slow or returning garbage, the chatbot can still answer from the PostgreSQL summary alone. It says: "Based on the newsletter summary: {answer}. For more detail, I'd recommend checking the source paper directly: {link}."

| Failure Point | Max Retries | Escalation Path | Final Fallback |
|---------------|-------------|-----------------|----------------|
| Chatbot (bad retrieval) | 1 regen at lower temp | Summary-only mode | "Check source directly" response |
