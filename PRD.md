# PRD: Leasehold Case Copilot

**Document status:** Draft v2 (updated 2026-03-25 — open questions resolved)
**Working title:** Leasehold Case Copilot

---

## One-line summary

A shared AI case manager for leaseholders and resident directors that lives in the building chat, reads management-company communications, gathers resident corrections and evidence, compares them with historic building documents, and drafts the next formal move once the right people approve it.

---

## 1. Background

In many leasehold buildings, communication with the managing agent is fragmented, slow, and difficult to verify. Critical information lives across email threads, WhatsApp chats, PDFs, invoices, service charge summaries, contractor notes, and the memory of one overburdened resident director.

Common failure modes:
- Residents do not understand what a letter, invoice, or charge actually means
- Managing agents provide incomplete, vague, or delayed responses
- People lose track of what was promised, what was disputed, and what is still unanswered
- Flat-specific facts are not systematically gathered from the people who actually experienced the work
- Older documents exist but are hard to find and even harder to compare with new claims
- Escalation happens too late, with weak evidence and poor written records

This creates a strong opportunity for an AI product that is not just a chatbot, but a **shared evidence and escalation manager** for a building.

---

## 2. Product vision

Create the default operating layer for leaseholders and resident directors dealing with managing agents, repairs, service charges, and building-level disputes.

The product should feel like:
- A very competent secretary
- A memory system for the building
- A translator of property-management language into plain English
- A structured evidence collector
- A careful next-step drafter
- A human-in-the-loop escalation copilot

---

## 3. Goals

### Primary goals

1. Help residents understand management-company communications quickly and accurately
2. Turn unstructured building chat into structured, attributable evidence
3. Preserve shared case memory across months of discussions and documents
4. Help the building challenge unclear, inflated, or unsupported charges and push for fair pricing for work that was actually carried out
5. Help residents identify, question, and reduce unnecessary costs they do not meaningfully benefit from
6. Help the building send better, faster, more evidence-based responses
7. Reduce delays caused by confusion, forgotten context, and missing documents
8. Support resident directors who are under-informed, overloaded, or inconsistent

### Secondary goals

1. Improve trust and transparency within the building
2. Create a reusable audit trail for complaints and escalation
3. Increase confidence in challenging unclear charges or vague repair updates
4. Build a foundation for later workflows around formal complaints, major works, and right-to-manage style transitions

---

## 4. Non-goals (for MVP)

1. Replace the building's resident portal or payment portal
2. Automatically send messages to the managing agent without explicit approval
3. Provide legal advice or definitive legal conclusions
4. Handle all of property management end-to-end
5. Manage contractors directly
6. Support every housing tenure model from day one
7. Act as a fully autonomous agent with no human review
8. Direct sending from the bot (outbound is copy-ready text only — director pastes and sends)
9. Payment workflows or contractor procurement
10. Native tribunal or ombudsman filing
11. Mobile app as a separate primary surface

---

## 5. Target users

### Primary users

**A. Resident Director / Volunteer Building Lead**
A resident who informally or formally coordinates communication with the managing agent, holds old documents, and is expected to know what is happening.

Pain points: forgets details; overwhelmed by emails and attachments; unsure what rights apply; struggles to synthesize resident input; cannot keep a clean timeline of disputes.

*First commercial buyer.* The director signs up, configures the building, and brings in residents.

**B. Active Leaseholder / Concerned Resident**
A resident who wants transparency, asks questions, disputes charges, or contributes facts about repairs and works.

Pain points: does not understand letters or invoices; feels ignored by the managing agent; has evidence but no structured way to contribute it; loses trust in both the director and the agent.

### Secondary users

**C. Building Committee / Core Resident Group**
A small group that discusses strategy, approves escalations, and aligns on next actions.

**D. Later-stage professional user (future)**
Solicitor, adviser, or specialist leasehold consultant brought in once the case becomes more formal.

---

## 6. Geography and market scope

**MVP scope:**
- Leasehold residential buildings in England
- Small and medium blocks with active resident coordination
- Cases focused on repairs, service charges, contractor claims, and slow/incomplete communication from managing agents

