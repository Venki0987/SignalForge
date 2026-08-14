# Layer 8: Human Oversight

## Component
- Human-in-the-Loop (HITL) Dashboard

---

## What it does
Final editorial gate before publishing. A human editor reviews what the AI system produced, ensuring nothing slips through that the automated Reviewer might have missed (tone, editorial judgment, strategic relevance).

---

## Implementation Plan

### Dashboard Interface
A dedicated page in the React frontend that shows all "APPROVED" stories awaiting publication for the current weekly run.

### What the Editor Sees (Per Story)

| Element | Description |
|---------|-------------|
| Rendered Markdown Draft | The Writer's final output, beautifully rendered |
| News Class Badge | Color-coded tag (Class 1-5) |
| Reviewer Approval Reasoning | Why DeepSeek R1 approved it |
| Source Links | Original URLs so the editor can spot-check claims |
| Distilled Context (collapsible) | The raw source material used for generation |
| Data Web Search Results (if any) | The structured JSON with benchmark data |
| Retry History (if any) | How many rewrites happened, what was fixed |

### Editor Actions

1. **Approve** — Story goes to publishing as-is
2. **Edit** — Inline Markdown editor for minor tweaks (typos, tone adjustments, adding context). Edited version becomes the final.
3. **Reject** — Story is killed with a reason. Gets logged but never published. Useful if the story became outdated during pipeline processing or if the editor disagrees with the Curator's relevance judgment.

### Publishing Flow
- Once ALL stories for a weekly run are reviewed (approved or rejected), the editor clicks "Publish."
- This triggers:
  1. Approved stories get written to PostgreSQL as finalized
  2. Source material gets chunked and embedded into the Vector DB
  3. The newsletter is assembled and pushed to the frontend
  4. (Optional) Email distribution via SendGrid/SES

### Backend Implementation
- FastAPI endpoints:
  - `GET /admin/pending-reviews/{run_id}` — fetch all stories awaiting review
  - `PUT /admin/review/{story_id}` — submit decision (approve/edit/reject)
  - `POST /admin/publish/{run_id}` — trigger publishing pipeline
- Authentication: Editor must be authenticated (role-based access control)

### Audit Trail
Every human decision is logged with:
- `editor_id`
- `action` (approved / edited / rejected)
- `timestamp`
- `edit_diff` (if edited, store the before/after)
- `rejection_reason` (if rejected)

This creates accountability and allows analysis of how often the AI output needs human correction (metric for system quality).

### Key Goal
Ensure a human always has final say before content reaches readers. The AI does 95% of the work, but this layer catches edge cases that automated systems miss: editorial judgment, strategic timing, tone appropriateness. Keep the interface fast and frictionless — the editor should be able to review 10-20 stories in under 30 minutes.


---

## Failures & Fallbacks

### HITL Gate — Human Never Responds

**What can fail:** The editor is on vacation, busy, or forgets. Stories sit in the HITL queue indefinitely and the newsletter never publishes.

**Fallback strategy:**
- **48-hour timeout with escalation chain:**
  - 0h: Story enters HITL queue. Notification sent.
  - 24h: Reminder notification.
  - 48h: Auto-escalate to a secondary editor (backup reviewer).
  - 72h: If no human has acted, auto-publish all "APPROVED" stories with a "Fast-tracked — not human-reviewed" tag visible in the dashboard. This is a business decision — better to publish a Reviewer-approved-but-not-human-reviewed newsletter than to publish nothing.
- **Critical stories exception:** Stories flagged as "contains regulatory/legal claims" (Class 5) NEVER auto-publish. They wait until a human reviews them, no matter how long.

| Failure Point | Max Retries | Escalation Path | Final Fallback |
|---------------|-------------|-----------------|----------------|
| HITL (human unresponsive) | 48h timeout | Escalate to backup editor | Auto-publish at 72h (except Class 5) |
