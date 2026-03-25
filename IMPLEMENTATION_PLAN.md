# Common Ground — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an AI case manager for leasehold buildings that lives in WhatsApp, processes documents, detects contradictions, drafts responses, and provides a director web dashboard.

**Architecture:** Next.js 15 monolith + Supabase (Postgres, pgvector, Storage, Auth, Edge Functions, pg_cron) + Vercel. WhatsApp webhook → ACK 200 → Supabase Edge Functions do heavy AI work async. Multi-tenant by building_id with RLS.

**Tech Stack:** Next.js 15 App Router, Supabase, Vercel, shadcn/ui + Tailwind, Claude Haiku + Sonnet (Anthropic), OpenAI embeddings, Meta WhatsApp Cloud API, Stripe Payment Link, Vitest + Playwright

**Project root:** `/Users/svtgr/Desktop/Leasehold buddy/`
**Repo:** `https://github.com/svtgrig-truest/Leasehold-buddy`
**Specs:** `PRD.md`, `VoC.md`, `TECH_SPEC.md` (in root, will move to `docs/`)

---

## Context

The Leasehold-buddy repo currently has 3 docs commits on main. This plan builds the full MVP from scratch. The spec is finalized (PRD v2, TECH_SPEC v1). All architectural decisions resolved.

**Review fixes applied (v2):**
- Added WhatsApp Group API spike as Task 0.2 (de-risk before any infrastructure)
- Split Edge Functions: `process-document`, `process-message`, `generate-draft` (avoids 60s timeout)
- Fixed `days_since_asked` — removed GENERATED STORED, compute at query time
- Added `processing_status` + `processing_error` columns to `documents` table
- Added 1:1 private document flow (PRD Flow 2) as Task 6.3
- Added targeted evidence prompts to Task 7.1
- Moved Edge Function skeleton to Task 3.3 (was dangling dependency)
- PDF extraction stays in Node.js API route (pdf-parse works in Node), Edge Function gets extracted text
- Added WhatsApp webhook signature verification to Task 3.1
- Added `whatsapp_phone_number_id` population to Task 2.2
- Split Task 9.2 into 9.2a/9.2b and Task 5.5 into 5.5a/5.5b
- Removed standalone Task 11.3 — unit tests written alongside each task
- Added Task 11.3 for integration/E2E tests only
- Contradiction detection is manual trigger (director action) for MVP
- Added error handling notes to AI pipeline tasks
- Updated estimates to ~88h sequential, ~50h wall-clock with 2 parallel streams

**Parallel execution (v3):**
- Stream A (Dashboard): Auth → Onboarding → Dashboard pages → Privacy (~26h)
- Stream B (Backend): WhatsApp → Registration → Docs → RAG → Cases → Drafts → Bot → Production (~42h)
- Streams A and B run in parallel after Phase 1 (shared foundation)
- Stream B owns Edge Functions, WhatsApp, AI pipeline
- Stream A owns web pages, components, Stripe
- Merge point: Phase 10 (Bot Intelligence) needs dashboard settings (group mode toggle from 9.5)
- Wall-clock estimate: **~50h** with 2 subagents

---

## Dependency Graph — 2 Parallel Streams

```
              Phase 0: Scaffold + WA Spike (sequential, both streams need this)
                              │
              Phase 1: Foundation — DB + RLS + Clients + UI (sequential)
                              │
           ┌──────────────────┴──────────────────┐
           │                                     │
    STREAM A (Dashboard)               STREAM B (Backend + AI)
           │                                     │
    Phase 2: Auth + Onboarding          Phase 3: WhatsApp Core
           │                                     │
    Phase 9.1: Dashboard home           Phase 4: Registration
    Phase 9.2a: Case detail pt1                  │
    Phase 9.2b: Case detail pt2        Phase 5: Document Pipeline
    Phase 9.3: Library                           │
    Phase 9.4: Drafts workspace         Phase 6: RAG + Q&A
    Phase 9.5: Settings                          │
    Phase 9.6: Privacy page             Phase 7: Case Management
    Phase 2.3: Stripe webhook                    │
           │                            Phase 8: Drafts + Approval
           │                                     │
           └──────────────────┬──────────────────┘
                              │
                    Phase 10: Bot Intelligence
                              │
                    Phase 11: Production + Tests
```

