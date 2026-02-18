# n8n Architecture Proposal: Slack → Freshdesk Ticketing + Knowledge Base Automation

## Executive summary
For a law-firm environment (high compliance, low tolerance for noisy tickets), the best design is a **hybrid approach**:

1. **Multiple specialized n8n workflows** for orchestration, state, retries, and auditability.
2. **A constrained agentic layer (OpenAI)** only for tasks where language understanding adds value (classification, confusion detection, professional rewriting, semantic deduplication, KB drafting).

This avoids a monolithic “do everything” agent and gives better control, transparency, and reliability.

---

## Decision: multiple workflows vs pure agentic

### Recommendation
Use **multiple workflows + controlled agentic nodes**.

### Why this is the right fit
- **Auditability and governance:** legal operations need explicit decision points and traceable logs.
- **Ticket quality control:** deterministic rules + AI reduces false positives and low-quality tickets.
- **Operational resilience:** integrations can fail independently without taking down the whole system.
- **Maintainability:** Slack/Freshdesk/API changes are isolated to one workflow at a time.
- **Cost control:** OpenAI is used only where it materially improves outcomes.

---

## End-to-end workflow design

## Workflow 1 — Slack Intake, Qualification, Deduplication, and Ticket Creation
**Trigger:** Slack new message event in approved support channels.

**Goal:** determine whether the message is a real support issue, enrich and structure content, prevent duplicates, create a Freshdesk ticket, and reply in the same Slack thread.

### Core steps
1. **Hard-rule prefilter**
   - Ignore bot messages, edits-only events, non-message events, and unsupported channels.
   - Ignore threads already linked to an existing ticket.

2. **Normalize the message payload**
   - Capture: `channel_id`, `thread_ts`, `message_ts`, `user_id`, text, links, and attachments.

3. **Attachment ingestion**
   - Download Slack files/screenshots.
   - Compute hashes for dedupe checks.
   - Preserve metadata (filename, mime type, source URL).

4. **OpenAI classification + enrichment (JSON only)**
   - Required output schema:
     - `is_issue` (boolean)
     - `confidence` (0-1)
     - `is_confusing` (boolean; conservative true threshold)
     - `category` (access, software, hardware, network, security, other)
     - `urgency` (low, medium, high, critical)
     - `criticality_reason` (string)
     - `title` (string)
     - `rewritten_description` (professional and clear)
     - `entities` (system, module, link, user context, error indicators)

5. **Decision gate**
   - If not a valid issue or confidence is below threshold: post a friendly clarification request in-thread and stop.

6. **Duplicate detection cascade**
   - **Exact fingerprint:** user + normalized text + extracted links + file hashes.
   - **Freshdesk lookup:** similar subject/reporter/recent window.
   - **Semantic similarity:** embeddings against recent tickets.
   - If duplicate: post duplicate response with existing ticket URL and stop.

7. **Freshdesk ticket creation**
   - Create ticket with generated title/description/category/urgency.
   - Attach original files/screenshots.
   - Include “Original Slack Message” section for legal/audit traceability.

8. **Slack confirmation in same thread**
   - Friendly response with ticket URL, severity, and what happens next.

9. **State persistence**
   - Store mapping: `ticket_id ↔ channel_id ↔ thread_ts`.
   - Store AI decision payload and dedupe evidence.

---

## Workflow 2 — Freshdesk Activity Sync Back to Slack Thread
**Trigger:** Freshdesk webhooks (status changes, public notes, agent replies, assignee changes, SLA updates).

**Goal:** keep the original Slack thread synchronized with ticket activity.

### Core steps
1. Resolve `ticket_id` to Slack thread mapping.
2. Normalize and sanitize update text.
3. Post update in the same Slack thread.
4. Prevent loops (ignore bot-originated or already-synced events).

---

## Workflow 3 — KB Candidate Detection and Draft Generation
**Trigger:** Daily scheduled job (optional additional trigger on ticket resolution).

**Goal:** detect recurring issues and generate draft KB articles (never auto-publish).

