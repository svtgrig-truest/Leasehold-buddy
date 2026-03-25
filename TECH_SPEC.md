# Common Ground — Technical Design Spec

**Document status:** v1 (2026-03-25)
**Product name:** Common Ground
**Working tagline:** Shared case management and evidence coordination for your block.

---

## 1. Overview

Common Ground is an AI case manager for leasehold buildings. It lives in the building's WhatsApp group, supports private 1:1 resident chats, and provides a director-facing web dashboard for case management, document library, and draft approval.

---

## 2. Tech stack

| Layer | Technology | Reason |
|-------|-----------|--------|
| Frontend + API | Next.js 15 (App Router) | Full-stack in one repo, fast deploy |
| Database | Supabase (PostgreSQL) | Auth, RLS, storage, pgvector, cron |
| Hosting | Vercel | Zero-config Next.js deploy |
| Chat (WhatsApp) | Meta WhatsApp Cloud API | Free tier, no Twilio markup |
| Chat (Facebook) | Facebook Messenger API | V2 — channel adapter placeholder |
| AI | Claude Sonnet (Anthropic API) | All tasks: summarize, analyze, draft |
| Embeddings | OpenAI text-embedding-3-small | $0.02/1M tokens, strong English quality |
| Vector search | Supabase pgvector | Native, no extra service |
| File storage | Supabase Storage | PDFs, photos, per-building buckets |
| Payments | Stripe | Per-building subscription |
| PDF parsing | pdf-parse (Node.js) | Text extraction from text-based PDFs |
| Cron | Supabase pg_cron | Daily reminder checks |
| Auth | Supabase Auth (magic link) | Director only, no password management |

---

## 3. Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    CHAT LAYER                           │
│                                                         │
│  WhatsApp Group           WhatsApp 1:1 (private)        │
│  (building chat)          (resident ↔ bot)              │
└──────────────┬────────────────────────┬────────────────┘
               │                        │
               ▼                        ▼
┌─────────────────────────────────────────────────────────┐
│              CHANNEL ADAPTER (Next.js API route)        │
│                                                         │
│  /api/webhook/whatsapp                                  │
│  → identifies building by group_id or sender phone     │
│  → routes to MessageProcessor                           │
│                                                         │
│  [FacebookAdapter — V2 placeholder]                     │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│              MESSAGE PROCESSOR                          │
│                                                         │
│  1. Classify: document upload / @mention / group msg   │
│  2. Save raw event to chat_events table                │
│  3. Enqueue heavy jobs to Supabase Edge Function       │
│  4. Return 200 immediately (webhook ACK)               │
└──────────────────────────┬──────────────────────────────┘
                           │ (async)
                           ▼
┌─────────────────────────────────────────────────────────┐
│         SUPABASE EDGE FUNCTION: process_event           │
│                                                         │
│  Document jobs:                                        │
│    - PDF → pdf-parse → text                            │
│    - Photo → Claude vision → text                      │
│    - text → chunk → embed (OpenAI) → pgvector          │
│    - Claude Sonnet: classify doc type, extract fields  │
│    - Save to documents, document_chunks tables         │
│    - Post summary to WhatsApp group                    │
│                                                         │
│  Message jobs:                                         │
│    - @mention → Claude: answer in context (RAG)        │
│    - Resident evidence → create evidence_item          │
│    - Update case status if needed                      │
│                                                         │
│  Draft jobs:                                           │
│    - Build context (case + claims + evidence + docs)   │
│    - Claude Sonnet: generate 2-3 draft options         │
│    - Post to group chat for discussion                 │
│    - Save to draft_actions table                       │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                   SUPABASE                              │
│                                                         │
│  PostgreSQL (data + RLS)                               │
│  pgvector (document embeddings)                        │
│  Storage (PDFs, photos — per-building buckets)         │
│  Auth (magic link — director only)                     │
│  pg_cron (daily reminder job)                          │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│            DIRECTOR WEB DASHBOARD (Next.js)             │
│                                                         │
│  /dashboard           — case list + status overview    │
│  /dashboard/cases/[id] — case view (timeline, claims,  │
│                          evidence, drafts, approvals)  │
│  /dashboard/library   — searchable document archive    │
│  /dashboard/drafts    — draft workspace + approve      │
│  /dashboard/settings  — building config, residents     │
└─────────────────────────────────────────────────────────┘
```

**Key principle:** webhook → ACK (< 200ms) → async Edge Function does heavy work. Avoids Vercel timeout on webhook endpoint.

---

## 4. Database schema

### 4.1 Multi-tenancy model

Every table (except `auth.users`) is scoped to `building_id`. Supabase Row-Level Security (RLS) enforces isolation: a director can only read/write rows for their building.

### 4.2 Tables

```sql
-- =============================================
-- BUILDINGS (one per leasehold block)
-- =============================================
CREATE TABLE buildings (
  id                       uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name                     text NOT NULL,
  address                  text,
  whatsapp_group_id        text UNIQUE,       -- Meta group thread ID
  whatsapp_phone_number_id text,              -- Meta Cloud API phone number ID
  stripe_customer_id       text,
  stripe_subscription_id   text,
  subscription_status      text DEFAULT 'trialing',
  created_at               timestamptz DEFAULT now()
);