**Merge point:** Phase 10 needs both streams complete (group mode toggle from 9.5 + all bot flows from B).
**Wall-clock:** Phase 0+1 (~9h seq) + max(Stream A ~26h, Stream B ~42h) + Phase 10+11 (~16h) = **~50h**

---

## Phase 0: Scaffold + De-risk (2 tasks)

### Task 0.1: Init Next.js + tooling + move docs

**Files:**
- Move: `PRD.md`, `VoC.md`, `TECH_SPEC.md` → `docs/`
- Create: `package.json`, `next.config.ts`, `tailwind.config.ts`, `tsconfig.json`
- Create: `vitest.config.ts`, `playwright.config.ts`
- Create: `.env.local.example` (all env vars, no values)
- Create: `src/app/layout.tsx`, `src/app/page.tsx`
- Create: `supabase/config.toml` via `supabase init`

- [ ] Move docs to `docs/` folder
- [ ] `npx create-next-app@latest . --ts --tailwind --app --src-dir --eslint`
- [ ] Install dev deps: `vitest @testing-library/react playwright`
- [ ] Create `vitest.config.ts` + `playwright.config.ts`
- [ ] Create `.env.local.example`
- [ ] Run `supabase init`
- [ ] Verify: `pnpm dev` starts, `pnpm test` runs cleanly
- [ ] Commit + push

### Task 0.2: 🔴 WhatsApp Group API spike

**CRITICAL: Validate before building anything else.** The entire product depends on Meta Cloud API delivering group messages with `group_id` in webhook payload.

- [ ] Create Meta Business App + WhatsApp test number (or use existing)
- [ ] Set up temporary webhook (e.g. ngrok + minimal Express handler)
- [ ] Add test number to a WhatsApp group
- [ ] Send messages in the group, inspect webhook payloads
- [ ] Confirm: group messages arrive with a usable group/conversation ID
- [ ] Document payload structure in `docs/whatsapp-group-spike.md`
- [ ] If fails: STOP and redesign architecture (fallback: 1:1 only + web group features)

---

## Phase 1: Foundation (4 tasks)

### Task 1.1: shadcn/ui + dashboard shell

**Files:**
- Create: `components.json`, `src/lib/utils.ts`
- Create: `src/components/ui/` — button, input, card, badge, dialog, dropdown-menu, table, textarea, tabs, toast
- Create: `src/components/layout/sidebar.tsx`, `header.tsx`, `dashboard-shell.tsx`

- [ ] Run `npx shadcn@latest init`
- [ ] Add components: `npx shadcn@latest add button input card badge dialog dropdown-menu table textarea tabs toast`
- [ ] Build `dashboard-shell.tsx` wrapping sidebar + header + `{children}`
- [ ] Verify: shell renders at localhost:3000 with empty content
- [ ] Commit + push

### Task 1.2: Database migrations — 12 tables + pgvector

**Files:**
- Create: `supabase/migrations/00001_create_buildings.sql` through `00012_create_open_questions.sql`

Schema from TECH_SPEC section 4.2 **with these fixes:**
- `documents`: add `processing_status text DEFAULT 'pending' CHECK (processing_status IN ('pending','processing','completed','failed'))` and `processing_error text`
- `open_questions`: **remove** `days_since_asked GENERATED ALWAYS AS` column (broken — `now()` evaluates at write time). Compute at query time instead.
- `00006` includes `CREATE EXTENSION IF NOT EXISTS vector;` + ivfflat index

- [ ] Write all 12 migration files
- [ ] Run `supabase db reset` locally
- [ ] Verify: all 12 tables, `document_chunks.embedding` is `vector(1536)`, `documents.processing_status` exists
- [ ] Commit + push

### Task 1.3: RLS policies

**Files:**
- Create: `supabase/migrations/00013_enable_rls.sql`
- Create: `supabase/migrations/00014_rls_policies.sql`

Pattern: `USING (building_id IN (SELECT building_id FROM profiles WHERE id = auth.uid()))`
Special: `profiles` — own row only. Service role bypasses all.

- [ ] Write RLS enable + policy migrations
- [ ] Verify: test user can only access own building's data
- [ ] Commit + push

### Task 1.4: Supabase client helpers

**Files:**
- Create: `src/lib/supabase/client.ts` — `createBrowserClient()`
- Create: `src/lib/supabase/server.ts` — `createServerClient()` (cookies, RSC)
- Create: `src/lib/supabase/admin.ts` — `createAdminClient()` (service_role)
- Create: `src/lib/supabase/types.ts` — generated via `supabase gen types typescript`
- Create: `src/lib/supabase/middleware.ts` — session refresh helper

