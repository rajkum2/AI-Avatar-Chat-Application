# Data contract — Plan  (module: plans)

> THE source of truth for this entity. Monetization is DEFERRED: plans are seeded and
> displayed on /pricing, limits are tracked via `usage` but NOT enforced in v1. This is
> the **vertical-slice module** (smallest; proves the pipeline).

- **Table:** `plans` — **REUSED from Core/Payments** (Phase 2 decision: the starter's
  payments module already owns `plans` with slug/name/type/billing_cycle/decimal
  price/features/limits JSON/is_active/sort_order). The plans module does NOT create
  its own table; it adds a module migration `add_price_paise_to_plans` (integer
  `price_paise`, backfilled ×100 from the legacy decimal `price`, which is kept in
  sync for Core/Payments compatibility) and projects the spec's limit fields into the
  existing `limits` JSON as `{ maxAvatars, monthlyVoiceMinutes, maxClonedVoices }`.
- **Resource casing:** `camelCase` (public catalog Resource mirroring the web mock shape) — money as `pricePaise` + `priceFormatted`
- **Type:** public catalog (seeded; `list` only) + admin manage via Core/Payments `AdminPlanController` (super-admin gated)
- **Tenancy:** `product_id` (nullable) — yes. `organization_id` — **no** (single-tenant).

## Fields
| Canonical name | Type | Null? | Money/unit | Enum values | Filter? | Sort? | Notes |
|----------------|------|-------|-----------|-------------|---------|-------|-------|
| id | bigint (auto) | no | — | — | — | — | PK |
| product_id | bigint | yes | — | — | — | — | scaffold tenancy |
| slug | string | no | — | — | ✓ (by) | — | unique: free\|plus\|pro |
| name | string | no | — | — | — | — | display name |
| price_paise | bigint | no | **paise** | — | — | ✓ | ADDED by module migration; 0 for Free; legacy decimal `price` kept in sync (÷100) for Core/Payments |
| billing_cycle | string | no | — | monthly\|quarterly\|yearly\|one_time | ✓ | — | existing Core/Payments column (spec uses monthly) |
| features | json (string[]) | yes | — | — | — | — | marketing bullet list (existing column) |
| limits | json | yes | — | — | — | — | `{ maxAvatars: int, monthlyVoiceMinutes: int, maxClonedVoices: int }` — spec limits, unenforced v1 (existing column) |
| is_active | bool | no | — | — | ✓ | — | Plus/Pro seeded `false` until billing launches |
| sort_order | int | no | — | — | — | ✓ (default asc) | pricing-page order |
| created_at | timestamp | no | — | — | — | — | IST ISO-8601 in Resource |

## Relationships
| Relation | Kind | Target | Serializes as | Loaded via |
|----------|------|--------|---------------|-----------|
| — | — | — | none in v1 (no subscriptions until billing) | — |

## Per-surface projection (fill once, then generate)
| Surface | Representation | Status |
|---------|----------------|--------|
| DB column (migration) | `$table->unsignedBigInteger('price_paise')->default(0); $table->json('features');` | ✅ live |
| API Resource field (catalog) | `'pricePaise' => $this->price_paise, 'priceFormatted' => formatInr($this->price_paise), 'limits' => ['maxAvatars' => $this->max_avatars, 'monthlyVoiceMinutes' => $this->monthly_voice_minutes, 'maxClonedVoices' => $this->max_cloned_voices]` | ✅ live |
| TS interface (`lib/data/types/plans.ts`) | `pricePaise: number; limits: { maxAvatars: number; … }` | 🔜 product-web |
| Mock JSON key (`public/data/plans/plans.json`) | `"pricePaise": 49900` | 🔜 product-web |
| Kotlin `@Serializable` | `val pricePaise: Long` | 🔜 product-mobile |
| GraphQL field | `pricePaise: Int!` | 🔜 product-mobile |

## Filters object (drives list endpoints + the web hook)
Public `GET /plans` → `{ active? }`. Default returns **all** plans (the /pricing page
renders inactive tiers as "coming soon"); pass `active=1` to restrict to purchasable
tiers. Sorted `sort_order asc`. No pagination (≤ a handful of rows).

## Create/Update input (DTO + FormRequest + web Input type)
Admin-only (`admin.plans.manage`); seeded rows per PRODUCT_SPEC §6.

| Input field | Validation | Maps to |
|-------------|-----------|---------|
| name | required\|string\|max:60 | name |
| slug | required\|alpha_dash\|unique per product | slug |
| price_paise | required\|integer\|min:0 | price_paise (+ keeps decimal `price` = price_paise/100 in sync) |
| billing_cycle | required\|in:monthly,quarterly,yearly,one_time | billing_cycle |
| features | sometimes\|array of string | features |
| limits | sometimes\|array (maxAvatars, monthlyVoiceMinutes, maxClonedVoices ints) | limits JSON |
| is_active | required\|boolean | is_active |
| sort_order | required\|integer | sort_order |
