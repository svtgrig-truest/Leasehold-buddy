# Leasehold Buddy — Full Build Plan (Site + Scout + Publisher)

## Context

Leasehold Buddy (Common Ground) is an AI case management platform for UK leaseholders. All documentation is complete (PRD, TECH_SPEC, VoC, MARKETING_PLAN, IMPLEMENTATION_PLAN). No code written yet.

This plan covers three parallel workstreams:
1. **Site** — Next.js 15 + Supabase monolith (landing + blog + director dashboard + WhatsApp integration)
2. **Scout** — community monitoring agent (two variants: Python cron + OpenClaw)
3. **Publisher** — content generation agent (Python cron, publishes via GitHub PR)

All three build in parallel across separate Claude sessions.

---

## Workstream 1: Site

### Scope
Full monolith per TECH_SPEC: single-page landing, /blog for SEO articles, /dashboard for directors, WhatsApp Cloud API webhook, Supabase backend. Deployed on Vercel (subdomain for now, custom domain later).

### Tech stack (per TECH_SPEC)
- Next.js 15 (App Router), TypeScript
- Supabase (PostgreSQL, pgvector, Storage, Auth, Edge Functions, pg_cron)
- Vercel hosting
- shadcn/ui + Tailwind CSS (I propose design, you approve)
- WhatsApp Cloud API (Meta)
- Claude Haiku + Sonnet (Anthropic API)
- OpenAI text-embedding-3-small (embeddings)
- Stripe Payment Link (manual for pilot)

### Site structure

```
/ (landing page — single-page with sections)
├── Hero: "Your building's AI case manager"
├── Problem: 3-4 pain points from VoC
├── How it works: 3 steps (Add WhatsApp bot → Forward a letter → Get clarity)
├── Features: summarize, evidence, draft, timeline
├── Social proof: placeholder for testimonials
├── CTA: "Add Common Ground to your building's WhatsApp" → WhatsApp signup link
└── Footer: links, legal

/blog
├── /blog/[slug] — individual article pages (MDX or Supabase CMS)
├── Article listing with tags, search
└── SEO optimized: meta, OG, sitemap, structured data

/dashboard (auth-gated, magic link)
├── /dashboard — case list + status overview
├── /dashboard/cases/[id] — case timeline, claims, evidence, drafts
├── /dashboard/library — document archive with search
├── /dashboard/drafts — draft workspace + approve flow
└── /dashboard/settings — building config, residents
```

### Blog architecture decision
Articles stored as MDX files in `/content/blog/` in the repo. Publisher agent creates PR with new MDX file → Svetlana merges → Vercel auto-deploys. No CMS needed for v1.

### Database
12 tables per TECH_SPEC (buildings, profiles, residents, cases, documents, document_chunks, claims, evidence_items, draft_actions, approvals, chat_events, open_questions). Plus `blog_views` table for analytics.

### Implementation phases (Site)

#### Phase S0: Scaffold (2h)
- [ ] S0.1 `npx create-next-app@latest` with App Router, TypeScript, Tailwind
- [ ] S0.2 Add shadcn/ui, configure theme
- [ ] S0.3 Supabase project setup, env vars
- [ ] S0.4 Deploy to Vercel (empty shell)
- [ ] S0.5 Git structure: `/site/` in monorepo (or root if site IS the main project)

#### Phase S1: Landing page (3-4h)
- [ ] S1.1 Hero section with value prop + WhatsApp CTA button
- [ ] S1.2 Problem section (pain points from VoC)
- [ ] S1.3 How it works (3-step visual)
- [ ] S1.4 Features grid
- [ ] S1.5 Social proof placeholder
- [ ] S1.6 Footer with legal links
- [ ] S1.7 Mobile responsive
- [ ] S1.8 SEO: meta tags, OG images, robots.txt, sitemap.xml

#### Phase S2: Blog engine (3-4h)
- [ ] S2.1 MDX content pipeline: `/content/blog/*.mdx` → Next.js pages
- [ ] S2.2 Blog index page with listing, tags, pagination
- [ ] S2.3 Individual article page with TOC, reading time, share links
- [ ] S2.4 SEO: structured data (Article schema), canonical URLs
- [ ] S2.5 Seed 2-3 placeholder articles to test pipeline
- [ ] S2.6 RSS feed at /blog/rss.xml

#### Phase S3: Database + Auth (3-4h)
- [ ] S3.1 Supabase schema: all 12 tables from TECH_SPEC
- [ ] S3.2 Row-Level Security policies (building_id scoped)
- [ ] S3.3 Supabase Auth: magic link for directors
- [ ] S3.4 Auth middleware in Next.js (protect /dashboard routes)
- [ ] S3.5 Seed test building + test director