- [ ] Write three client wrappers
- [ ] Generate types: `supabase gen types typescript --local > src/lib/supabase/types.ts`
- [ ] Verify: TypeScript compiles, types match schema
- [ ] Commit + push

---

## Phase 2: Auth + Onboarding (3 tasks)

### Task 2.1: Magic link auth

**Files:**
- Create: `src/app/auth/login/page.tsx` — email input → `supabase.auth.signInWithOtp()`
- Create: `src/app/auth/callback/route.ts` — exchange code for session
- Create: `src/middleware.ts` — protect `/dashboard/*`, `/onboarding/*`

- [ ] Build login page + callback route
- [ ] Write auth middleware
- [ ] Verify: magic link flow works (Supabase Inbucket locally)
- [ ] Commit + push

### Task 2.2: Onboarding wizard (4 steps)

**Files:**
- Create: `src/app/onboarding/page.tsx` — multi-step container
- Create: `src/components/onboarding/step-building.tsx` — name, address, flats
- Create: `src/components/onboarding/step-whatsapp.tsx` — instructions + verify + **populates `buildings.whatsapp_phone_number_id`** from config
- Create: `src/components/onboarding/step-documents.tsx` — drag-and-drop upload (wired in Phase 5)
- Create: `src/components/onboarding/step-subscribe.tsx` — Stripe Payment Link button

On complete: creates `buildings` row (incl. `whatsapp_phone_number_id`) + `profiles` row with role='director'.

- [ ] Build 4-step onboarding form
- [ ] Verify: complete flow creates DB rows with `whatsapp_phone_number_id`, redirects to dashboard
- [ ] Commit + push

### Task 2.3: Stripe Payment Link webhook

**Files:**
- Create: `src/app/api/stripe/webhook/route.ts` — handle `checkout.session.completed`, `customer.subscription.updated/deleted`
- Create: `src/lib/stripe.ts` — client init + webhook signature verification

- [ ] Write webhook handler + Stripe client
- [ ] Verify: `stripe trigger checkout.session.completed` → `buildings.subscription_status` updates
- [ ] Commit + push

---

## Phase 3: WhatsApp Core (3 tasks)

### Task 3.1: Webhook — verification + message receipt + signature check

**Files:**
- Create: `src/app/api/webhook/whatsapp/route.ts` — GET (verify_token challenge) + POST (verify `X-Hub-Signature-256`, parse, save, ACK 200)
- Create: `src/lib/whatsapp/webhook-parser.ts` — extract message_id, sender_phone, group_id, message_type, content/media_id
- Create: `src/lib/whatsapp/types.ts` — Meta webhook payload TypeScript types
- Create: `src/lib/whatsapp/signature.ts` — HMAC-SHA256 webhook signature verification

- [ ] Build GET + POST webhook handlers with signature verification
- [ ] Write webhook parser with Meta payload types
- [ ] Write unit test: `tests/unit/whatsapp/webhook-parser.test.ts`
- [ ] Verify: GET returns challenge, POST with valid signature returns 200, invalid signature returns 403
- [ ] Commit + push

### Task 3.2: WhatsApp client + message splitting

**Files:**
- Create: `src/lib/whatsapp/client.ts` — `sendTextMessage()`, `sendReaction()`, `downloadMedia()`
- Create: `src/lib/whatsapp/message-sender.ts` — `sendSplitMessage()`: split at 4096 chars, paragraph boundaries, max 3 messages
- Create: `src/lib/whatsapp/constants.ts` — MAX_MESSAGE_LENGTH, MAX_SPLIT_MESSAGES

- [ ] Build WhatsApp API client
- [ ] Build message splitter
- [ ] Write unit test: `tests/unit/whatsapp/message-sender.test.ts`
- [ ] Commit + push

### Task 3.3: Message processor — classify + save + enqueue

**Files:**
- Create: `src/lib/processing/message-processor.ts` — `processIncomingMessage()`:
  1. Identify building: group_id → buildings OR sender_phone → residents
  2. Save to `chat_events`
  3. Classify: document_upload / at_mention / group_text / private_text / reaction
  4. For heavy work: fire-and-forget call to Supabase Edge Function (**do NOT await response** — avoids blocking webhook)
  5. For reactions: handle inline (approval check)