-- =============================================
-- PROFILES (director web auth + resident metadata)
-- =============================================
CREATE TABLE profiles (
  id            uuid PRIMARY KEY REFERENCES auth.users ON DELETE CASCADE,
  building_id   uuid REFERENCES buildings,
  role          text CHECK (role IN ('director', 'resident')) DEFAULT 'resident',
  flat_number   text,
  whatsapp_phone text,
  display_name  text,
  created_at    timestamptz DEFAULT now()
);

-- =============================================
-- RESIDENTS (whatsapp phone → building + flat)
-- Populated when a resident first messages the bot
-- =============================================
CREATE TABLE residents (
  id                uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  building_id       uuid REFERENCES buildings NOT NULL,
  whatsapp_phone    text NOT NULL,
  flat_number       text,               -- null until they self-register
  display_name      text,
  registered_at     timestamptz,        -- when they told the bot their flat
  profile_id        uuid REFERENCES profiles,
  UNIQUE (building_id, whatsapp_phone)
);

-- =============================================
-- CASES (one per dispute / issue)
-- =============================================
CREATE TABLE cases (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  building_id uuid REFERENCES buildings NOT NULL,
  title       text NOT NULL,
  description text,
  case_type   text CHECK (case_type IN (
    'service_charge', 'repair', 'contractor',
    'communication', 'major_works', 'other'
  )),
  status      text CHECK (status IN (
    'intake', 'waiting_resident_evidence',
    'waiting_agent_reply', 'under_review',
    'complaint_ready', 'closed'
  )) DEFAULT 'intake',
  created_at  timestamptz DEFAULT now(),
  updated_at  timestamptz DEFAULT now()
);

-- =============================================
-- DOCUMENTS (PDFs, photos, emails ingested)
-- =============================================
CREATE TABLE documents (
  id                 uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  building_id        uuid REFERENCES buildings NOT NULL,
  case_id            uuid REFERENCES cases,       -- null until linked to a case
  storage_path       text,                        -- Supabase Storage path
  original_filename  text,
  doc_type           text CHECK (doc_type IN (
    'invoice', 'service_charge_demand', 'repair_update',
    'estimate', 'complaint_reply', 'contractor_update',
    'consultation_notice', 'lease_extract', 'correspondence',
    'historical_account', 'photo_evidence', 'other'
  )),
  extracted_text     text,
  extracted_metadata jsonb,  -- {dates, amounts, parties, flat_numbers, deadlines, red_flags}
  ingested_at        timestamptz DEFAULT now(),
  ingested_by_phone  text,   -- which WhatsApp number forwarded it
  visibility         text DEFAULT 'building' CHECK (visibility IN ('building', 'private'))
);

-- =============================================
-- DOCUMENT CHUNKS (for RAG retrieval)
-- =============================================
CREATE TABLE document_chunks (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id  uuid REFERENCES documents ON DELETE CASCADE,
  building_id  uuid REFERENCES buildings NOT NULL,
  chunk_index  int NOT NULL,
  content      text NOT NULL,
  embedding    vector(1536),  -- OpenAI text-embedding-3-small
  metadata     jsonb          -- doc_type, date, page_number
);

CREATE INDEX ON document_chunks USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);