### Core steps
1. Retrieve recently resolved/closed tickets.
2. Cluster recurring issue patterns by category + semantic similarity.
3. Detect whether KB article already exists.
4. Decide draft-vs-update path:
   - Existing article with gaps → draft update.
   - No suitable article → draft new article.
5. Generate draft article (OpenAI) in professional structure:
   - Title
   - Summary
   - Environment
   - Symptoms / Reproduction
   - Root cause (if known)
   - Resolution steps
   - Alternative solutions / variants
   - Edge cases
   - Related ticket references
6. Save as draft in Freshdesk.
7. Send to Slack KB review channel with approval actions.

---

## Workflow 4 — KB Review, Approval, and Publishing
**Trigger:** Slack interactive action (approve/edit/reject).

### Core steps
1. Validate approver role (support lead or authorized reviewer).
2. Approve → publish in Freshdesk automatically.
3. Edit → apply modifications and keep draft or publish per action.
4. Reject → log reason and notify reviewer channel.
5. Persist full audit trail (who approved, what changed, when).

---

## Key technical components
- **n8n:** orchestration and integrations.
- **OpenAI:** issue understanding, rewriting, semantic clustering, and KB drafting.
- **PostgreSQL (recommended):**
  - thread-ticket mapping
  - dedupe fingerprints and similarity metadata
  - AI decision snapshots for audit
- **Object storage (optional):** file retention and hash cache.
- **Slack API + Freshdesk API + webhooks.**

---

## AI guardrails (critical for legal operations)
1. Force **strict JSON outputs** via schema validation.
2. Use **low temperature** for classification (0.1–0.2).
3. Use uncertainty policy:
   - If uncertain, ask for clarification before creating ticket.
4. Never fabricate missing data.
5. Always preserve original user wording alongside rewritten text.
6. Mark `is_confusing=true` only when truly ambiguous/incoherent.

---

## Deduplication policy
Use layered scoring:
1. Exact match
2. Fuzzy lexical similarity
3. Semantic similarity
4. Contextual signals (same user/link/file hash/time window)

Suggested thresholds:
- `> 0.88` → duplicate
- `0.75 - 0.88` → likely duplicate (flag for internal confirmation)
- `< 0.75` → create new ticket

---

## Slack response style (friendly and empathetic)

**Ticket created**
> ✅ Thanks for reporting this, {{name}}. I created ticket **#{{id}}**: {{url}}.
> I marked it as **{{urgency}}** under **{{category}}**.
> I’ll keep this thread updated 🙌

**Need clarification (only when truly needed)**
> Thanks for flagging this 🙌 To help us triage correctly, could you share:
> 1) What you were trying to do
> 2) What happened instead
> 3) Since when this started
> 4) Any screenshot/error message (if available)

**Duplicate detected**
> 👀 It looks like this is already tracked in **#{{id}}**: {{url}}.
> To avoid duplicate tickets, we’ll continue there. I’ll still follow up here if needed 🙏

---

## Recommended Freshdesk fields
- `subject`
- `description`
- `priority`
- `type/category`
- `tags`
- `custom_fields`:
  - `slack_channel_id`
  - `slack_thread_ts`
  - `reported_by`
  - `is_confusing`
  - `criticality_score`
  - `dedupe_fingerprint`

---

## Suggested rollout plan
1. **Phase 1 (controlled MVP)**
   - Workflows 1 and 2
   - Exact + basic semantic dedupe
2. **Phase 2**
   - Workflows 3 and 4
   - Draft + Slack approval gate
3. **Phase 3**
   - Continuous KB updates based on new edge cases
   - Model quality metrics and periodic threshold tuning

---

## KPIs
- Support-message detection recall
- Ticket creation precision
- Duplicate prevention rate
- Slack-message-to-ticket latency
- KB approval rate without edits
- Reduction in repeated incidents after KB publishing

---

## Final recommendation
For your requirements, **build this with multiple workflows** and a **controlled agentic layer**.

This architecture gives you clean ticketing, better legal-grade traceability, reduced duplication/noise, and reliable Slack thread continuity.