- Create: `supabase/functions/process-document/index.ts` — **skeleton** Edge Function (dispatch stub)
- Create: `supabase/functions/process-message/index.ts` — **skeleton** Edge Function (dispatch stub)
- Create: `supabase/functions/_shared/supabase.ts` — shared admin client for Edge Functions
- Modify: `src/app/api/webhook/whatsapp/route.ts` — call processIncomingMessage

**Edge Function split:**
- `process-document` — download, extract text, classify, chunk, embed, save, summarize, post
- `process-message` — Q&A, case routing, evidence intake, registration follow-ups
- `generate-draft` (created in Phase 8) — context building + 3× Claude Sonnet calls

- [ ] Build message processor with classification
- [ ] Create Edge Function skeletons with shared Supabase client
- [ ] Wire into webhook POST handler (fire-and-forget invocation)
- [ ] Verify: group message → chat_events row with correct building_id, Edge Function invoked (check logs)
- [ ] Commit + push

---

## Phase 4: Resident Registration (1 task)

### Task 4.1: Self-registration via WhatsApp

**Files:**
- Create: `src/lib/processing/registration.ts` — check if phone exists, create residents row, send privacy notice link, ask flat number, parse response, confirm
- Create: `src/lib/whatsapp/templates.ts` — welcome, privacy_link, ask_flat, confirm_registration messages

- [ ] Build registration handler with flat number parsing (`/flat\s*\d+/i`)
- [ ] Write unit test: `tests/unit/processing/registration.test.ts`
- [ ] Verify: new phone → privacy notice → ask flat → "Flat 7" → updated + confirmed
- [ ] Commit + push

---

## Phase 5: Document Pipeline (6 tasks)

### Task 5.1: Media download + Storage upload

**Files:**
- Create: `src/lib/documents/media-downloader.ts` — download from WhatsApp → upload to Supabase Storage at `buildings/{building_id}/documents/{uuid}_{filename}`

Note: evidence files go to `buildings/{building_id}/evidence/` (handled in Task 10.2).

- [ ] Build downloader + uploader
- [ ] Verify: valid media_id → file in Supabase Storage at correct path
- [ ] Commit + push

### Task 5.2: PDF text extraction (Node.js API route)

**Files:**
- Create: `src/app/api/documents/extract/route.ts` — receives storage_path, downloads PDF from Supabase Storage, uses `pdf-parse` (Node.js — works reliably), returns extracted text. If text < 100 chars → `needs_vision=true`.
- Create: `src/lib/documents/pdf-parser.ts` — `extractTextFromPdf(buffer)` wrapper

**Architecture decision:** PDF extraction stays in Node.js API route (not Deno Edge Function) because `pdf-parse` doesn't work in Deno. The Edge Function calls this API route to get extracted text, then does the AI-heavy work.

- [ ] Build PDF extraction API route + parser module
- [ ] Write test: text PDF → extracted text; scanned PDF → needs_vision flag
- [ ] Commit + push

### Task 5.3: Photo/scan processing (Claude Vision)

**Files:**
- Create: `supabase/functions/process-document/photo-processor.ts` — download image, base64 → Claude Sonnet vision → extracted text
- Create: `supabase/functions/process-document/anthropic.ts` — Anthropic client for Deno runtime
- Create: `src/lib/ai/anthropic.ts` — Anthropic client for Node.js runtime

- [ ] Build photo processor with Claude Vision
- [ ] Build both Anthropic client wrappers
- [ ] Verify: photo of letter → text extracted correctly
- [ ] Commit + push

### Task 5.4: Document classification + metadata extraction

**Files:**
- Create: `supabase/functions/process-document/classify.ts` — Claude Sonnet → `{doc_type, dates, amounts, parties, flat_numbers, deadlines, red_flags, missing_items}`
- Create: `src/lib/ai/prompts.ts` — all AI prompt templates (shared Node.js + Deno)

- [ ] Build classifier with TECH_SPEC section 8.2 prompt
- [ ] Verify: service charge text → correct doc_type, amounts, red flags
- [ ] Commit + push

### Task 5.5a: Chunking + embedding modules

**Files:**
- Create: `supabase/functions/process-document/chunker.ts` — 800 tokens, 100 overlap
- Create: `supabase/functions/process-document/embedder.ts` — OpenAI text-embedding-3-small, with retry on failure (max 3 attempts, exponential backoff)

