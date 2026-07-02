# AvatarChat — Product Spec

> One-line: A consumer platform where you talk out loud to lifelike AI avatars — they listen,
> understand, and answer back in a natural human voice with an animated, lip-synced face.
> Tenant slug: `avatarchat` · Currency: INR (paise) · Surfaces: backend · web · mobile
>
> *Working name — rename freely; only the tenant slug propagates into code.*

## 1. Problem & value

- **Who has the problem:** People who want conversational practice, companionship, tutoring,
  or coaching find text chatbots flat and human services expensive/unavailable. Today they
  type at text LLMs or watch one-way videos.
- **Why AvatarChat is better:** Full-duplex *voice-to-voice* conversation with a visible,
  animated avatar — speak naturally, get a spoken reply in a human-like voice with a
  lip-synced face, in your language. Feels like a call, not a chat.
- **Primary persona:** `user` (consumer). Secondary: `avatarchat_staff` (platform admin).
- **Core interaction model:** Human ↔ AI avatar (voice → STT → LLM → TTS → lip-synced
  avatar render). Human↔human avatar calls are explicitly out of scope for v1 (§8).

### Success metrics
- Activation: % of new users who complete a first conversation ≥ 60 seconds.
- Engagement: weekly voice minutes per active user; D7 retention.
- Quality: median time-to-first-audio per avatar turn < 1.5 s; conversation completion rate.
- Creation loop: # of user-created avatars published to the public catalog.

## 2. Personas

Single-tenant consumer product — **no `organization_id`**; `product_id` kept per scaffold
convention. Persona set is deliberately small.

| Persona | Description | Default home | Can do (summary) |
|---------|-------------|-------------|------------------|
| guest | unauthenticated visitor | `/` | browse public avatar catalog, view pricing |
| user | consumer; talks to avatars, builds personal avatars | `/avatars` | start/end voice conversations, view own history, create & manage own avatars, pick voices, view own usage |
| avatarchat_staff (super admin) | platform admin | `/admin` | everything: moderate user avatars, manage stock voices, plans, users, view platform usage |

## 3. Modules & entities (see MODULE_MANIFEST.md for detail)

| Module | Type (transactional\|catalog) | Entities |
|--------|------------------------------|----------|
| plans | catalog (seeded, public read) | Plan |
| voices | transactional | Voice |
| avatars | catalog (public browse) **+ transactional exposure** (studio/admin) — dual-casing | Avatar |
| conversations | transactional | Conversation, Message |
| usage | transactional | UsageRecord |

## 4. Screen list (drives web pages + mobile prototype)

- **Public (`(site)`):**
  - `/` — home: hero, featured avatars, how-it-works
  - `/avatars` — catalog: category/language filters, search, featured strip
  - `/avatars/[slug]` — avatar detail: portrait, voice sample, greeting, "Start talking" CTA
  - `/pricing` — plans (all-free at launch, tiers displayed as "coming soon")
- **Authed (`(app)`):**
  - `/talk/[avatarSlug]` — live voice session: animated avatar, mic state, waveform, live captions, end-call
  - `/conversations` — history list (by avatar, date)
  - `/conversations/[id]` — transcript with per-turn audio playback
  - `/studio` — my avatars list (status chips: draft/pending_review/published)
  - `/studio/new`, `/studio/[id]/edit` — avatar builder: persona prompt, greeting, voice picker (with samples), visual style, visibility
  - `/settings` — profile, preferred language, mic/device check
- **Admin (`(admin)`):**
  - `/admin` — dashboard: platform voice minutes, active users, top avatars
  - `/admin/moderation` — user-avatar review queue (approve/reject → published/draft)
  - `/admin/users` — user management
  - `/admin/voices` — stock voice management
  - `/admin/plans` — plan management (dormant until billing)
- **Mobile verticals/personas:** single consumer vertical, talk-first — Home (avatar grid),
  Talk (full-screen avatar call), History, Studio (basic create/edit), Profile.

## 5. Permission catalog

| Permission | Granted to personas |
|------------|--------------------|
| avatars.view | guest, user (public catalog) |
| avatars.manage | user (own avatars only) |
| admin.avatars.moderate | avatarchat_staff |
| conversations.manage | user (own only) |
| voices.view | user (stock + own) |
| voices.manage | user (own cloned voices — deferred) |
| admin.voices.manage | avatarchat_staff |
| plans.view | guest, user |
| admin.plans.manage | avatarchat_staff |
| usage.view | user (own only) |
| admin.usage.view | avatarchat_staff |
| admin.users.manage | avatarchat_staff |

Wildcards available: `*.*`, `admin.all`, `{module}.all`. Ownership checks (own avatar /
own conversation) are enforced in the service layer on top of the permission gate.

## 6. Plans & limits (monetization DEFERRED — spec'd now, unenforced at launch)

Billing/enforcement is out of scope for v1. Plans are seeded and displayed; every account
runs as Free with limits tracked (via `usage`) but **not enforced**. Paid tiers are
inactive rows (`is_active = false`) so the pricing page and future billing need no schema
change.

| Plan | Price (paise/mo) | Key features | Limits |
|------|-----------------|--------------|--------|
| Free | 0 | catalog access, personal avatars | maxAvatars: 3 · monthlyVoiceMinutes: 300 · maxClonedVoices: 0 |
| Plus (inactive) | 49900 | more minutes, priority TTS | maxAvatars: 10 · monthlyVoiceMinutes: 1500 · maxClonedVoices: 1 |
| Pro (inactive) | 149900 | max minutes, early features | maxAvatars: 50 · monthlyVoiceMinutes: 6000 · maxClonedVoices: 5 |

## 7. Money & units

- Monetary fields (all paise): `plans.price_paise` — the only money field in v1.
- Display: `formatInr()` web · `price_formatted` backend (catalog: `priceFormatted`) · paise everywhere in storage/transport.
- Time units: conversation duration in **seconds** (`duration_seconds`), audio in
  **milliseconds** (`audio_duration_ms`, `latency_ms`), usage metered in **seconds**
  (`seconds_used`); minutes are a display-only derivation.

## 8. Out of scope / deferred

- **Billing & plan enforcement** (plans seeded, limits tracked, nothing blocked).
- **Voice cloning pipeline** (`Voice.kind = cloned` is in the contract; upload/processing deferred).
- **Human↔human avatar calls** (translation/dubbing/anonymized presence) — session model doesn't preclude it, but no v1 work.
- **PSTN/phone dial-in, push notifications, payments, avatar marketplace/payouts.**
- **Realtime media plane is not a Laravel module:** the live loop (WebSocket/WebRTC
  gateway → STT → LLM → TTS → viseme/lip-sync stream) runs as a separate realtime service.
  Laravel owns identity, catalog, conversation/message persistence, and usage, and issues
  short-lived realtime session tokens (`POST /conversations/{id}/realtime-token`). The
  gateway writes turns back through an internal message-append endpoint.

## 9. Data contracts

One per entity, in `contracts/`:
- `contracts/avatar.md` (dual-casing: public catalog camelCase + studio/admin snake_case)
- `contracts/conversation.md`
- `contracts/message.md`
- `contracts/voice.md`
- `contracts/plan.md` (catalog camelCase)
- `contracts/usage-record.md`
