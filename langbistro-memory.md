# LangBistro — Project Memory File

## What is LangBistro
A Telegram-based AI-powered language learning bot targeting English-speaking users in the US market. Currently supports **Spanish and French**. The AI tutor persona is named **Bistro**. The product is freemium with a subscription paywall. The CEO (Vasilii) drives product priorities; Claude acts as CTO/COO; Cursor is the coding agent.

**Business goals:**
- Build a clean, acquisition-ready codebase from day 1
- Keep infra costs low
- Ship fast without regressions
- Target 500-600 users in first 3-6 months

---

## Working Relationship & Workflow
- Claude acts as CTO/COO: architecture, Cursor prompt authoring, code review
- Vasilii is CEO and head of product: drives priorities, tests features, reports back
- Cursor is the coding agent: implements prompts, returns status reports
- GitHub Desktop used for version control (not terminal git)

**Workflow loop:**
1. CEO describes feature or bug
2. CTO asks clarifying questions until fully clear
3. CTO writes discovery prompt for Cursor (gathers file names, function names, structure)
4. Cursor returns discovery response
5. CTO breaks task into phases
6. CTO writes Cursor prompt per phase, asking for a status report on each
7. CEO passes prompts to Cursor and returns status reports

**Important:** Break Cursor prompts into small focused chunks. Large prompts cause boot-looping.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 20 TypeScript (Node 18 breaks Whisper uploads — never downgrade) |
| Bot framework | GrammY |
| Database | Supabase (Postgres, us-east-1) |
| Hosting | Railway (live: langbistro-bot-production.up.railway.app) |
| Package manager | npm |
| AI — Conversation (free) | OpenAI GPT-4o-mini |
| AI — Conversation (paid) | OpenAI GPT-4o |
| AI — Explain button | OpenAI GPT-4o (all users) |
| AI — Quiz grading / moderation / fill-blank gen | OpenAI GPT-4o-mini |
| AI — Transcription | OpenAI Whisper (language hint: 'es') |
| AI — Voice replies | OpenAI TTS (tts-1, alloy voice, speed: beginner=0.85x, intermediate/advanced=1.0x) |
| Payments | Paddle (MoR, live and verified with real payment) |
| Website | langbistro.com (Cloudflare Pages, plain HTML) |
| Repos | langbistro-bot (private), langbistro-web (public) |

---

## Product Description

### Core mechanics
- Conversational AI chat in Spanish (voice-first)
- Bistro speaks Spanish by default; switches to English only via Explain button
- Voice replies use TTS; user can send text or voice
- Corrections shown inline; Explain button provides English grammar explanation
- replyExplanation uses "I said..." / "I asked you..." phrasing, never "the user"

### Freemium model
- Free tier: 10 text messages + 3 voice messages per day (resets midnight) + daily vocabulary words
- Subscribed users: unlimited conversation + unlimited vocabulary
- Daily vocabulary delivery is available to ALL users (free and paid)
- Conversation limits only apply to text/voice message count

### Pricing
- Free: $0 (permanent freemium)
- Pro Monthly: $11.99/mo (pri_01kmzyffw6df3z0d2gh851fz5j)
- Pro Annual: $79.99/yr (~$6.67/mo, 44% saving) (pri_01kmzygpx2955dsdetymrqd3rg)
- Tax-inclusive; Paddle handles VAT as MoR

### Onboarding flow
1. /start → welcome message explaining Bistro
2. Three level buttons: Beginner / Intermediate / Advanced (callback: onboarding_level:*)
3. Level saved → six time selection buttons in ET: 8am, 11am, 2pm, 5pm, 8pm, 11pm
4. Time saved → onboarding_complete=true → opening conversation sent as text + voice with 📖 Read button
   - beginner: "¡Hola! Soy Bistro. ¿Cómo te llamas?"
   - intermediate: "¡Hola! Soy Bistro. ¿Cómo te llamas y de dónde eres?"
   - advanced: "¡Buenas! Soy Bistro. Cuéntame — ¿cómo te llamas y qué te trae aquí?"
- Onboarding state stored in Set<number> (process-local)

### Conversation system (level-aware)
- Beginner (A1-A2): max 8 word sentences, present tense only, one question at a time
- Intermediate (B1-B2): 10-15 word sentences, past/present/future tenses
- Advanced (C1-C2): full tenses including subjunctive, no simplification, idioms used freely
- Level fetched from DB at start of each runAssistantTurn

### Vocabulary system
- 4,438 Spanish words seeded (frequency-ordered, tiers 1-5)
- Tier boundaries: 1-500=t1, 501-1500=t2, 1501-2500=t3, 2501-3500=t4, 3501+=t5
- Daily word delivery at user's preferred time via cron job
- Only fill-blank quiz per session (review quiz removed)
- engaged flag set BEFORE sending to prevent duplicate delivery on same-minute cron ticks
- GPT-generated fill-blank sentences using exact word form

### Milestones
Sent at: 50, 100, 250, 500, 750, 1000, 1500, 2000, 3000, 4000 words learned

