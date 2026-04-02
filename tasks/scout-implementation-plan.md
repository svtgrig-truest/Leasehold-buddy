# Leasehold Scout — Implementation Plan

## Context

Leasehold Buddy (Common Ground) is an AI-powered case management platform for UK leaseholders (documentation stage, no code written yet). We need a promotion agent called "Leasehold Scout" to monitor Reddit/MSE, generate reply drafts, and collect pain packets for future content creation.

Two variants of Scout will be built: a simple Python cron script and an OpenClaw agent on 192.168.0.237. Both do the same job using different tooling. Goal: compare approaches and evaluate OpenClaw for agent workflows.

## Scope

### What Scout does (both variants):
1. Scans r/HousingUK by keywords (leasehold, service charge, managing agent, Section 20, major works, RTM, reserve fund)
2. Scores threads by reply_worthiness (freshness, specificity, intent)
3. Generates a draft reply (value-first, no product links, official sources only)
4. Sends draft to Telegram for human approval
5. Listens to MSE forum (listen-only) — extracts pain statements, search-like phrases
6. Builds pain packets for the future Leasehold Publisher agent
7. Persists state (seen threads) to avoid duplicate processing

### What Scout does NOT do:
- No auto-posting (human approval mandatory)
- No Leasehold Buddy links in any reply
- No replies on MSE
- No more than 10 threads/run, 3 full-thread opens/run

## Architecture

### Shared components (used by both variants):
- **Reddit API wrapper** — OAuth2 auth, search/fetch posts from r/HousingUK
- **Thread scorer** — reply_worthiness evaluation (age < 12h, concrete question, not already well-answered, leasehold-specific)
- **Draft generator** — Claude API prompt for generating helpful replies
- **Telegram bot** — draft delivery with Approve/Edit/Skip buttons
- **State store** — SQLite or JSON file for seen_threads, pain_packets
- **Pain packet builder** — structures recurring pains into actionable data

### Variant A: Python Cron

```
/agents/scout-cron/
├── scout.py              # main entry point
├── reddit_client.py      # Reddit API wrapper (PRAW)
├── mse_listener.py       # MSE scraper (BeautifulSoup)
├── scorer.py             # thread scoring logic
├── drafter.py            # Claude API draft generation
├── telegram_bot.py       # Telegram bot (python-telegram-bot)
├── pain_packets.py       # pain packet builder
├── state.py              # SQLite state management
├── config.py             # env vars, constants
├── requirements.txt
├── .env.example
└── README.md
```

**Execution:** `crontab -e` → `0 */4 * * * cd /path && python3 scout.py scan`
**Telegram bot:** separate long-polling process (systemd service or `nohup`)
**Cadence:** scan every 4 hours, Telegram bot always-on

### Variant B: OpenClaw Agent

```
~/.openclaw/workspace-scout/    (on 192.168.0.237)
├── SOUL.md                     # identity: Leasehold Scout
├── AGENTS.md                   # operational procedures
├── HEARTBEAT.md                # patrol checklist (scan Reddit every 30 min)
├── USER.md                     # info about Svetlana
├── memory/                     # persistent memory
└── skills/
    ├── reddit_monitor/
    │   └── SKILL.md            # Reddit scanning skill
    ├── mse_listener/
    │   └── SKILL.md            # MSE listening skill
    ├── draft_reply/
    │   └── SKILL.md            # Reply drafting skill
    └── pain_capture/
        └── SKILL.md            # Pain packet extraction skill
```

