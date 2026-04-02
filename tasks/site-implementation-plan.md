# Common Ground — Site Implementation Plan (v1)

## Context

Common Ground (working name: Leasehold Buddy) needs a marketing site with blog to support Scout/Publisher agents and start collecting waitlist signups. Dashboard and WhatsApp integration come later.

## Scope: v1 = Landing + Blog + Waitlist

No dashboard, no WhatsApp integration, no auth in v1. Just:
1. Single-page landing with waitlist email capture
2. Blog engine (MDX files in repo)
3. 3 seed SEO articles from VoC research
4. Deploy on Vercel (subdomain)

## Design System

### Reference: jupid.com
- Modular section-based layout
- Serif headlines + sans-serif body
- Warm accent color on dark/light alternating sections
- Clean minimalism, high whitespace
- Staggered reveal animations

### Color Palette
| Token | Value | Usage |
|-------|-------|-------|
| `--bg-primary` | `#FFFFFF` | Main background |
| `--bg-secondary` | `#FAF7F2` | Alternating sections (warm off-white) |
| `--bg-dark` | `#1C2024` | Dark sections, footer |
| `--text-primary` | `#1C2024` | Headlines, body |
| `--text-secondary` | `#1C2024/60` | Muted text |
| `--text-on-dark` | `#FFFFFF` | Text on dark bg |
| `--accent` | `#E8B86D` | Warm amber — CTAs, highlights, hover states |
| `--accent-light` | `#E8B86D/15` | Accent bg tint |
| `--border` | `#1C2024/8` | Subtle borders |

### Typography
- **Headlines:** Serif font (Playfair Display or Lora via Google Fonts)
- **Body:** Sans-serif (Inter via shadcn default)
- **Hero headline:** 48-64px, serif, bold
- **Section headlines:** 36-40px, serif, semibold
- **Body:** 16-18px, sans-serif, regular
- **Small/labels:** 14px, sans-serif, medium

### Components (shadcn/ui + custom)
- Buttons: rounded-xl, h-12/h-14, amber primary / outlined secondary
- Cards: rounded-2xl, subtle shadow, warm bg
- Input: rounded-xl, border, h-12 (for waitlist email)
- Section spacing: py-20 lg:py-32

## Site Structure

```
/ (landing — single page)
├── Nav: Logo + Blog link + "Join waitlist" button
├── Hero
│   ├── Headline: "Your managing agent isn't answering. Your service charge just doubled. Sound familiar?"
│   ├── Subheadline: "Common Ground is an AI case manager that lives in your building's WhatsApp. It reads the letters, gathers the evidence, and drafts the response."
│   ├── CTA: Email input + "Join the waitlist" button
│   └── Social proof line: "Built for leaseholders and resident directors in England"
├── Problem section (dark bg)
│   ├── "Sound familiar?" — 4 pain cards:
│   │   1. "Letters you can't decode" — managing agent sends jargon-heavy letter
│   │   2. "Charges that don't add up" — service charge doubles, no breakdown
│   │   3. "Repairs that never happen" — promised 6 months ago, still waiting
│   │   4. "One person carries everything" — resident director drowning in admin
│   └── Source: VoC.md top pain points
├── How it works (3 steps, light bg)
│   ├── 1. "Forward a letter" — director sends PDF/photo to WhatsApp group
│   ├── 2. "Get clarity in seconds" — bot summarises in plain English, flags red flags
│   └── 3. "Draft the next move" — bot writes formal response, you approve and send
├── Features grid (dark bg)
│   ├── "Plain English summaries" — no jargon
│   ├── "Building memory" — remembers every document, promise, deadline
│   ├── "Evidence from every flat" — residents contribute corrections privately
│   ├── "Draft responses" — calm, factual, evidence-based
│   ├── "Case timeline" — track what's open, what's overdue
│   └── "Your WhatsApp, not another app" — lives where you already talk
├── Social proof (light bg)
│   └── Placeholder: "Launching soon. Built from 200+ real conversations on r/HousingUK, MoneySavingExpert, and the National Leasehold Campaign."
├── CTA section (amber accent bg)
│   ├── "Stop losing track. Start building your case."
│   ├── Email input + "Join the waitlist" button
│   └── "Free during beta. No credit card required."
└── Footer (dark bg)
    ├── Logo + tagline
    ├── Links: Blog, Privacy Policy, Terms
    ├── "Made for leaseholders in England"
    └── © 2026 Common Ground

/blog
├── Blog index: article cards with title, excerpt, date, tag
├── Tags: service-charges, repairs, rights, section-20, tribunal, rtm
└── Pagination (if > 10 articles)

/blog/[slug]
├── Article layout: title, date, reading time, tags
├── MDX content with TOC sidebar
├── "Was this helpful?" feedback (optional v2)
├── Related articles
└── CTA banner: "Common Ground helps your building manage exactly this. Join the waitlist."

/privacy — Privacy policy page
/terms — Terms of service page
```