### Key UX flows
- Normal conversation: user sends text/voice → correction line (if mistake) + Explain button → voice note reply → Read/Explain buttons
- Daily words: cron fires at preferred_word_time → 10 word cards + 🔊 buttons → fill-in-blank prompt → quiz state set in memory
- Quiz: user replies → evaluated → correct/wrong feedback + voice → Explain button → clears quiz state → marks word learned
- /settings: level display + 3 level buttons + 6 time slot buttons (ET). No subscription info.
- /subscribe: if free → "⚡ Subscribe to Pro" button → langbistro.com/subscribe?telegram_id=X; if subscribed → Pro status + support@langbistro.com

### Inactivity re-engagement
- Never engaged after onboarding: 24h nudge (stage 1), 72h nudge (stage 2)
- Was active, then stopped: day 7 (stage 1), day 15 (stage 2), day 30 (stage 3), day 45 (stage 4)
- Daily words paused after day 7 inactivity, auto-resumes on message
- Stage values: 0=active, 1=day7/24h, 2=day15/72h, 3=day30, 4=day45

### Moderation & safety
- OpenAI omni-moderation-latest for content checking
- 3 violations → warning; 5 violations → soft ban
- Violation logs: type + timestamp only (no message content stored)

---

## File Structure

```
src/
├── ai/
│   └── openai.ts
├── bot/
│   ├── index.ts
│   ├── ux-memory.ts
│   └── handlers/
│       ├── callback.ts
│       ├── message.ts
│       ├── settings.ts
│       ├── start.ts
│       ├── subscribe.ts
│       ├── ux-flow.ts
│       └── voice.ts
├── config/
│   └── env.ts
├── db/
│   ├── client.ts
│   ├── schema.sql
│   └── seeds/
│       ├── vocabulary.ts
│       ├── vocabulary-5k.ts      # resumable via upsert — safe to re-run
│       └── run-seed.ts
├── jobs/
│   ├── wordDelivery.ts
│   └── inactivityJob.ts
├── server.ts
└── services/
    ├── conversation.ts
    ├── dailySession.ts
    ├── milestone-notify.ts
    ├── moderation.ts
    ├── onboarding.ts
    ├── quizHandler.ts
    ├── quizState.ts
    ├── usage.ts
    ├── users.ts
    └── vocabulary.ts
└── utils/
    ├── dateTz.ts
    └── text.ts
```

---

## Database Schema

### users
telegram_id, username, language_code, preferred_word_time, preferred_word_timezone, is_subscribed, is_banned, violation_count, level, onboarding_complete, words_learned_count, current_tier, last_active_at, inactivity_stage, created_at

### messages
user_id (FK), role, content, message_type, created_at

### usage_daily
user_id (FK), date, text_count, voice_count — UNIQUE(user_id, date)

### word_sets
user_id (FK), words (JSONB), sent_at

### subscriptions
user_id (FK), paddle_customer_id, paddle_subscription_id, status, current_period_end

### violations
user_id (FK), violation_type, created_at

### vocabulary
word, translation, example_sentence, tier, frequency_rank — UNIQUE(word)

### user_vocabulary
user_id (FK), vocabulary_id (FK), learned_at — UNIQUE(user_id, vocabulary_id)

### daily_sessions
user_id (FK), date, words_sent, fill_blank_word_id, review_word_id, engaged

**All tables:** RLS enabled (tighten before production at scale)

---

## In-Memory State (process-local — lost on restart)

| State | Purpose |
|---|---|
| Onboarding Set<number> | Tracks onboarding in progress |
| Quiz state map | Active quiz type, word, sentence, vocabularyId (keyed by telegram_id) |
| UX callback payloads | Reply text and explanations for Read/Explain buttons (keyed by chatId+messageId) |

All maps: max 1000 entries, evict oldest when full.

---

## Paddle Setup

- Webhook: POST /webhook/paddle on Railway
- Handles: subscription.activated, subscription.updated, subscription.canceled, subscription.past_due
- Signature verification implemented
- is_subscribed toggled correctly — verified with real payment
- Checkout: /subscribe → langbistro.com/subscribe → user picks plan → Paddle overlay opens
- Subscription management: support@langbistro.com (authenticated portal deferred post-launch)

---

## Website (langbistro-web)

- langbistro.com — Cloudflare Pages, plain HTML
- Pages: index.html, pricing.html, subscribe.html, terms.html, privacy.html, refund.html
- subscribe.html: reads telegram_id from URL param, shows two plan cards, Paddle.js v2 overlay on tap

---

## Environment Variables (Railway)

TELEGRAM_BOT_TOKEN, OPENAI_API_KEY, SUPABASE_URL, SUPABASE_ANON_KEY,
NODE_ENV=production, NODE_VERSION=20, PADDLE_API_KEY, PADDLE_WEBHOOK_SECRET,
PADDLE_MONTHLY_PRICE_ID, PADDLE_YEARLY_PRICE_ID

