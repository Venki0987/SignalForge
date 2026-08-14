# Layer 9: Storage

## Components
- PostgreSQL Database (Metadata & Content Store)
- Vector Database — Qdrant/Pinecone (RAG Retrieval Store)

---

## PostgreSQL Database

### What it does
The system of record for all finalized content, operational metadata, session logs, and audit trails. Also serves as the source for the chatbot's "Global Anchor" context.

### Schema Design

```sql
-- Core content tables
newsletters (
    run_id UUID PRIMARY KEY,
    publish_date TIMESTAMP,
    status VARCHAR(20),  -- 'in_progress', 'published', 'failed'
    total_stories INT,
    created_at TIMESTAMP
)

stories (
    story_id UUID PRIMARY KEY,
    run_id UUID REFERENCES newsletters(run_id),
    news_class INT,  -- 1-5
    title TEXT,
    writer_draft_final TEXT,  -- Final Markdown
    distilled_context JSONB,  -- The enrichment payload
    data_search_results JSONB,  -- Structured benchmark data (nullable)
    reviewer_status VARCHAR(20),  -- 'approved', 'rejected', 'escalated'
    human_approval_status VARCHAR(20),  -- 'approved', 'edited', 'rejected'
    retry_count INT DEFAULT 0,
    source_urls TEXT[],
    created_at TIMESTAMP,
    published_at TIMESTAMP
)

-- Operational tables
audit_log (
    log_id UUID PRIMARY KEY,
    story_id UUID REFERENCES stories(story_id),
    editor_id UUID,
    action VARCHAR(20),  -- 'approved', 'edited', 'rejected'
    edit_diff TEXT,
    rejection_reason TEXT,
    timestamp TIMESTAMP
)

chat_sessions (
    session_id UUID PRIMARY KEY,
    user_id UUID,
    story_id UUID REFERENCES stories(story_id),  -- Which story they're asking about
    started_at TIMESTAMP,
    last_message_at TIMESTAMP
)

chat_messages (
    message_id UUID PRIMARY KEY,
    session_id UUID REFERENCES chat_sessions(session_id),
    role VARCHAR(10),  -- 'user' or 'assistant'
    content TEXT,
    chunks_used JSONB,  -- Which vector chunks were retrieved for this response
    timestamp TIMESTAMP
)
```

### Dual Role
1. **Operational Store:** Tracks pipeline state, approvals, audit history
2. **Chatbot Anchor Source:** When a user opens the chatbot for a specific story, the `writer_draft_final` is pulled and injected as the system prompt context — this is the "Global Anchor" that gives the chatbot immediate high-level understanding

### Connection
FastAPI backend uses `asyncpg` or SQLAlchemy async for non-blocking database access.

---

## Vector Database (Qdrant/Pinecone)

### What it does
Stores the chunked raw source material (full ArXiv papers, video transcripts) for the RAG chatbot to retrieve from when users ask detailed questions.

### When Data Gets Stored
After a story is published (post-HITL approval), the Sourcer's raw content gets processed and embedded into the vector store.

### Chunking Strategy

**NOT fixed 512-token windows.** We use LangChain Semantic Chunking that respects logical document structure.

#### For Academic Papers:
Split by headers:
- Abstract (1 chunk)
- Introduction (1-2 chunks)
- Methodology (2-4 chunks, depending on length)
- Experiments / Results (2-3 chunks)
- Conclusion (1 chunk)
- References (excluded from chunking)

#### For Video Transcripts:
Split by topic shifts:
- Use embedding similarity between consecutive paragraphs
- When similarity drops below threshold → new chunk boundary
- Ensures coherent topic segments stay together

#### For Articles/Blog Posts:
Split by section headers or logical paragraph groups.

### Chunk Metadata
Each chunk carries:
```json
{
  "story_id": "uuid",
  "run_id": "uuid",
  "news_class": 2,
  "section_name": "Methodology",
  "source_type": "arxiv_paper | transcript | article",
  "chunk_index": 3,
  "total_chunks": 12,
  "original_url": "https://..."
}
```

### Embedding Model
- `text-embedding-3-large` (OpenAI) or `bge-large-en-v1.5` (open-source alternative)
- Dimension: 1024-3072 depending on model choice

### Dual Indexing for Hybrid Search
The vector store supports two retrieval methods simultaneously:
1. **Dense Vectors** — Semantic similarity search (understands meaning)
2. **BM25 Sparse Vectors** — Keyword matching (catches exact terms the dense model might miss)

Qdrant supports both natively via sparse vectors. If using Pinecone, pair with an Elasticsearch instance for BM25.

### Key Goal
Store source material in a way that preserves logical coherence (no severed math formulas or split reasoning chains) and enables fast, accurate retrieval for the chatbot. The chunking strategy is critical — bad chunks = bad chatbot answers.


---

## Failures & Fallbacks

### Database Write Failures (PostgreSQL / Vector DB)

**What can fail:** PostgreSQL is down during publish. Qdrant/Pinecone times out during chunk embedding.

**Fallback strategy:**
- **PostgreSQL down:** Retry 3x. If still failing, write the finalized stories to a local JSON file as a temporary buffer. Fire a critical alert. When PostgreSQL recovers, a recovery job replays the buffered writes.
- **Vector DB down:** Non-blocking for publishing. The newsletter can go live (content is in PostgreSQL and the frontend reads from there). Vector DB embedding is queued and retried. Chatbot will show "Deep dive unavailable for this story" until embeddings are indexed.
- **Partial embedding failure (some chunks fail):** Log which chunks failed. The story is still searchable via the chunks that succeeded. Re-attempt failed chunks on a background retry queue.

| Failure Point | Max Retries | Escalation Path | Final Fallback |
|---------------|-------------|-----------------|----------------|
| DB writes | 3 | Buffer to local JSON | Recovery job on reconnect |