-- =============================================
-- CLAIMS (statements: agent vs resident reality)
-- =============================================
CREATE TABLE claims (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  case_id             uuid REFERENCES cases NOT NULL,
  building_id         uuid REFERENCES buildings NOT NULL,
  content             text NOT NULL,
  claimed_by          text CHECK (claimed_by IN (
    'managing_agent', 'contractor', 'resident',
    'historical_document', 'bot_inference'
  )),
  source_document_id  uuid REFERENCES documents,
  source_message_text text,
  flat_number         text,  -- which flat this claim concerns, if flat-specific
  status              text CHECK (status IN (
    'unverified', 'confirmed', 'disputed', 'uncertain'
  )) DEFAULT 'unverified',
  visibility          text DEFAULT 'building',
  created_at          timestamptz DEFAULT now()
);

-- =============================================
-- EVIDENCE ITEMS (attached to claims)
-- Private by default — resident opts to share
-- =============================================
CREATE TABLE evidence_items (
  id                 uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  claim_id           uuid REFERENCES claims,
  building_id        uuid REFERENCES buildings NOT NULL,
  content            text,
  evidence_type      text CHECK (evidence_type IN (
    'photo', 'message', 'document', 'prior_record', 'resident_statement'
  )),
  storage_path       text,
  submitted_by_phone text,
  flat_number        text,
  visibility         text DEFAULT 'private' CHECK (visibility IN ('private', 'anonymous', 'named')),
  shared_at          timestamptz,  -- null until resident opts to share
  created_at         timestamptz DEFAULT now()
);

-- =============================================
-- DRAFT ACTIONS (bot-proposed formal responses)
-- =============================================
CREATE TABLE draft_actions (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  case_id        uuid REFERENCES cases NOT NULL,
  building_id    uuid REFERENCES buildings NOT NULL,
  action_type    text CHECK (action_type IN (
    'request_clarification', 'request_documents',
    'follow_up_delay', 'formal_complaint',
    'contradiction_summary', 'case_summary'
  )),
  tone           text CHECK (tone IN ('neutral', 'firm', 'urgent')) DEFAULT 'neutral',
  content        text NOT NULL,          -- the draft letter text
  evidence_cited jsonb,                  -- array of evidence_item ids and summaries
  status         text CHECK (status IN (
    'proposed', 'approved', 'rejected', 'edited'
  )) DEFAULT 'proposed',
  whatsapp_message_id text,             -- message ID when posted to group
  created_at     timestamptz DEFAULT now()
);

-- =============================================
-- APPROVALS (audit log of director decisions)
-- =============================================
CREATE TABLE approvals (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  draft_action_id  uuid REFERENCES draft_actions NOT NULL,
  building_id      uuid REFERENCES buildings NOT NULL,
  approved_by      uuid REFERENCES profiles NOT NULL,
  decision         text CHECK (decision IN ('approved', 'rejected')),
  edited_content   text,  -- if director edited before approving
  decided_at       timestamptz DEFAULT now()
);

-- =============================================
-- CHAT EVENTS (raw message log for building memory)
-- =============================================
CREATE TABLE chat_events (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  building_id         uuid REFERENCES buildings NOT NULL,
  whatsapp_message_id text UNIQUE,
  sender_phone        text,
  message_type        text CHECK (message_type IN (
    'text', 'image', 'document', 'audio', 'video'
  )),
  content             text,
  storage_path        text,      -- if file attachment
  is_group_message    boolean DEFAULT true,
  case_id             uuid REFERENCES cases,  -- linked after processing
  processed           boolean DEFAULT false,
  created_at          timestamptz DEFAULT now()
);

-- =============================================
-- OPEN QUESTIONS (tracked per case)
-- =============================================
CREATE TABLE open_questions (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  case_id         uuid REFERENCES cases NOT NULL,
  building_id     uuid REFERENCES buildings NOT NULL,
  question        text NOT NULL,
  asked_at        timestamptz DEFAULT now(),
  resolved_at     timestamptz,
  days_since_asked int GENERATED ALWAYS AS (
    EXTRACT(DAY FROM now() - asked_at)::int
  ) STORED
);
```

### 4.3 Row-Level Security (RLS)

```sql
-- Example: director can only see their building's cases
ALTER TABLE cases ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Director access own building"
  ON cases FOR ALL
  USING (
    building_id IN (
      SELECT building_id FROM profiles WHERE id = auth.uid()
    )
  );