#### Phase S4: Director dashboard (8-10h)
- [ ] S4.1 Dashboard layout with sidebar navigation
- [ ] S4.2 Case list view (status, dates, priority)
- [ ] S4.3 Case detail view (timeline, claims, evidence, open questions)
- [ ] S4.4 Document library with search (Supabase pgvector)
- [ ] S4.5 Draft workspace: view drafts, approve/edit/reject
- [ ] S4.6 Settings: building config, manage residents
- [ ] S4.7 Real-time updates (Supabase Realtime)

#### Phase S5: WhatsApp integration (6-8h)
- [ ] S5.1 WhatsApp Cloud API setup (Meta Business, phone number)
- [ ] S5.2 Webhook handler: /api/webhook/whatsapp
- [ ] S5.3 Message processor: classify, save to chat_events, enqueue
- [ ] S5.4 Supabase Edge Function: process-document (PDF → text → chunks → embed)
- [ ] S5.5 Supabase Edge Function: process-message (@mention → RAG answer)
- [ ] S5.6 Supabase Edge Function: generate-draft (case context → 2-3 options)
- [ ] S5.7 Approval flow: director ✅ emoji → bot posts copy-ready text
- [ ] S5.8 Group mode: open/closed behavior based on who's in chat

#### Phase S6: Polish + launch prep (2-3h)
- [ ] S6.1 Error handling, loading states, edge cases
- [ ] S6.2 Privacy policy, terms of service pages
- [ ] S6.3 GDPR: cookie consent, data retention policy
- [ ] S6.4 Performance: Core Web Vitals, image optimization
- [ ] S6.5 Monitoring: Vercel Analytics, error tracking

---

## Workstream 2: Scout (already planned)

Detailed plan in `tasks/scout-implementation-plan.md`. Summary:

- **Phase 0:** Prerequisites (Telegram bot, Reddit API, SSH access)
- **Phase 1:** Python Cron Scout (PRAW + Claude API + Telegram + SQLite)
- **Phase 2:** OpenClaw Scout (SOUL.md + AGENTS.md + HEARTBEAT.md + skills)
- **Phase 3:** Comparison run

Key config:
- Reddit: u/Honest_Addendum_8517
- Channels: r/HousingUK (listen + draft), MSE (listen-only)
- Approval: Telegram bot
- No Leasehold Buddy links in v1

---

## Workstream 3: Publisher

### Scope
Python cron agent that reads pain packets from Scout and generates SEO articles + FAQ pages. Publishes via GitHub PR to the monorepo's `/content/blog/` directory. Svetlana reviews and merges.

### Architecture

```
/agents/publisher/
├── publisher.py          # main entry point
├── pain_reader.py        # reads pain packets from Scout's SQLite / JSON
├── keyword_planner.py    # clusters pains into keyword targets
├── article_generator.py  # Claude API: pain packet → full MDX article
├── faq_generator.py      # Claude API: recurring questions → FAQ page
├── github_pr.py          # creates PR with new content via gh CLI
├── state.py              # tracks published articles, avoids duplicates
├── config.py             # env vars, constants
├── prompts/
│   ├── article_system.md # system prompt for article generation
│   └── faq_system.md     # system prompt for FAQ generation
├── requirements.txt
├── .env.example
└── README.md
```

### How it works

1. `publisher.py cluster` — reads all pain packets, clusters by keyword/topic, ranks by frequency + search volume
2. `publisher.py generate --topic "service charge dispute"` — picks top unwritten topic, generates full MDX article
3. `publisher.py pr` — creates GitHub PR with new article(s), assigns Svetlana as reviewer
4. Svetlana reviews MDX in PR diff → merges → Vercel auto-deploys

### Article generation prompt strategy
- Input: pain packet cluster (3+ related pains with user phrases)
- Claude Sonnet generates: title, meta description, H2/H3 structure, 1500-2000 word article
- Tone: plain English, practical, action-oriented (per MARKETING_PLAN)
- Must cite official sources (GOV.UK, LEASE, legislation)
- Must include "What to do next" section
- No Leasehold Buddy product pitch in article body
- Soft CTA in sidebar/footer only: "Need help with this? Common Ground can help your building manage this."

### Content types (v1)
1. **SEO articles** — targeting keywords from MARKETING_PLAN:
   - "how to dispute service charge UK"
   - "section 20 notice explained"
   - "management company not responding to repairs"
   - "first tier tribunal leasehold"
   - +6 more from pain packet analysis