## Tech Stack

| Layer | Choice | Why |
|-------|--------|-----|
| Framework | Next.js 15 (App Router) | Per TECH_SPEC, future-proof for dashboard |
| Styling | Tailwind CSS + shadcn/ui | Fast, consistent, Jupid-like customization |
| Typography | Playfair Display (serif) + Inter (sans) | Warm + readable |
| Blog | MDX via next-mdx-remote or @next/mdx | Files in repo, Publisher creates PRs |
| Waitlist | Supabase table + API route | Zero dependencies |
| Hosting | Vercel | Zero-config from root |
| Analytics | Vercel Analytics (free tier) | Built-in, no GDPR cookie banner needed |
| SEO | next-sitemap + structured data | Article schema for blog posts |

## Database (minimal for v1)

Only 1 table needed:

```sql
CREATE TABLE waitlist (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email text NOT NULL UNIQUE,
  source text,              -- 'landing_hero', 'landing_bottom', 'blog_cta'
  referrer text,            -- utm_source or HTTP referrer
  created_at timestamptz DEFAULT now()
);
```

Full schema from TECH_SPEC deferred until dashboard/WhatsApp phase.

## Blog Engine

### MDX Pipeline
- Articles in `/content/blog/*.mdx`
- Frontmatter: title, description, date, tags, keywords, author, readingTime
- Rendered via next-mdx-remote (dynamic) or @next/mdx (static)
- Sitemap auto-generated via next-sitemap
- RSS feed at /blog/rss.xml

### Seed Articles (from VoC)
1. **"How to challenge a service charge increase in England"**
   - Target: "how to dispute service charge UK" (1,000-2,000 monthly searches)
   - Pain: charges double with no breakdown
2. **"Section 20 notice explained: what leaseholders need to know"**
   - Target: "section 20 notice explained"
   - Pain: confusing legal notices about major works
3. **"Your managing agent isn't responding — here's what to do next"**
   - Target: "management company not responding to repairs"
   - Pain: silence, delays, no accountability

## Implementation Phases

### Phase S0: Scaffold (1-2h)
- [ ] S0.1 Init Next.js 15 at repo root (npx create-next-app@latest)
- [ ] S0.2 Install + configure shadcn/ui, Tailwind, fonts (Playfair Display + Inter)
- [ ] S0.3 Set up color tokens and theme config
- [ ] S0.4 Supabase project: create waitlist table
- [ ] S0.5 Env vars: NEXT_PUBLIC_SUPABASE_URL, SUPABASE_ANON_KEY
- [ ] S0.6 Deploy empty shell to Vercel
- [ ] S0.7 Commit + push

### Phase S1: Landing Page (3-4h)
- [ ] S1.1 Navbar: logo + "Blog" link + "Join waitlist" CTA button
- [ ] S1.2 Hero section: problem-first headline, subheadline, email input + CTA
- [ ] S1.3 Problem section (dark bg): 4 pain cards from VoC
- [ ] S1.4 How it works: 3-step visual (forward → clarity → draft)
- [ ] S1.5 Features grid (dark bg): 6 feature cards
- [ ] S1.6 Social proof placeholder
- [ ] S1.7 Bottom CTA section (amber bg): email capture repeat
- [ ] S1.8 Footer: links, legal, copyright
- [ ] S1.9 Waitlist API route: POST /api/waitlist → Supabase insert
- [ ] S1.10 Mobile responsive pass
- [ ] S1.11 SEO: meta tags, OG image, robots.txt
- [ ] S1.12 Commit + push + verify on Vercel