**Telegram:** built-in OpenClaw integration (grammY), configured in openclaw.json
**Reddit API:** via exec tool (Python script) or native plugin
**Cadence:** HEARTBEAT every 30 min + cron for daily pain clustering
**State:** OpenClaw memory system (memory/*.md files)

## Implementation Steps

### Phase 0: Prerequisites (1-2h)
- [ ] 0.1 Create Telegram bot via @BotFather, save token
- [ ] 0.2 Register Reddit app at reddit.com/prefs/apps (script type), obtain client_id + client_secret
- [ ] 0.3 Verify SSH access to 192.168.0.237, inspect existing OpenClaw setup

### Phase 1: Python Cron Scout (4-5h)
- [ ] 1.1 Scaffold `/agents/scout-cron/` in monorepo
- [ ] 1.2 Reddit client (PRAW): OAuth2 auth, search r/HousingUK by keywords, fetch posts
- [ ] 1.3 Thread scorer: age, specificity, intent detection, existing answers quality
- [ ] 1.4 Draft generator: Claude API prompt with safe-link policy and tone guidelines
- [ ] 1.5 Telegram bot: inline keyboard (Approve / Edit / Skip), draft delivery
- [ ] 1.6 State management: SQLite — seen_threads, draft_status, pain_packets
- [ ] 1.7 MSE listener: scrape titles + opening posts from House buying section
- [ ] 1.8 Pain packet builder: extract recurring pains, user language, search-like phrases
- [ ] 1.9 Main orchestrator: wire everything, cron-friendly CLI
- [ ] 1.10 Tests + manual verification

### Phase 2: OpenClaw Scout (3-4h)
- [ ] 2.1 SSH to 192.168.0.237, inspect existing agent, create second: `openclaw agents add scout`
- [ ] 2.2 Write SOUL.md — identity, values, tone for Leasehold Scout
- [ ] 2.3 Write AGENTS.md — operational procedures, memory protocol, safe-link policy
- [ ] 2.4 Write HEARTBEAT.md — patrol checklist (check Reddit, scan keywords)
- [ ] 2.5 Create skill `reddit_monitor` — instructions + Python helper script for Reddit API
- [ ] 2.6 Create skill `mse_listener` — MSE scanning instructions
- [ ] 2.7 Create skill `draft_reply` — reply generation with guardrails
- [ ] 2.8 Create skill `pain_capture` — pain packet extraction
- [ ] 2.9 Configure Telegram in openclaw.json for scout agent
- [ ] 2.10 Cron job for daily pain clustering: `openclaw cron add --name "daily-pain-cluster" ...`
- [ ] 2.11 Test full cycle: scan → score → draft → Telegram → approve

### Phase 3: Comparison & Docs (1h)
- [ ] 3.1 Run both variants in parallel for 1-2 days
- [ ] 3.2 Compare: draft quality, token usage, latency, ease of maintenance
- [ ] 3.3 Document findings in tasks/lessons.md

## Key Files to Create/Modify

**Monorepo (Leasehold-buddy/):**
- NEW: `/agents/scout-cron/` — entire Python cron variant
- NEW: `/agents/scout-openclaw/` — mirror of OpenClaw files (SOUL.md, AGENTS.md, skills/) for version control

**OpenClaw machine (192.168.0.237):**
- NEW: `~/.openclaw/workspace-scout/SOUL.md`
- NEW: `~/.openclaw/workspace-scout/AGENTS.md`
- NEW: `~/.openclaw/workspace-scout/HEARTBEAT.md`
- NEW: `~/.openclaw/workspace-scout/skills/*/SKILL.md`
- EDIT: `~/.openclaw/openclaw.json` — add scout agent + Telegram config

## Config & Credentials

```
REDDIT_CLIENT_ID=<from reddit.com/prefs/apps>
REDDIT_CLIENT_SECRET=<secret>
REDDIT_USERNAME=Honest_Addendum_8517
REDDIT_PASSWORD=<password>
TELEGRAM_BOT_TOKEN=<from BotFather>
TELEGRAM_CHAT_ID=<Svetlana's chat ID>
ANTHROPIC_API_KEY=<for Claude API in cron variant>
```

## Safe-Link Policy (embedded in both variants)

| Tier | Status | Examples |
|------|--------|---------|
| A: Safe | Use freely | GOV.UK, LEASE, First-tier Tribunal, Property Ombudsman |
| B: Conditional | Use when tightly matched | Citizens Advice, council pages |
| C: Unsafe | Never in v1 | Leasehold Buddy links, any tracking URLs |

## Reply Scoring Rubric

| Factor | Weight | Criteria |
|--------|--------|----------|
| Freshness | 30% | < 6h = 10, 6-12h = 7, 12-24h = 3, > 24h = 0 |
| Specificity | 25% | Concrete question with details = 10, vague complaint = 3 |
| Intent match | 25% | Dispute help / rights check / next steps = 10, general chat = 2 |
| Answer gap | 20% | No good answers yet = 10, partial = 5, well-answered = 0 |
| Minimum score: 6.0 to generate draft |

## Pain Packet Schema (v1 simplified — 6 fields)

```json
{
  "packet_id": "pp_20260402_001",
  "source": { "channel": "reddit", "thread_url": "...", "thread_title": "..." },
  "persona": { "role": "leaseholder|director", "stage": "early|escalating|tribunal" },
  "primary_pain": "Service charge doubled with no breakdown provided",
  "user_phrases": ["no breakdown", "how do I challenge this", "section 20"],
  "seo_intent": "how to challenge service charge increase uk"
}
```

## Verification

1. **Cron variant:** run `python3 scout.py scan --dry-run`, verify it finds threads, scores them, generates drafts, sends to Telegram
2. **OpenClaw variant:** trigger heartbeat manually, verify Telegram delivery
3. **Both:** confirm no Leasehold Buddy links in any draft
4. **Both:** confirm state persistence (re-run doesn't duplicate threads)
5. **Both:** verify pain packet output format

## Unresolved Questions

1. ~~SSH access to 192.168.0.237~~ — network unreachable, will fix during Phase 2 deployment
2. MSE scraping legality — need to check robots.txt and ToS before launching listener
3. Reddit API tier — is free research access sufficient or do we need a paid tier?
4. Telegram bot name — @LeaseholdScoutBot?
5. Pain packet storage — SQLite in cron, memory/*.md in OpenClaw. Do we need a unified format for Publisher?