2. **FAQ pages** — compiled from recurring questions across pain packets

### Implementation phases (Publisher)

#### Phase P0: Pain packet integration (1h)
- [ ] P0.1 Define shared pain packet storage format (JSON files in `/data/pain-packets/`)
- [ ] P0.2 Scout writes packets there, Publisher reads from there

#### Phase P1: Article generator (3-4h)
- [ ] P1.1 Keyword planner: cluster pain packets by topic, rank by frequency
- [ ] P1.2 Article system prompt with tone, structure, source requirements
- [ ] P1.3 Claude API integration: pain cluster → full MDX article
- [ ] P1.4 MDX frontmatter: title, description, date, tags, keywords, author
- [ ] P1.5 FAQ generator: recurring questions → structured FAQ MDX page
- [ ] P1.6 Output validation: check links, structure, word count

#### Phase P2: GitHub PR pipeline (2h)
- [ ] P2.1 Create branch: `content/YYYY-MM-DD-slug`
- [ ] P2.2 Write MDX to `/content/blog/slug.mdx`
- [ ] P2.3 Create PR via `gh pr create` with summary + pain packet sources
- [ ] P2.4 Assign Svetlana as reviewer
- [ ] P2.5 Track published articles in state to avoid duplicates

#### Phase P3: Manual seed run (1h)
- [ ] P3.1 Manually create 3 pain packets from VoC research
- [ ] P3.2 Run publisher on them, verify PR quality
- [ ] P3.3 Merge first articles to seed the blog

---

## Cross-Workstream Dependencies

```
Scout ──pain packets──→ Publisher ──PRs──→ Site/blog
                                            ↑
                                     MDX files in /content/blog/
```

- Scout can start immediately (no site dependency)
- Publisher needs: (a) pain packets from Scout, (b) blog engine on site to be built
- Site blog engine (Phase S2) should be ready before Publisher creates its first PR
- All three CAN build in parallel if we seed Publisher with manual pain packets initially

## Parallel Build Strategy

| Session | Workstream | Dependency |
|---------|-----------|------------|
| Session A | Site: S0 → S1 → S2 → S3 → S4 → S5 → S6 | None |
| Session B | Scout: Phase 0 → 1 → 2 → 3 | None |
| Session C | Publisher: P0 → P1 → P2 → P3 | Needs S2 done for real PRs, but can dev independently |

## Monorepo Structure

```
Leasehold-buddy/
├── site/                       # Next.js 15 app (or root-level)
│   ├── app/                    # App Router pages
│   ├── components/             # React components
│   ├── lib/                    # utilities, Supabase client
│   ├── content/blog/           # MDX articles (Publisher writes here)
│   └── public/                 # static assets
├── agents/
│   ├── scout-cron/             # Python cron Scout
│   ├── scout-openclaw/         # OpenClaw config mirror (version control)
│   └── publisher/              # Python Publisher agent
├── data/
│   └── pain-packets/           # shared pain packet JSON files
├── supabase/
│   ├── migrations/             # SQL migrations
│   └── functions/              # Edge Functions
├── tasks/                      # plans, lessons
├── PRD.md
├── TECH_SPEC.md
├── MARKETING_PLAN.md
└── IMPLEMENTATION_PLAN.md
```

## Verification

### Site
1. Landing page loads, mobile responsive, Lighthouse score > 90
2. Blog renders MDX articles with proper SEO metadata
3. Dashboard auth flow works (magic link → protected routes)
4. WhatsApp webhook receives and processes test message
5. Document upload → chunk → embed → searchable in dashboard

### Scout
(per existing scout-implementation-plan.md)

### Publisher
1. `python3 publisher.py generate --topic "service charge"` produces valid MDX
2. `python3 publisher.py pr` creates proper GitHub PR
3. Merged PR triggers Vercel deploy, article visible on /blog

## Unresolved Questions

1. **Site root vs /site/**: should Next.js app be at repo root or in /site/ subdirectory? Root is simpler for Vercel, /site/ is cleaner for monorepo. Recommend: root-level Next.js, agents in /agents/.
2. **Blog MDX vs Supabase CMS**: plan says MDX files. If later you want non-developer editors, we'd migrate to Supabase CMS. For now MDX + Publisher PRs is cleanest.
3. **WhatsApp Business API approval**: Meta requires business verification for WhatsApp Cloud API. This takes 1-2 weeks. Start the application early.
4. **First 3 SEO articles**: should we write them manually during site build, or wait for Publisher to generate them? Recommend: manually seed from VoC data to launch blog faster.
5. **MSE scraping legality**: still needs robots.txt + ToS check before Scout listener goes live.
