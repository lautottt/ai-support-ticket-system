# AI Customer Support Automation — v2

An automated support ticket processing system built with n8n, Claude AI, and Supabase.  
Receives tickets via Gmail, processes them with AI, stores everything in a structured database,  
and routes draft responses to a human agent for approval before sending.

---

## What changed from v1 — and why

The first version of this system worked. It detected multiple issues in a single ticket, classified them, generated responses, and composed a final message for the customer. The prompts were solid and the outputs were reliable.

But when I rebuilt it from scratch, I focused on three things I had ignored the first time: **cost efficiency**, **reliability under real conditions**, and **scalability for selling to clients**.

Here is what changed and the reasoning behind each decision.

---

### 1. From 15 LLM calls to 5 — a 67% cost reduction

**v1 architecture (3 issues = 15 calls):**
```
Data Extraction          → 1 call
Issue Extraction         → 1 call
Per issue × 3:
  Classification         → 3 calls
  Summary                → 3 calls
  Action Generator       → 3 calls
  Customer Response      → 3 calls
Final Composer           → 1 call
Total                    = 15 calls
```

**v2 architecture (3 issues = 5 calls):**
```
Data + Issue Extraction  → 1 call  (combined)
Per issue × 3:
  Classify + Summary
  + Action + Response    → 3 calls  (combined into one per issue)
Final Composer           → 1 call
Total                    = N + 2 calls
```

The key insight: each of those original prompts was doing one small task and passing the result to the next. They were modular for the sake of modularity, not because the separation added value. A single well-structured prompt with sequential steps produces the same output quality at a fraction of the cost.

The prompts now use numbered steps where each step explicitly builds on the previous one. The model follows sequential instructions reliably — no quality loss, significant cost reduction.

---

### 2. A free guard node before any AI call

In production, a support inbox receives a lot of noise: auto-replies, delivery failures, out-of-office messages, newsletters. The first version had no filter — every email triggered an LLM call.

The guard node runs before any AI call and costs nothing:

- Rejects auto-replies and bounces by subject pattern
- Rejects no-reply addresses
- Rejects emails with bodies under 10 characters
- Truncates bodies over 3000 characters before passing to the LLM

In a real inbox, 30-40% of emails are noise. This node eliminates that entire cost.

---

### 3. Human-in-the-loop approval before sending

v1 sent responses automatically. That is fine for demos but not for selling to real businesses — no ecommerce company will let an AI send emails to customers without a review step.

The approval flow works like this:
1. The processed ticket is saved to Supabase with status `pending_review`
2. The agent receives an email with the ticket summary, detected issues, and the draft response
3. The email contains two links: **Approve** and **Needs Edit**
4. Each link hits a dedicated n8n webhook using a unique token
5. Approve → sends the email to the customer → marks ticket as `sent`
6. Needs Edit → marks ticket as `needs_edit` → agent edits directly in Supabase

The token is a UUID generated per ticket. It makes approval links impossible to guess and ties each action to the exact ticket it belongs to.

---

### 4. Supabase as the data layer

v1 had no persistent storage. Processing happened in memory and the output went directly to email. There was no record of what the system decided, why, or what happened next.

v2 stores the full ticket lifecycle in Supabase:

- Raw email — for re-processing if something fails
- All detected issues with classification, priority, actions, and confidence scores — stored as JSONB, fully queryable
- Final response draft
- Approval status and timestamps
- A `company_id` column built in from day one for multi-tenant deployment

This turns the system from a processing pipeline into something you can build a business on. You can filter tickets by category, measure response times, identify recurring issue patterns, and audit every decision the AI made.

---

### 5. Model selection: Claude Haiku 3.5

All prompts in this system output structured JSON. The tasks are well-defined: extract, classify, summarize, generate. These do not require a frontier model.

Claude Haiku 3.5 costs ~$0.80 per million input tokens. It follows strict output schemas reliably and handles all the prompt instructions without deviation. Using a more expensive model here would add cost with no measurable improvement in output quality for these specific tasks.

The upgrade path is intentionally simple: swap the model node for a single issue without changing anything else.

---

## Architecture

```
Gmail Trigger
  │
  ├─ Get Full Email (Gmail API)
  │
  ├─ Guard (Code node — zero tokens)
  │   Filters: auto-replies, bounces, no-reply addresses, short bodies
  │   Truncates: body to 3000 characters
  │
  ├─ LLM Call 1 — Data + Issue Extraction
  │   Output: { client_data, issues[] }
  │
  ├─ Parse + Split Issues (Code node)
  │   Splits issues array into individual items for the loop
  │
  └─ Loop Over Items
      │
      ├─ LLM Call 2 — Per-issue processing
      │   Step 1: Classification (category + priority)
      │   Step 2: Summary (2 sentences)
      │   Step 3: Action (recommended + fallback)
      │   Step 4: Customer response draft
      │
      └─ Parse + Validate (Code node — zero tokens)
          Validates all required fields and enum values
          Throws on invalid output — fails loudly, not silently
  │
  ├─ Aggregate Issues (Code node)
  │   Collects all processed issues into a single item
  │
  ├─ LLM Call 3 — Final Response Composer
  │   Merges all issue responses into one coherent message
  │   Orders by priority: high → medium → low
  │
  ├─ Parse Final Response (Code node)
  │   Generates approval token (UUID)
  │
  ├─ Save to Supabase
  │   status: "pending_review"
  │
  └─ Send approval email to agent
      Contains: customer info, issues detected, draft response, action links
           │
           ├─ Agent clicks Approve
           │   → Webhook → Update status: approved
           │   → Send email to customer
           │   → Update status: sent
           │
           └─ Agent clicks Needs Edit
               → Webhook → Update status: needs_edit
```

---

## Stack

| Component | Service | Cost |
|---|---|---|
| Workflow automation | n8n (self-hosted, Docker) | Free |
| Email input/output | Gmail | Free |
| LLM | Claude Haiku 3.5 | ~$0.80/M input tokens |
| Database | Supabase (PostgreSQL) | Free tier |
| Agent notification | Gmail | Free |
| Tunnel (development) | Cloudflare Tunnel | Free |

Total infrastructure cost for a PoC: $0 (plus LLM usage per ticket).

---

## Supabase Schema

Table: `tickets`

| Column | Type | Notes |
|---|---|---|
| id | uuid | Auto-generated primary key |
| created_at | timestamptz | Ticket received timestamp |
| company_id | text | Multi-tenant support from day one |
| customer_email | text | Extracted by LLM |
| customer_name | text | Extracted by LLM |
| client_id | text | Extracted by LLM |
| order_number | text | Extracted by LLM |
| raw_email | text | Original body, unmodified |
| issues | jsonb | Full array of processed issues |
| final_response | text | Draft from Final Composer |
| status | text | pending_review / approved / sent / needs_edit |
| agent_notes | text | Free field for the agent |
| approval_token | uuid | Unique token per ticket |
| approved_at | timestamptz | Approval timestamp |
| sent_at | timestamptz | Send timestamp |

---

## What's next

- [ ] Dashboard to monitor ticket volume, response times, and issue categories
- [ ] Customer history lookup — query past tickets by client_id before generating a response
- [ ] Slack/Teams notification as an alternative to email approval
- [ ] Retry logic for failed LLM calls
- [ ] Multi-tenant workflow separation per client