-- Same pattern applied to all tables
```

---

## 5. WhatsApp integration

### 5.1 Setup

- One WhatsApp Business Account with one phone number (MVP)
- Buildings identified by `whatsapp_group_id` (thread ID in webhook payload)
- For 1:1 messages: `sender_phone` is looked up in `residents` table → `building_id`

### 5.2 Webhook flow

```
Meta Platform → POST /api/webhook/whatsapp
  ↓
Verify token check (GET /api/webhook/whatsapp)
  ↓
Parse message:
  - Extract: message_id, sender_phone, group_id (if group), message_type, content/media
  - Identify building: group_id → buildings.whatsapp_group_id
    OR sender_phone → residents.whatsapp_phone → building_id
  ↓
Save to chat_events (processed = false)
  ↓
Return 200 immediately
  ↓ (async via Supabase Realtime trigger or direct Edge Function call)
Edge Function: process_event(chat_event_id)
```

### 5.3 Bot triggers

In the group chat, the bot activates when:
1. **Document attachment** — message contains `type: image` or `type: document`
2. **@mention** — message text contains the bot's display name or a `/` command

In 1:1 chat, the bot responds to all messages.

### 5.4 Resident self-registration

When a new phone number messages the bot for the first time (1:1 or in group):
```
Bot: "Hi! I'm Common Ground, the AI case manager for your building.
      What's your flat number? (e.g. Flat 12)"
Resident: "Flat 7"
Bot: "Thanks! You're registered as Flat 7.
      You can send me documents, photos, or questions privately here."
```
Saves `residents` row with `building_id`, `whatsapp_phone`, `flat_number`.

### 5.5 Channel adapter interface

```typescript
interface Channel {
  sendMessage(buildingId: string, recipientId: string, message: OutboundMessage): Promise<void>
  sendGroupMessage(buildingId: string, text: string): Promise<void>
  downloadMedia(mediaId: string): Promise<Buffer>
}

type OutboundMessage = {
  text: string
  replyToMessageId?: string
}

// Implementations:
class WhatsAppChannel implements Channel { ... }
class FacebookChannel implements Channel { ... }  // V2 placeholder
```

---

## 6. Document processing pipeline

### 6.1 PDF processing

```
1. Meta webhook → download PDF via media_id → Supabase Storage
2. pdf-parse → extract text
   - If text length > 100 chars: use extracted text
   - If text is empty/too short (scanned PDF): fall back to Claude vision
3. Claude Sonnet: classify doc type + extract structured metadata:
   {
     doc_type: "service_charge_demand",
     dates: ["2026-01-15"],
     amounts: [{ label: "Cleaning", amount: 3200, currency: "GBP" }],
     parties: ["Buildium Management Ltd"],
     flat_numbers: ["Flat 7", "Flat 12"],
     deadlines: ["2026-02-28"],
     red_flags: ["Amount increased 47% vs prior year"],
     missing_items: ["Breakdown of cleaning costs", "Contractor invoices"]
   }
4. Save to documents table
5. Chunk text (800 tokens, 100 token overlap)
6. Embed each chunk with OpenAI text-embedding-3-small
7. Save chunks + embeddings to document_chunks
8. Generate plain-English summary (Claude Sonnet)
9. Post summary to WhatsApp group
```

### 6.2 Photo processing

```
1. Download image → Supabase Storage
2. Send directly to Claude vision (base64):
   - Prompt: "This is a letter/invoice/photo from a leaseholder.
     Extract all text. Classify document type.
     Note any unusual amounts, missing information, or red flags."
3. Continue with steps 3-9 from PDF pipeline
```

### 6.3 Summary format posted to WhatsApp group

```
📄 *New document received*
Type: Service Charge Demand

📝 *Summary:*
The managing agent is requesting £3,200 for the annual service charge,
due by 28 February. This is 47% higher than last year's £2,178.

⚠️ *Red flags:*
- No breakdown of cleaning costs provided
- Contractor invoices not attached
- Increase not explained in the letter

❓ *Missing:*
- Supporting invoices for cleaning (£1,400)
- Explanation for management fee increase

💡 *Suggested next step:*
Request itemized breakdown and supporting invoices before paying.