**Why this scope:** The workflows, documents, and escalation logic are specific enough to create a meaningful product wedge without trying to solve all of property operations.

---

## 7. Core jobs to be done

### Functional jobs

- When a letter or invoice arrives, help us understand what it says and what is missing
- When the managing agent makes a claim, help us test it against resident reality
- When residents remember facts or upload evidence, organize it into something usable
- When we need to respond, draft the strongest reasonable next message
- When the case drags on, keep a reliable timeline and remind us what has not been answered

### Emotional jobs

- Help residents feel less powerless
- Help the director feel less alone and less likely to make mistakes
- Reduce social tension in the building by replacing opinion wars with shared evidence

### Social jobs

- Allow the group to look organized, factual, and credible when communicating with the managing agent

---

## 8. Product principles

1. **Human approval before action.** The bot proposes; humans approve outbound formal communication.
2. **Evidence over opinion.** Every strong claim should be linked to a source.
3. **Shared memory by default.** The system should remember what the building forgets.
4. **Private when needed.** Flat-specific evidence is private by default — resident explicitly chooses to share with the group or contribute anonymously to the case.
5. **Plain English first.** Residents should not need leasehold expertise to understand what is happening.
6. **Structured contradiction handling.** The bot should surface disputed facts rather than pretending certainty.
7. **Role-based visibility.** Not every resident needs access to every flat-specific detail.
8. **Careful, not aggressive.** The bot should sound professional, calm, and precise.
9. **Safe by default in mixed groups.** If the managing agent has access to the building chat, the bot automatically restricts what it posts publicly. Strategy, evidence analysis, and draft responses stay in private channels.

---

## 9. Core product concept

The bot lives inside the building's shared chat, also supports private 1:1 resident conversations, and connects both interaction modes to a building document library and director web dashboard.

It performs six core functions:
1. **Read incoming communications** from the managing agent or contractor
2. **Explain and summarize** them in plain English for residents, either publicly in the building context or privately in a 1:1 context
3. **Collect corrections, facts, and evidence** from the relevant residents
4. **Support private resident understanding** of flat-specific invoices, charges, and issues that a resident may not want to discuss in the shared building chat
5. **Compare new claims with old documents and prior statements**
6. **Draft the next formal step** once the right people approve it

---

## 10. User roles and permissions

**Role 1: General resident**
Can: read building-wide summaries; ask questions in chat; submit facts and evidence relevant to their flat or shared areas; comment on draft responses.
Cannot: send formal communications; approve outbound messages; access restricted flat-specific content unless permitted.

**Role 2: Resident director / authorized representative**
Can: upload historical building documents; define who can approve outbound communication; approve or reject drafts; trigger the final message generation; manage permissions and visibility; access the web dashboard.
Cannot: act without review for formal outbound communications.

**Role 3: Committee approver (optional, V2)**
Can: vote or approve certain classes of outbound messages; access restricted case summaries.

**Role 4: Bot**
Can: read configured documents and messages; summarize, compare, ask follow-up questions, and draft; create structured case records; propose actions.
Cannot: send formal emails or letters without explicit approval; make legal determinations; reveal restricted evidence outside the allowed audience.

---

## 11. Core objects / data model

**Building** — the overall unit containing residents, roles, chat history, documents, and cases. Multi-tenant; each building is isolated.

**Case** — a structured issue such as: disputed service charge line item; unclear repair status; contractor cost discrepancy; major works communication problem; unanswered document request.

**Claim** — a statement made by the managing agent, contractor, resident, or historical document. Examples: "Works were completed in Flat 12 on 14 March." / "Residents of Flat 12 state that no such works took place."

**Evidence item** — a source attached to a claim: email; PDF; invoice; photo; message from resident; older budget or prior year account. Each evidence item carries: source, author, timestamp, visibility (private / anonymous / shared), and a vector embedding for RAG retrieval.

**Draft action** — a proposed next step: request clarification; request documents; follow up on delay; begin formal complaint; summarize contradictions.

**Approval** — a recorded decision on whether a draft action may proceed. Stores: approver identity, timestamp, and any edits made.