- [ ] Build chunker + embedder with error handling
- [ ] Write unit test: `tests/unit/documents/chunker.test.ts`
- [ ] Commit + push

### Task 5.5b: Document processing orchestrator + WhatsApp summary

**Files:**
- Create: `supabase/functions/process-document/orchestrator.ts` — full pipeline:
  1. Call Node.js API `/api/documents/extract` for PDF text (or photo-processor for images)
  2. Classify + extract metadata (5.4)
  3. Save `documents` row with `processing_status='processing'`
  4. Chunk text → embed chunks → save `document_chunks`
  5. Generate summary (Claude Sonnet) → post to WhatsApp (respecting group mode)
  6. Update `processing_status='completed'` (or `'failed'` + `processing_error` on error)
- Modify: `supabase/functions/process-document/index.ts` — wire event dispatch to orchestrator

**Error handling:** Wrap in try/catch. On failure: set `processing_status='failed'`, `processing_error=message`. Partial progress preserved (e.g., text extracted but embedding failed — document row exists with text).

- [ ] Build orchestrator with full error handling
- [ ] Verify: PDF in WhatsApp → documents + chunks rows + summary posted; failed API → status='failed'
- [ ] Commit + push

---

## Phase 6: RAG + Q&A (3 tasks)

### Task 6.1: Vector search

**Files:**
- Create: `supabase/migrations/00015_match_documents_function.sql` — PostgreSQL function for cosine similarity search
- Create: `supabase/migrations/00016_days_since_asked_view.sql` — view or SQL helper for `open_questions` with computed `days_since_asked` (replaces broken GENERATED column)
- Create: `src/lib/ai/rag.ts` — `searchDocuments()`: embed query → call match function → return ranked chunks
- Create: `src/lib/ai/embeddings.ts` — `generateEmbedding()` via OpenAI

- [ ] Write match_document_chunks SQL function
- [ ] Write days_since_asked view/helper
- [ ] Build RAG search module
- [ ] Verify: upload doc about "roof repair", query "fix the roof?" → relevant chunks, building-scoped
- [ ] Commit + push

### Task 6.2: @mention Q&A handler (group)

**Files:**
- Create: `supabase/functions/process-message/qa-handler.ts` — RAG search + active case claims → Claude Haiku → answer with citations
- Modify: `supabase/functions/process-message/index.ts` — wire at_mention events

- [ ] Build Q&A handler
- [ ] Verify: @mention "When did they promise to fix the roof?" → answer citing source document
- [ ] Commit + push

### Task 6.3: 1:1 private document flow (PRD Flow 2)