Reply with questions or type @commonground to ask me anything.
```

---

## 7. RAG (Retrieval-Augmented Generation) design

### 7.1 Query flow

```
User asks: "Did they promise to fix the roof last year?"
↓
1. Embed query with OpenAI text-embedding-3-small
2. pgvector similarity search:
   SELECT content, metadata
   FROM document_chunks
   WHERE building_id = $1
   ORDER BY embedding <=> $query_embedding
   LIMIT 10
3. Build context: top 10 chunks + current case claims
4. Claude Sonnet: answer with citations
5. Response includes [Source: Letter from Buildium, March 2025]
```

### 7.2 Chunking strategy

- Chunk size: 800 tokens
- Overlap: 100 tokens
- Metadata stored per chunk: `doc_type`, `doc_date`, `case_id`, `page_number`

---

## 8. AI pipeline design

### 8.1 System prompt (base)

```
You are Common Ground, the AI case manager for this leasehold building.

Your role:
- Help residents understand complex property management communications
- Extract and structure evidence from documents and resident messages
- Identify contradictions between managing agent claims and resident experience
- Draft professional, calm, evidence-based formal responses

Rules:
- Always separate facts (sourced) from inference (flagged as "Based on available documents...")
- Never make definitive legal statements. Use: "You may want to seek legal advice about..."
- Never share flat-specific personal details in group context without resident consent
- Never send formal communications autonomously — always require director approval
- Be calm, precise, and non-inflammatory in all outputs
- If uncertain, say so explicitly

Building context: {building_name}, {address}
Current date: {date}
```

### 8.2 Key prompts

**Document classification + extraction:**
```
Analyse this document from a leasehold building managing agent.
1. Classify the document type (invoice / service_charge_demand / repair_update / estimate / complaint_reply / contractor_update / consultation_notice / correspondence / other)
2. Extract: all dates, all monetary amounts with labels, named parties, flat numbers mentioned, deadlines, promised actions
3. Identify red flags: unexplained increases, missing supporting documents, vague language, contradictions
4. List missing items that residents should request
Return as JSON.
```

**Claim contradiction detection:**
```
Compare the following managing agent claim with the resident evidence provided.
Managing agent claim: {claim}
Resident evidence: {evidence_items}
Historical records: {rag_context}

Classify: confirmed / disputed / uncertain / insufficient_evidence
Explain the contradiction if disputed.
List what additional proof would resolve this.
```

**Draft generation:**
```
Draft a formal response to the managing agent on behalf of the residents of {building_name}.

Context:
- Issue: {case_description}
- Claims disputed: {disputed_claims}
- Evidence available: {evidence_summary}
- Missing documents requested: {missing_docs}
- Tone requested: {tone} (neutral / firm / urgent)