---

## 12. Key user flows

### Flow 1: Incoming letter → shared understanding

1. Director or resident forwards a new email or uploads a PDF to the WhatsApp group or 1:1 bot chat
2. Bot posts a concise building-chat summary:
   - what the letter says
   - what it asks for
   - deadlines
   - missing documents
   - possible red flags
   - suggested next step
3. Residents ask questions in the chat
4. Bot answers in plain English and links each answer to the relevant source

### Flow 2: Private resident question → individual clarification

1. A resident messages the bot 1:1 with their own invoice, charge breakdown, or flat-specific issue
2. The bot explains the document in plain English and highlights confusing items, unusual charges, missing backup, and questions the resident may want to ask
3. The resident can ask follow-up questions privately without exposing their issue to the whole building
4. If the resident chooses, the bot suggests which parts of the issue should be escalated into the shared building case
5. With the resident's consent, the bot converts the relevant part of the issue into a shared claim or evidence item for the building

**Privacy default:** evidence is private until the resident explicitly opts to share. When shared, the resident's identity can be shown (default) or kept anonymous (resident's choice).

### Flow 3: Managing agent claim → resident correction intake

1. Bot identifies a building-level or flat-specific claim from the managing agent
2. Bot posts targeted prompts in the chat: "Flat 12, can you confirm whether this work happened?" / "Do you have photos, dates, or prior emails?"
3. Residents submit corrections and evidence (privately to bot or directly in group)
4. Bot converts replies into structured claim/evidence cards
5. Bot shows: confirmed facts / disputed facts / missing evidence / suggested clarification to request

### Flow 4: Historic document comparison

1. Director uploads old accounts, invoices, prior correspondence, lease extracts, or consultation notices
2. Bot indexes the material (vector embeddings via Supabase pgvector) and creates a building memory layer
3. When new claims arrive, the bot cross-checks them against historical records
4. Bot flags contradictions, changes in cost, repeated issues, and missing references

### Flow 5: Drafting the next formal move

