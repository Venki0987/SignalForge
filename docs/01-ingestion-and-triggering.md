# Layer 1: Ingestion & Triggering

## Components
- 7-Day Time Trigger (CRON)
- Scout Node (Pure Python — No LLM)

---

## 7-Day Time Trigger (CRON)

### What it does
Kicks off the entire pipeline on a weekly schedule.

### Implementation Plan
- Use a CRON job scheduler. In production, this could be a cloud-native scheduler (AWS EventBridge, GCP Cloud Scheduler) or a simple system cron if self-hosted.
- The CRON fires an event that triggers the LangGraph DAG execution. It invokes the first node (Scout Agent) by creating a new LangGraph "run" with an empty initial state.
- Include a manual trigger button on the admin dashboard for ad-hoc runs (e.g., breaking news scenarios).
- The scheduler passes a `timestamp` and `run_id` into the initial state so every downstream node can reference which execution cycle it belongs to.

---

## Scout Node (Pure Python — No LLM)

### What it does
The hunter. Goes out to multiple sources and brings back raw signals — URLs, titles, snippets, metadata.

### Why No LLM?
The Scout's job is purely data fetching — "get me the newest posts from these sources." No reasoning, no synthesis, no decision-making. Using an LLM for this was overkill. Benefits of the pure Python approach:
- Zero inference cost on this layer
- Sub-second execution (network latency only)
- No hallucination risk (real URLs, real titles, real content)
- Simpler code — just Python functions, no agent framework needed
- Reduced complexity and latency

### Data Sources & Fetching Strategy

| Source | Library | What to Fetch |
|--------|---------|---------------|
| TechCrunch AI | `feedparser` (RSS) | 6-7 newest blog posts — title, link, summary, published date |
| OpenAI Blog | `feedparser` (RSS) | 6-7 newest blog posts — title, link, summary, published date |
| Anthropic Blog | `feedparser` (RSS) | 6-7 newest blog posts — title, link, summary, published date |
| Reddit (r/MachineLearning, r/artificial, r/LocalLLaMA) | `praw` (Reddit API) | Top/hot posts from the past week — title, url, selftext, score, num_comments |

### Fetching Flow (No Agent, Just Functions)
1. CRON triggers the pipeline, passes `run_id` and `timestamp` into state
2. `fetch_rss_feeds()` — Calls feedparser on each RSS URL, extracts latest 6-7 entries per feed
3. `fetch_reddit_posts()` — Uses PRAW to pull top/hot posts from target subreddits (past 7 days)
4. `normalize_signals()` — Maps both RSS and Reddit results into the unified Signal schema
5. Combined list written to `state["raw_signals"]`

### Libraries Required
```
feedparser    — RSS/Atom feed parsing
praw          — Reddit API wrapper (needs Reddit app credentials)
```

### Reddit Setup (PRAW)
- Create a Reddit "script" app at https://www.reddit.com/prefs/apps
- Store credentials (`client_id`, `client_secret`, `user_agent`) in environment variables
- PRAW free tier is generous — more than enough for a weekly fetch

### Output Schema
```json
{
  "signal_id": "uuid",
  "source": "rss_techcrunch | rss_openai | rss_anthropic | reddit",
  "url": "https://...",
  "title": "...",
  "snippet": "First 200-500 chars of content / selftext",
  "raw_metadata": {
    "authors": [],
    "published_date": "...",
    "subreddit": null,
    "score": null,
    "num_comments": null
  },
  "fetched_at": "ISO timestamp"
}
```

### Normalizer Function
Since RSS and Reddit return different structures, a normalizer maps both into the unified schema:
```python
def normalize_rss_entry(entry, source_name) -> Signal:
    return {
        "signal_id": str(uuid4()),
        "source": f"rss_{source_name}",
        "url": entry.link,
        "title": entry.title,
        "snippet": entry.summary[:500],
        "raw_metadata": {
            "authors": [entry.get("author", "")],
            "published_date": entry.get("published", ""),
        },
        "fetched_at": datetime.utcnow().isoformat()
    }

def normalize_reddit_post(post) -> Signal:
    return {
        "signal_id": str(uuid4()),
        "source": "reddit",
        "url": post.url,
        "title": post.title,
        "snippet": (post.selftext or post.title)[:500],
        "raw_metadata": {
            "subreddit": post.subreddit.display_name,
            "score": post.score,
            "num_comments": post.num_comments,
            "published_date": datetime.fromtimestamp(post.created_utc).isoformat(),
        },
        "fetched_at": datetime.utcnow().isoformat()
    }
```

### Error Handling
- Each source fetch is wrapped in a try/except with 3 retries and exponential backoff (2s → 4s → 8s).
- If a source is completely down, the function logs a warning and continues with other sources — it does NOT fail the entire run.
- Failed fetches are logged in the observability layer with the error reason.
- Reddit rate limiting: PRAW handles this internally (auto-sleeps when rate-limited).

### Key Goal
Produce a raw list of AI signal objects from the past 7 days using simple, deterministic Python functions. No LLM needed. Prioritize reliability and simplicity. Never crash the pipeline due to a single source failure.


---

## Failures & Fallbacks

### Scout Node — External API Failures

**What can fail:** RSS feed URL changed or returns 404, Reddit API rate-limits or auth token expired, network timeout.

**Fallback strategy:**
- 3 retries with exponential backoff (2s → 4s → 8s) per source — already planned.
- **Partial success is acceptable.** If 2 out of 4 sources return data, the run continues with what it has. Don't fail the entire pipeline because one source is flaky.
- Log which sources failed. If the SAME source fails 3 consecutive weekly runs, fire an alert to the team — likely an RSS URL changed or Reddit credentials expired.
- **Minimum viable threshold:** If fewer than 5 total signals come back from ALL sources combined, abort the run and notify the admin. Something systemic is wrong.

| Failure Point | Max Retries | Escalation Path | Final Fallback |
|---------------|-------------|-----------------|----------------|
| Scout (source down) | 3 per source | Continue with partial data | Abort run if < 5 signals total |