Requirements:
- Professional, calm, factual
- Each paragraph should cite its source
- Request specific documents by name
- Set a reasonable response deadline
- Do not threaten legal action (leave that to the director's judgment)
- End with a clear ask

Output: the letter text only, ready to copy-paste.
```

---

## 9. Director web dashboard

### 9.1 Routes

```
/                          → redirect to /dashboard or /auth/login
/auth/login                → magic link email input
/auth/callback             → Supabase auth callback
/onboarding                → new building setup form
/dashboard                 → case list + building overview
/dashboard/cases/[id]      → case detail view
/dashboard/library         → document library + search
/dashboard/drafts          → pending draft approvals
/dashboard/settings        → building config, residents, billing
```

### 9.2 Case detail view

Sections:
1. **Status bar** — current status + days in status
2. **Timeline** — chronological events (documents received, claims made, evidence submitted, drafts proposed, approvals)
3. **Claims table** — claim / source / status (confirmed/disputed/uncertain) / evidence
4. **Evidence panel** — private evidence visible only to director; resident-shared evidence visible to building
5. **Open questions** — unanswered items with days elapsed
6. **Drafts** — proposed drafts with tone indicator; approve / reject / edit
7. **Copy-ready output** — approved draft in a copy-paste box

### 9.3 Building library

- Upload historical documents (bulk PDF upload)
- Search bar → natural language query → RAG result with source highlights
- Filter by doc_type, date range, case

### 9.4 Onboarding form (first run)

```
Step 1: Your building
  - Building name (e.g. "32 Elm Park Gardens")
  - Address
  - Number of flats

Step 2: Connect WhatsApp
  - Instructions to add Common Ground's number (+44 xxxx) to your WhatsApp group
  - Verify connection button (bot sends test message to group)

Step 3: Upload historical documents (optional)
  - Drag-and-drop area for old service charge summaries, invoices, correspondence
  - Progress indicator for indexing

Step 4: Subscribe
  - Stripe checkout (£X/month per building)
```

---

## 10. Proactive reminders (pg_cron)

```sql
-- Runs daily at 09:00 UTC
SELECT cron.schedule(
  'daily-reminder-check',
  '0 9 * * *',
  $$
  SELECT net.http_post(
    url := 'https://[project].supabase.co/functions/v1/daily-reminders',
    headers := '{"Authorization": "Bearer [SERVICE_ROLE_KEY]"}'
  )
  $$
);
```

**Edge Function: daily-reminders**
```
For each building with active cases:
  - Check open_questions where days_since_asked >= 7, 14, 21
  - Check cases where status = 'waiting_agent_reply' and last_update > 14 days
  - Check draft_actions where status = 'proposed' and created_at > 3 days
  → Post reminder to WhatsApp group if threshold met
```

Example reminder:
```
⏰ *Case update — Service Charge 2026*

Still waiting (14 days):
• Invoice backup for cleaning costs
• Contractor invoices (£1,400)

No reply from managing agent since 10 March.

Would you like to send a follow-up? Type @commonground "draft follow-up"
```

---

## 11. Privacy and permissions

### 11.1 Evidence visibility model

| Visibility | Who sees it |
|-----------|-------------|
| `private` | Bot + director only (default for flat-specific evidence) |
| `anonymous` | Bot + director + building (no resident name shown) |
| `named` | Bot + director + building (resident name shown) |

When a resident shares evidence from 1:1 to group:
```
Bot (1:1): "Your photo has been added to the case.
            Share with the group? Options:
            1. Yes, with my name (Flat 7)
            2. Yes, anonymously
            3. No, keep private"
```

### 11.2 What the bot can post to the group

✅ Can post: document summaries, case status updates, evidence prompts, draft proposals, reminders, answers to @mention questions

❌ Cannot post: resident names + their evidence without consent, flat-specific financial details, anything marked private

---

## 12. Payments (Stripe)

- Product: **Common Ground Building Plan**
- Price: £X/month per building (TBD)
- Integration: Stripe Checkout + webhook for subscription status
- `buildings.subscription_status` checked on each director dashboard load
- Expired subscription → read-only mode (can view but not create new cases/drafts)

---

## 13. Deployment

```
Vercel:
  - Next.js app (production + preview branches)
  - Environment vars: SUPABASE_URL, SUPABASE_ANON_KEY, SUPABASE_SERVICE_ROLE_KEY,
    OPENAI_API_KEY, ANTHROPIC_API_KEY, META_WHATSAPP_TOKEN, META_VERIFY_TOKEN, STRIPE_*

Supabase:
  - Project: commonground-prod
  - Edge Functions: process_event, daily_reminders
  - Storage buckets: buildings/{building_id}/documents, buildings/{building_id}/evidence
  - pg_cron: daily reminder job
```

---

## 14. V2 roadmap

| Feature | Notes |
|---------|-------|
| Gmail / Outlook integration | Auto-ingest managing agent emails |
| Facebook Messenger groups | Channel adapter already stubbed |
| Case pack export (PDF) | For solicitors / advisers |
| Committee voting flows | Multiple approvers |
| Complaint workflow templates | FTT / TPO / ombudsman |
| Email send via Gmail | After director approves draft |
| Major works / S20 workflow | Specific consultation notice handling |

---

## 15. Open technical risks

| Risk | Mitigation |
|------|-----------|
| WhatsApp Cloud API group limitations | Test group webhook early; confirm group message receipt works with Meta Cloud API |
| Facebook Groups API restrictions | V2; may require web-only approach for FB buildings |
| Vercel timeout on heavy PDF processing | Routed to Supabase Edge Functions |
| RAG quality on short/poor documents | Hybrid: keyword search fallback + recency boost |
| Resident doesn't self-register flat | Bot gently re-asks; director can map in dashboard |
| Stripe webhook failures | Idempotency keys + retry logic |