### Phase S2: Blog Engine (2-3h)
- [ ] S2.1 MDX pipeline: content/blog/*.mdx → pages
- [ ] S2.2 Blog index page: card list with tags, dates
- [ ] S2.3 Article page: layout, TOC, reading time, share links
- [ ] S2.4 CTA banner component in article pages
- [ ] S2.5 Structured data (Article schema) for SEO
- [ ] S2.6 Sitemap via next-sitemap
- [ ] S2.7 RSS feed at /blog/rss.xml
- [ ] S2.8 Commit + push

### Phase S3: Seed Content (2-3h)
- [ ] S3.1 Write article 1: "How to challenge a service charge increase"
- [ ] S3.2 Write article 2: "Section 20 notice explained"
- [ ] S3.3 Write article 3: "Managing agent not responding — what to do"
- [ ] S3.4 Review all 3 for accuracy (cite GOV.UK, LEASE, legislation)
- [ ] S3.5 Commit + push + verify on Vercel

### Phase S4: Polish (1-2h)
- [ ] S4.1 Privacy policy page (/privacy)
- [ ] S4.2 Terms of service page (/terms)
- [ ] S4.3 Vercel Analytics setup
- [ ] S4.4 Lighthouse audit: target > 90 all categories
- [ ] S4.5 OG image for social sharing
- [ ] S4.6 Final review + push

## File Structure

```
Leasehold-buddy/              # repo root = Next.js app
├── app/
│   ├── layout.tsx            # root layout: fonts, metadata, analytics
│   ├── page.tsx              # landing page
│   ├── blog/
│   │   ├── page.tsx          # blog index
│   │   └── [slug]/
│   │       └── page.tsx      # individual article
│   ├── privacy/
│   │   └── page.tsx
│   ├── terms/
│   │   └── page.tsx
│   └── api/
│       └── waitlist/
│           └── route.ts      # POST: email → Supabase
├── components/
│   ├── ui/                   # shadcn components
│   ├── navbar.tsx
│   ├── hero.tsx
│   ├── problem-section.tsx
│   ├── how-it-works.tsx
│   ├── features-grid.tsx
│   ├── social-proof.tsx
│   ├── cta-section.tsx
│   ├── footer.tsx
│   ├── waitlist-form.tsx
│   ├── blog-card.tsx
│   ├── article-layout.tsx
│   └── blog-cta-banner.tsx
├── content/
│   └── blog/
│       ├── challenge-service-charge.mdx
│       ├── section-20-explained.mdx
│       └── managing-agent-not-responding.mdx
├── lib/
│   ├── supabase.ts           # Supabase client
│   ├── blog.ts               # MDX utilities (getAllPosts, getPostBySlug)
│   └── constants.ts          # site metadata, nav links
├── public/
│   ├── og-image.png
│   └── favicon.ico
├── agents/                   # Scout + Publisher (separate workstream)
├── data/                     # pain packets (separate workstream)
├── tasks/                    # plans, lessons
├── tailwind.config.ts
├── next.config.mjs
├── package.json
└── tsconfig.json
```

## Verification

1. `npm run build` succeeds with zero errors
2. Landing page loads < 2s, Lighthouse > 90
3. Waitlist form submits email to Supabase (check table)
4. Blog index shows 3 articles, each renders correctly
5. Articles have proper meta tags, OG data, structured data
6. Sitemap at /sitemap.xml includes all pages
7. RSS at /blog/rss.xml is valid
8. Mobile: all sections readable and tappable
9. Privacy/terms pages exist and link from footer

## Total Estimated Time: ~10-12h across 4 phases
