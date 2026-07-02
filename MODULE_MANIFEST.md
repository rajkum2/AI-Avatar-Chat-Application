# AvatarChat — Module Manifest

For each module decide **transactional** (authed, own `Routes/api.php`, registered in
`config/modules.php`, snake_case Resource + `_formatted` money) vs **catalog** (public
read-only, routes added to root `routes/api.php`, camelCase Resource mirroring the web
mock shape, often a separate admin controller using the `guard()` permission pattern).

Single-tenant consumer: **no `organization_id`** anywhere; `product_id` kept per scaffold.

| Module | Type | Entities | Public? | Notes |
|--------|------|----------|---------|-------|
| plans | catalog | Plan | yes (read) + admin manage | seeded rows; `list` only; camelCase `pricePaise`/`priceFormatted`; `AdminPlanController` with `guard(admin.plans.manage)` |
| voices | transactional | Voice | no | stock voices seeded; user-cloned voices deferred (contract ready); authed read for the studio voice picker |
| avatars | catalog + transactional (dual-casing) | Avatar | yes (read, published+public only) + authed studio CRUD + admin moderation | PropYaar-Projects pattern: public camelCase catalog Resource (list/findBySlug/similar) **and** snake_case studio/admin Resource; `personality_prompt` NEVER in the public Resource |
| conversations | transactional | Conversation, Message | no | core module; start/end session, realtime-token endpoint, internal message-append for the gateway; messages paginated asc by `sequence` |
| usage | transactional | UsageRecord | no | one row written per conversation-end; user sees own, admin sees all; feeds future billing |

## Decision matrix (from the verified backend recipe)
- **Transactional / authenticated** (voices, conversations, usage; avatars-studio):
  own `Routes/api.php`, `config/modules.php` registration, full DTO + FormRequest + Event
  stack, snake_case Resource with `_paise` + `_formatted`. Gated inside the auth +
  `account.approved` route group.
- **Public read-only catalog** (plans; avatars-catalog): NO `Routes/` dir — add
  `Route::get(...)` lines to the public group of the root `routes/api.php`; lighter
  service (`list`/`findBySlug`/`similar`); camelCase Resource mirroring the frontend mock
  JSON; separate `Admin{Entity}Controller` using `guard(self::PERMISSION)`.
- **Dual-casing (avatars only):** the same `avatars` table serves both. Public catalog
  routes (root `routes/api.php`) return the camelCase catalog Resource filtered to
  `status = published AND visibility = public`. The studio/admin surface is a normal
  transactional module (own `Routes/api.php`) returning the snake_case Resource with the
  full field set. Both are documented in `contracts/avatar.md`.

## Module → web placement
| Module | Web route group | Why |
|--------|----------------|-----|
| plans | `(site)` /pricing | public marketing |
| avatars (catalog) | `(site)` /avatars, /avatars/[slug] | public browse + SEO |
| avatars (studio) | `(app)` /studio | authed creator workspace |
| conversations | `(app)` /talk/[avatarSlug], /conversations | authed core experience |
| voices | `(app)` (voice picker inside /studio) | authed studio dependency |
| usage | `(app)` /settings + `(admin)` /admin | own usage + platform dashboard |

## Build order
Dependency order (core/simple first). The vertical slice is **plans** — smallest module,
proves the whole pipeline (migration → seeder → catalog Resource → public route → web
mock ↔ api swap) before the rest.

1. **plans** ← vertical slice (catalog, seeded, read-only)
2. **voices** (avatars depend on `voice_id`)
3. **avatars** (dual-casing; depends on voices)
4. **conversations** (depends on avatars; realtime-token + internal append)
5. **usage** (depends on conversations; written on conversation-end event)