**Files:**
- Create: `supabase/functions/process-message/private-doc-handler.ts` — when resident sends document/photo in 1:1:
  1. Process document privately (don't post summary to group)
  2. Send private explanation to resident: plain-English summary, red flags, unusual charges, questions to ask
  3. Offer: "Would you like to add this to the building case? Reply YES to share."
  4. If YES → link document to case, optionally trigger evidence sharing flow (Task 10.2)
- Modify: `supabase/functions/process-message/index.ts` — route 1:1 document uploads to private handler

- [ ] Build private document handler
- [ ] Verify: PDF in 1:1 → private explanation (not posted to group) → "share?" → YES → linked to case
- [ ] Commit + push

---

## Phase 7: Case Management (3 tasks)

### Task 7.1: Case CRUD + claim/evidence structuring + targeted prompts

**Files:**
- Create: `supabase/functions/process-message/case-manager.ts` — `createCase()`, `linkDocumentToCase()`, `createClaim()`, `createEvidenceItem()`
- Create: `src/lib/ai/classify.ts` — Claude Haiku classifies message intent (evidence / question / chat) + relevant case_id
- Create: `supabase/functions/process-message/evidence-prompter.ts` — when flat-specific claim detected, send targeted prompt: "Flat 12, can you confirm whether this work happened?"

- [ ] Build case manager + intent classifier + evidence prompter
- [ ] Verify: doc upload → case created; flat-specific claim → targeted prompt sent to group
- [ ] Commit + push

### Task 7.2: Contradiction detection (manual trigger)

**Files:**
- Create: `src/lib/ai/contradiction.ts` — load claims → find evidence + RAG context → Claude Sonnet → classify confirmed/disputed/uncertain → update claims.status
- Create: `src/app/api/cases/[id]/analyze/route.ts` — API route, director triggers from dashboard

For MVP: manual trigger only (director clicks "Analyze contradictions" on case detail page). Automatic triggering deferred to V2.

**Error handling:** If Claude Sonnet call fails, return partial results for claims already processed + error for remaining.

- [ ] Build contradiction detector with TECH_SPEC prompt
- [ ] Build API route for manual trigger
- [ ] Verify: agent claim + conflicting resident evidence → claim marked 'disputed'
- [ ] Commit + push

### Task 7.3: Case routing in group chat

**Files:**
- Create: `supabase/functions/process-message/case-router.ts` — Claude Haiku: which case? If ambiguous + @mentioned → ask. If not @mentioned → ignore.
- Modify: `supabase/functions/process-message/index.ts` — wire group_text through router

- [ ] Build case router
- [ ] Verify: 1 active case → linked; 2 cases + no @mention → silent; 2 cases + @mention → asks
- [ ] Commit + push

---

## Phase 8: Drafts + Approval (2 tasks)

### Task 8.1: Draft generation (3 tones)

**Files:**
- Create: `src/lib/ai/drafts.ts` — build context → Claude Sonnet × 3 (neutral/firm/urgent) → save draft_actions
- Create: `supabase/functions/generate-draft/index.ts` — Edge Function: context building + 3× Claude Sonnet calls

**Error handling:** If one tone fails, save the tones that succeeded. Set `processing_error` on failures. Each draft is independent.

- [ ] Build draft generator with TECH_SPEC prompt
- [ ] Verify: 3 drafts with distinct tones, saved as 'proposed', posted to correct channel
- [ ] Commit + push

### Task 8.2: Emoji approval + rejection

**Files:**
- Create: `src/lib/processing/approval.ts` — handle ✅ reaction: verify draft_actions match + director phone → approve → post copy-ready text with AI disclaimer
- Modify: `src/lib/processing/message-processor.ts` — route reactions to approval handler

- [ ] Build approval handler
- [ ] Write unit test: `tests/unit/processing/approval.test.ts`
- [ ] Verify: director ✅ → approved + copy-ready text; non-director ✅ → ignored; director reply → rejected
- [ ] Commit + push

---

## Phase 9: Dashboard (7 tasks)

### Task 9.1: Dashboard home — case list

**Files:**
- Create: `src/app/dashboard/page.tsx` — fetch cases, render list
- Create: `src/app/dashboard/layout.tsx` — DashboardShell wrapper
- Create: `src/components/dashboard/case-list.tsx` — table with status badges
- Create: `src/components/dashboard/building-overview.tsx` — summary stats

- [ ] Build dashboard home with case list
- [ ] Verify: director sees only own building's cases
- [ ] Commit + push

### Task 9.2a: Case detail — page + timeline + claims

**Files:**
- Create: `src/app/dashboard/cases/[id]/page.tsx`
- Create: `src/components/dashboard/case-timeline.tsx` — chronological events
- Create: `src/components/dashboard/claims-table.tsx` — claim / source / status / evidence

- [ ] Build case page shell + timeline + claims
- [ ] Verify: renders with correct data
- [ ] Commit + push

### Task 9.2b: Case detail — evidence + questions + drafts + analyze button

**Files:**
- Create: `src/components/dashboard/evidence-panel.tsx` — items with visibility indicators
- Create: `src/components/dashboard/open-questions.tsx` — computed days elapsed (query-time)
- Create: `src/components/dashboard/draft-section.tsx` — draft cards + approve/reject/edit
- Add: "Analyze contradictions" button → calls `POST /api/cases/[id]/analyze`

- [ ] Build evidence, questions, drafts sections + contradiction trigger
- [ ] Verify: all sections render; analyze button triggers contradiction detection
- [ ] Commit + push

### Task 9.3: Document library

**Files:**
- Create: `src/app/dashboard/library/page.tsx`
- Create: `src/components/library/document-list.tsx`, `document-search.tsx`, `upload-zone.tsx`
- Create: `src/app/api/documents/search/route.ts` — RAG search API
- Create: `src/app/api/documents/upload/route.ts` — file upload → triggers Phase 5 `process-document` pipeline

Note: web upload reuses the same `process-document` Edge Function from Phase 5, just with a web upload trigger instead of WhatsApp media download.

- [ ] Build library with search + upload
- [ ] Verify: search returns relevant docs, upload triggers processing
- [ ] Commit + push

### Task 9.4: Draft workspace

**Files:**
- Create: `src/app/dashboard/drafts/page.tsx`
- Create: `src/components/dashboard/draft-workspace.tsx`, `draft-actions.tsx`, `copy-ready-output.tsx`

- [ ] Build draft review + approve/reject + copy button
- [ ] Verify: approve creates approval row, copy-ready text appears
- [ ] Commit + push

### Task 9.5: Settings

**Files:**
- Create: `src/app/dashboard/settings/page.tsx`
- Create: `src/components/dashboard/settings-building.tsx` (group mode toggle), `settings-residents.tsx` (list + remove), `settings-billing.tsx`

- [ ] Build settings with group mode toggle + resident management
- [ ] Verify: toggle group mode → DB updated; remove resident → soft delete + anonymize evidence
- [ ] Commit + push

### Task 9.6: Privacy page

**Files:**
- Create: `src/app/privacy/page.tsx` — public, no auth. GDPR content per TECH_SPEC section 16.

- [ ] Write privacy page
- [ ] Verify: accessible without login at /privacy
- [ ] Commit + push

---

## Phase 10: Bot Intelligence (3 tasks)

### Task 10.1: Group mode enforcement

**Files:**
- Create: `src/lib/whatsapp/group-mode.ts` — `shouldPostToGroup(buildingId, contentType)` → target channel
  - Open: everything to group
  - Closed: only neutral summaries + reminders to group; strategy/contradictions/drafts → director 1:1
- Modify: all WhatsApp posting callsites in Edge Functions

Content routing:
- `document_summary` → always group
- `evidence_request` → open: group; closed: 1:1
- `contradiction_alert` → open: group; closed: 1:1
- `draft_proposal` → open: group; closed: 1:1
- `reminder` → always group (neutral phrasing)
- `qa_answer` → where asked

- [ ] Build group mode gatekeeper
- [ ] Update all posting callsites
- [ ] Verify: closed mode → sensitive content goes to director 1:1 only
- [ ] Commit + push

### Task 10.2: Evidence visibility + sharing flow

**Files:**
- Create: `src/lib/processing/evidence-sharing.ts` — save as private → ask "Share? 1) Named 2) Anonymous 3) Private" → update visibility on reply
- Evidence files stored at `buildings/{building_id}/evidence/` (separate from documents)

- [ ] Build sharing flow
- [ ] Verify: 1:1 evidence → private → reply "2" → anonymous in group
- [ ] Commit + push

### Task 10.3: GDPR bot behavior

**Files:**
- Modify: `src/lib/whatsapp/templates.ts` — first-contact privacy message, AI disclaimer footer
- Modify: `src/lib/processing/registration.ts` — send privacy notice before flat question
- Create: `src/lib/processing/gdpr.ts` — `anonymizeResident()`: soft delete + null out submitted_by_phone

- [ ] Wire GDPR behaviors
- [ ] Verify: new resident → privacy notice; all drafts → AI disclaimer; removal → anonymized
- [ ] Commit + push

---

## Phase 11: Production (3 tasks)

### Task 11.1: Proactive reminders (pg_cron)

**Files:**
- Create: `supabase/functions/daily-reminders/index.ts` — check thresholds using query-time `days_since_asked` computation:
  - `open_questions`: `EXTRACT(DAY FROM now() - asked_at)::int >= 7` (7/14/21 days)
  - `cases`: `status = 'waiting_agent_reply'` and no update > 14 days
  - `draft_actions`: `status = 'proposed'` and `created_at > 3 days ago`
- Create: `supabase/migrations/00017_pg_cron_daily_reminders.sql` — schedule daily 09:00 UTC

- [ ] Build reminder Edge Function + cron migration
- [ ] Verify: 8-day-old question → reminder fires; no thresholds → no message
- [ ] Commit + push

### Task 11.2: Deployment config

**Files:**
- Update: `.env.local.example` with all vars
- Create: `vercel.json` — region lhr1, function timeouts

```
NEXT_PUBLIC_SUPABASE_URL, NEXT_PUBLIC_SUPABASE_ANON_KEY, SUPABASE_SERVICE_ROLE_KEY,
ANTHROPIC_API_KEY, OPENAI_API_KEY,
META_WHATSAPP_TOKEN, META_VERIFY_TOKEN, META_PHONE_NUMBER_ID, META_APP_SECRET,
STRIPE_SECRET_KEY, STRIPE_WEBHOOK_SECRET,
NEXT_PUBLIC_APP_URL
```

- [ ] Finalize env config
- [ ] Deploy to Vercel, set all env vars at project level
- [ ] Deploy Edge Functions: `supabase functions deploy process-document`, `process-message`, `generate-draft`, `daily-reminders`
- [ ] Verify: health check passes, webhook reachable, Edge Functions deployed
- [ ] Commit + push

### Task 11.3: Integration + E2E tests

Unit tests are written alongside each task (Tasks 3.1, 3.2, 4.1, 5.5a, 8.2 each include their unit test). This task adds integration and E2E tests that require the full system.

**Files:**
- Create: `tests/integration/document-pipeline.test.ts` — PDF upload → chunks + embeddings
- Create: `tests/integration/rag-search.test.ts` — query returns relevant chunks
- Create: `tests/e2e/auth-flow.spec.ts` (Playwright) — login → onboarding → dashboard
- Create: `tests/e2e/dashboard.spec.ts` (Playwright) — case list, case detail, library search

- [ ] Write integration tests
- [ ] Write Playwright E2E tests
- [ ] Verify: `pnpm test` passes, `pnpm test:e2e` passes
- [ ] Commit + push

---

## Verification (end-to-end)

1. **Director onboarding:** login → create building → connect WhatsApp → upload historical docs
2. **Resident registration:** new phone messages group → privacy notice → ask flat → registered
3. **Document flow:** forward PDF to group → summary posted with red flags + missing items
4. **Q&A (group):** @mention "Did they promise to fix the roof?" → answer with source citation
5. **Q&A (1:1):** resident sends PDF in 1:1 → private explanation → option to share with building
6. **Evidence:** resident sends photo in 1:1 → chooses to share anonymously → appears in case
7. **Targeted prompts:** flat-specific claim detected → "Flat 12, can you confirm?"
8. **Contradiction:** director clicks "Analyze" → agent claim vs resident evidence → dispute flagged
9. **Draft:** 3 tones generated → director ✅ → copy-ready text with AI disclaimer
10. **Dashboard:** case list → case detail → library search → draft approve → settings
11. **Reminders:** stale question → daily reminder in group
12. **Group mode:** switch to closed → sensitive content goes to director 1:1 only

---

## Estimates

### Sequential (total work)

| Phase | Tasks | ~Effort | Stream |
|-------|-------|---------|--------|
| 0: Scaffold + Spike | 2 | 4h | shared |
| 1: Foundation | 4 | 5h | shared |
| 2: Auth + Onboarding | 2 | 4h | A |
| 2.3: Stripe webhook | 1 | 1h | A |
| 9: Dashboard | 7 | 16h | A |
| 3: WhatsApp Core | 3 | 7h | B |
| 4: Registration | 1 | 2h | B |
| 5: Document Pipeline | 6 | 14h | B |
| 6: RAG + Q&A | 3 | 6h | B |
| 7: Case Management | 3 | 8h | B |
| 8: Drafts + Approval | 2 | 5h | B |
| 10: Bot Intelligence | 3 | 5h | shared |
| 11: Production | 3 | 11h | shared |
| **Total** | **40 tasks** | **~88h** | |

### Wall-clock with 2 parallel subagents

| Segment | Duration | Notes |
|---------|----------|-------|
| Phase 0 + 1 (sequential) | ~9h | Both streams need foundation |
| Stream A ‖ Stream B | ~42h | A finishes in ~21h, waits for B's ~42h |
| Phase 10 + 11 (sequential) | ~16h | After both streams merge |
| **Wall-clock total** | **~50h** | **43% faster than sequential** |

## Risks

| Risk | Mitigation |
|------|------------|
| WhatsApp group webhooks don't deliver group_id | **Task 0.2 spike validates before any infrastructure.** Fallback: 1:1 only + web group features |
| pdf-parse in Deno | **Resolved: PDF extraction stays in Node.js API route** (Task 5.2) |
| Edge Function timeout (60s) | **Split into 3 functions:** process-document, process-message, generate-draft |
| Anthropic/OpenAI API failures mid-pipeline | Error handling in each task; `processing_status` column tracks failures; partial results preserved |
| pgvector ivfflat with low data (<100 chunks) | Consider HNSW or exact search for early buildings; switch to ivfflat when data grows |
| Edge Function cold starts | Webhook returns 200 immediately; Edge Function called fire-and-forget; cold start doesn't affect webhook ACK |