---

## Phase Progress

| Phase | Status | Description |
|---|---|---|
| Phase 1 | ✅ Complete | Scaffold, DB, GrammY bot, echo handler |
| Phase 2 | ✅ Complete | GPT-4o conversation, Whisper, TTS, moderation, usage limits, ban system |
| Phase 2b | ✅ Complete | UX: corrections, voice notes, Read/Explain buttons |
| Phase 3a | ✅ Complete | Vocabulary (4,438 words, 5 tiers), daily word delivery, quiz flow |
| Pre-Phase 4 | ✅ Complete | Model tiering, inactivity system |
| Phase 4 | ✅ Complete | Railway deployment, Paddle subscriptions, HTTP server, webhook, /subscribe, /settings |
| Phase 5 | ✅ Complete | New onboarding, level-based difficulty, Whisper fix, voice fix, Explain tone fix, BotFather setup |
| Phase 5b | ✅ Complete | Beta feedback fixes: TTS speed by level, re-engagement reminders, duplicate quiz fix, GPT fill-blank sentences, 4,438 word seed, milestone thresholds |
| Phase 6 | ✅ Complete | French language support — onboarding, AI layer, settings, 9,900 word seed |

---

## Immediate Next Priorities

1. Automatic level progression detection
2. Response lag reduction (~40% improvement via GPT streaming + parallel TTS)
3. Telegram Mini App for native checkout experience
4. French vocabulary — 100 words missing from seed (batch 94 failed during seeding, words around frequency rank 4,700-4,800). Re-run npm run seed:fr when convenient to fill the gap.

---

## Technical Debt (Priority Order)

1. **In-memory state not shared across instances** — move onboarding state, quiz state, UX callback payloads to DB before multi-instance scaling
2. **Quiz state not cleared on bot restart** — user can get stuck in quiz
3. **daily_sessions cleanup job** — rows older than 90 days accumulate; needs cron
4. **RLS policies need tightening** before production scale
5. **Response lag** — parallelize GPT streaming + parallel TTS (do when complaints arise)
6. **Subscription management via email only** — authenticated Paddle portal deferred post-launch
7. **Payment UX** — subscribe page requires manual plan tap; revisit when Paddle hosted checkout approved or build Telegram Mini App
8. **Automatic level assessment** — suggest level upgrade after 30 days or 50 messages
9. **RLS + service role key** — bot connects with SUPABASE_ANON_KEY; RLS policies are allow_all (USING true). Should switch to service role key + least-privilege RLS before production scale. Low risk while anon key is kept server-side only.
10. **Non-atomic DB operations** — usage quota (usage.ts), session engagement flag (dailySession.ts), and markWordsLearned (vocabulary.ts) are all read-modify-write sequences with no transactions. Can race under concurrent requests. Fix before scaling beyond ~1000 DAU.
11. **Supabase Data API grant changes (deadline: October 30, 2026)** — Supabase is enforcing explicit GRANTs on public schema tables for the anon role. Bot uses SUPABASE_ANON_KEY (anon role via PostgREST). Without action before Oct 30, all DB queries will fail with 42501 errors. Fix: switch to service_role key (resolves both this and item #9 in one change). Schedule for Aug/Sep 2026.

---

## Key Principles (Engineering)

- Cursor prompts: small focused chunks — large prompts cause boot-looping
- Supabase RLS policies can silently block operations — check first when debugging DB issues
- GrammY bot.start() is blocking — cron jobs must be started before it
- Telegram MarkdownV2 requires careful escaping
- GPT-4o partial JSON can fail Zod — normalize first
- Node.js 20+ required — never downgrade
- Two polling instances conflict — never run local npm run dev while Railway is live
- Whisper must have language: 'es' hint (prevents Arabic misidentification)
- engaged flag must be set BEFORE sending messages to prevent duplicate delivery
- vocabulary-5k.ts seed is resumable via upsert — safe to re-run

---

## Key Product Decisions (Do Not Revisit)

- **Telegram only** — WhatsApp dropped for v1
- **Spanish only for v1** — French is Phase 6; architecture changes needed
- **English UI** — Bistro speaks Spanish for learning; UI messages in English
- **Paddle over Stripe** — fixed constraint, not revisitable
- **Voice-first** — all conversation replies include voice note
- **Accent marks ignored in user input** — normalizeText() strips accents

---

## Security Notes

- `.env` never committed to GitHub
- Bot token never in logs — redaction regex in bot.catch
- If token exposed: revoke immediately via @BotFather → /mybots → API Token → Revoke
- Supabase anon key is public-safe but RLS must be tightened at scale
- Violation logs: type + timestamp only (GDPR-friendly)

---

## Acquisition Readiness Notes

- Legal documents in place: Terms of Service, Privacy Policy, Refund Policy
- Paddle as MoR handles tax compliance automatically
- Clean modular codebase with clear phase separation
- Analytics instrumentation planned (key metrics: DAU, retention, free-to-paid conversion)