1. Bot proposes 2–3 possible next actions
2. Residents and director discuss in the WhatsApp group chat
3. Director gives final approval
4. Bot drafts a formal email/letter in a calm, factual style
5. Bot posts copy-ready text in the chat (and in the director's web dashboard)
6. Director copies, pastes into their email client, and sends manually

### Flow 6: Long-running case management

1. Bot maintains a live case timeline (visible in director's web dashboard)
2. It tracks unanswered questions, missing documents, and elapsed time
3. It periodically posts case status updates in the group chat:
   - "Still awaiting invoice backup — 14 days since our last request"
   - "Repair date not confirmed"
   - "No response to our last request after X days"
4. The group uses this timeline as the basis for escalation

---

## 13. Functional requirements

### A. Communication ingestion
- Ingest forwarded WhatsApp messages, pasted text, PDFs, and photo attachments via chat
- Support both shared building-level ingestion and private resident-level ingestion
- Detect document type where possible: invoice, service charge demand, estimate, complaint reply, contractor update, consultation notice
- Extract dates, amounts, named parties, flat numbers, promised actions, and deadlines

### B. Plain-English explanation
- Summarize documents in non-technical language
- Highlight unclear or incomplete elements
- Explain terminology and likely process implications
- Answer resident follow-up questions conversationally
- Allow a resident to keep that conversation private unless they choose to escalate part of it

### C. Contradiction and evidence engine
- Turn freeform resident messages into structured factual statements
- Attribute each statement to a source and timestamp
- Distinguish between: confirmed / disputed / uncertain / missing evidence
- Support flat-specific evidence intake with privacy controls
- Evidence private by default; resident opts in to share (named or anonymous)

### D. Historic building memory
- Index uploaded building documents via vector embeddings
- Support retrieval across prior years and cases
- Compare new charges/claims with older records
- Surface repeated issues and previously promised actions

### E. Drafting and approval
- Generate multiple response options (neutral / firm / urgent)
- Keep outbound communications factual and source-linked
- Require explicit director approval before producing final draft
- Output: copy-ready text in chat + web dashboard
- Record approval history

### F. Timeline and case tracking
- Maintain case-level timelines in director web dashboard
- Track open questions, pending documents, missed promises, next actions
- Case status labels: intake / waiting for resident evidence / waiting for managing agent reply / under review / complaint-ready
- Periodic proactive reminders to group chat

### G. Search and Q&A
- Allow director and residents to ask natural-language questions:
  - "Did they already promise to fix this last year?"
  - "Have we ever seen this contractor before?"
  - "Which flats disputed this charge?"
  - "What exactly are we still waiting for?"

---

## 14. AI behavior requirements

The assistant should:
- Be calm, precise, and non-inflammatory
- Clearly separate facts from inference
- Ask for missing evidence instead of over-claiming
- Cite the source of important conclusions
- Avoid making definitive legal judgments
- Recommend next steps in operational language rather than legal language
- Explicitly state uncertainty when documents are incomplete

The assistant should not:
- Hallucinate rights, deadlines, or obligations
- Present itself as a lawyer
- Publish flat-specific personal details into public building chat without permission
- Send or escalate anything automatically

---

## 15. UX surfaces

**Surface 1: Building chat (WhatsApp group)**
Primary collaborative workspace. Bot responds to: PDF/photo attachment, or direct @mention. Residents interact naturally; bot provides summaries, evidence prompts, draft text.
Two group modes: **Open** (no managing agent in group — full bot functionality) and **Closed** (agent present — bot posts only neutral summaries, all strategy goes to 1:1/dashboard).

**Surface 2: Private resident chat (1:1 with bot via WhatsApp)**
Individual resident Q&A; flat-specific document explanation; controlled sharing into building case.

**Surface 3: Director web dashboard**
Director-only access (MVP). Contains:
- Case list and status overview
- Per-case view: timeline, claims, evidence, unresolved questions, draft history
- Building library: searchable document archive
- Draft workspace: copy-ready outbound text with evidence annotations
- Approval history log

**Surface 4: Building library (within web dashboard)**
Searchable archive of historical documents and prior cases indexed for RAG retrieval.

**Surface 5: Draft workspace (within web dashboard)**
Review screen for formal outbound messages showing what evidence each paragraph relies on. One-click copy.

---

## 16. MVP scope

### Must-have
- WhatsApp Cloud API integration (bot in building group + 1:1 support)
- PDF and image ingestion via chat forwarding
- Plain-English summaries posted to group
- Resident evidence intake (private by default, explicit opt-in to share)
- Claim/evidence structuring with attribution
- Historical document upload and vector search (Supabase pgvector)
- Draft response generation (2–3 tone options)
- Director approval flow + copy-ready text output
- Case timeline and open-question tracking
- Director web dashboard (cases, library, drafts, approvals)
- Per-building subscription via Stripe Payment Link (~£20–50/month)
- Open/closed group mode (managing agent detection)
- GDPR minimal compliance kit (privacy notice, bot disclaimers, data minimization)
- Multi-tenant data isolation per building

### Nice-to-have (V2)
- Gmail / Outlook integration for automatic email ingestion
- Facebook Messenger group integration
- Voting flows for committee approval
- Suggested follow-up reminders / cron nudges
- Smart red-flag templates by document type
- Exportable case packs (PDF)
- Email send via Gmail integration

### Out of scope for MVP
- Direct sending from the bot
- Payment workflows
- Contractor procurement
- Full legal workflow automation
- Native tribunal or ombudsman filing
- Mobile app as a separate primary surface
- Non-England housing tenure models

---

## 17. Technical architecture (decided)

**Stack:** Next.js (App Router) + Supabase + Vercel

**Architecture pattern:** Next.js monolith
- API routes handle WhatsApp webhook (Twilio/Cloud API)
- Supabase: PostgreSQL (data), pgvector (embeddings), Storage (files), Auth (director login)
- Heavy AI jobs (PDF parsing, embedding generation, draft generation) routed to Supabase Edge Functions to avoid Vercel 30s timeout
- Webhook → API route → queue job → Edge Function → write result → bot responds

**Bot triggers in group:**
- PDF or image attachment detected
- Direct @mention of bot

**Multi-tenancy:** each building is a row in `buildings` table; all data (cases, evidence, documents, residents) scoped to `building_id`; row-level security via Supabase RLS

**AI models:** Claude Haiku (chat, classification) + Claude Sonnet (document analysis, drafts)

**Group mode:** open/closed configurable per building

---

## 18. Privacy, trust, and safety requirements

1. Role-based access control required (Supabase RLS)
2. Flat-specific evidence private by default; resident explicitly opts to share (named or anonymous)
3. The system maintains visible provenance for major claims
4. Users should know when AI is summarizing versus quoting source material
5. Sensitive personal data minimized in shared contexts
6. All outbound communication must be human-approved (director)
7. The product keeps a log of who approved what and when

---

## 19. Go-to-market hypothesis

**Initial wedge:** small and medium leasehold blocks where at least one resident is highly engaged; there is an overloaded or underpowered director; there are ongoing issues with service charges, repairs, or confusing managing-agent communication.

**Likely buyer/user pattern:**
- Resident director buys first, brings building in
- Core resident group adopts organically through the chat (no new app to install — WhatsApp they already use)
- Later expansion into advisers or resident-management workflows

**Positioning:** Not "an AI chatbot for leaseholders." Rather: *"Shared case management and evidence coordination for your block."*

---

## 20. Pricing (decided)

**Model:** Per building subscription
**Target:** ~£20–50/month per building
**Rationale:** Predictable revenue; easy to sell to director; aligns with value delivered per building unit

V2 tiers possible: storage limits, case limits, committee voting features.

---

## 21. Risks

1. Residents may disagree on strategy and use the chat to fight rather than clarify
2. AI may overstate confidence when documents are incomplete
3. Permission boundaries may be hard to manage in small communities
4. Directors may expect legal certainty the product should not provide
5. WhatsApp Cloud API group limitations may constrain interaction patterns
6. Facebook Groups API is significantly restricted — V2 Facebook integration may require a different approach (web-only for Facebook buildings)
7. Some buildings may not have enough engagement to sustain collaborative use

---

## 22. Open questions — RESOLVED

| # | Question | Decision |
|---|----------|----------|
| 1 | Chat-native only vs. web dashboard? | Web dashboard for director (MVP); residents via chat only |
| 2 | How should approvals work? | Group discussion → director final approval → copy-ready text |
| 3 | Flat-specific evidence: private or public by default? | **Private by default** — resident explicitly opts to share (named or anonymous) |
| 4 | Which document types get strongest red-flag detection first? | Service charge demands and repair update letters (highest VoC signal) |
| 5 | First commercial user: director, residents' association, or committee? | **Resident director** buys and onboards the building |
| 6 | Email export vs. copy-ready text? | **Copy-ready text in chat + web dashboard** for MVP; Gmail send in V2 |
| 7 | Managing agent in the group chat? | **Two modes**: open (full) and closed (safe). Director chooses at onboarding. |

---

## 23. Suggested MVP statement

Build an AI case manager for leasehold buildings that lives in the WhatsApp building chat, reads managing-agent communications, gathers resident corrections and evidence, compares them with historic building documents, and prepares the next approved formal response — with a director web dashboard for case management and evidence review.

---

## 24. Version 2 opportunities

- Gmail / Outlook inbox integration for automatic email ingestion
- Facebook Messenger group integration
- Complaint workflow templates
- Structured document requests
- Case pack export for advisers
- Stronger major works / service charge review flows
- Committee voting and governance tooling
- Insurer / contractor / solicitor collaboration layers

---

## 25. Appendix: example jobs the bot should handle well

- "Explain this service charge demand in plain English."
- "Which parts of this repair update are still unanswered?"
- "Ask Flat 12 whether those works actually happened."
- "Compare this year's invoice with last year's records."
- "Draft a calm but firm follow-up asking for evidence."
- "What are the missing documents before we can assess this charge?"
- "Summarize the current state of the case for the whole building."
- "Prepare a final draft for the director to approve."
