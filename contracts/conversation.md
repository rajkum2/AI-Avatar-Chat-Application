# Data contract — Conversation  (module: conversations)

> THE source of truth for this entity.

- **Table:** `conversations`
- **Resource casing:** `snake_case` (transactional; no money fields)
- **Type:** transactional
- **Tenancy:** `product_id` (nullable) — yes. `organization_id` — **no** (single-tenant).

## Fields
| Canonical name | Type | Null? | Money/unit | Enum values | Filter? | Sort? | Notes |
|----------------|------|-------|-----------|-------------|---------|-------|-------|
| id | bigint (auto) | no | — | — | — | — | PK |
| product_id | bigint | yes | — | — | — | — | scaffold tenancy |
| user_id | bigint FK | no | — | — | ✓ (implicit: mine) | — | owner; admin may filter by user |
| avatar_id | bigint FK | no | — | — | ✓ | — | → avatars.id |
| title | string | yes | — | — | ✓ search | — | auto-generated summary after end |
| status | string→enum | no | — | active\|ended\|failed | ✓ | — | cast to `ConversationStatus` |
| language | string | no | — | — | ✓ | — | BCP-47; defaults from avatar |
| started_at | timestamp | no | — | — | ✓ (from/to) | ✓ | session start |
| ended_at | timestamp | yes | — | — | — | — | null while active |
| duration_seconds | int | no | **seconds** | — | — | ✓ | 0 until ended; display mm:ss |
| message_count | int | no | — | — | — | — | denormalized |
| last_message_at | timestamp | yes | — | — | — | ✓ (default desc) | drives history ordering |
| created_at | timestamp | no | — | — | — | ✓ | IST ISO-8601 in Resource |

## Relationships
| Relation | Kind | Target | Serializes as | Loaded via |
|----------|------|--------|---------------|-----------|
| avatar | belongsTo | Avatar | nested object (catalog-lite: slug, name, portraitUrl → snake_case here: `{ slug, name, portrait_url }`) | `whenLoaded('avatar')` |
| user | belongsTo | User | nested object (admin only) | `whenLoaded('user')` |
| messages | hasMany | Message | paginated sub-resource, never embedded in list | own endpoint |

## Per-surface projection (fill once, then generate)
| Surface | Representation | Status |
|---------|----------------|--------|
| DB column (migration) | `$table->foreignId('avatar_id'); $table->string('status')->default('active'); $table->unsignedInteger('duration_seconds')->default(0);` | ✅ live |
| API Resource field | `'duration_seconds' => $this->duration_seconds, 'last_message_at' => ...` | ✅ live |
| TS interface (`lib/data/types/conversations.ts`) | `duration_seconds: number;` | 🔜 product-web |
| Mock JSON key (`public/data/conversations/conversations.json`) | `"duration_seconds": 312` | 🔜 product-web |
| Kotlin `@Serializable` | `val durationSeconds: Int` | 🔜 product-mobile |
| GraphQL field | `durationSeconds: Int!` | 🔜 product-mobile |

## Filters object (drives list endpoints + the web hook)
`{ avatar_id?, status?, search?, from?, to?, sort? (default last_message_at desc), page?, limit? }`
— always implicitly scoped to `user_id = me` (admin: `admin.usage.view` may pass `user_id`).
`limit` → `per_page` on the backend.

## Create/Update input (DTO + FormRequest + web Input type)
| Input field | Validation | Maps to |
|-------------|-----------|---------|
| avatar_id | required\|exists:avatars,id (published, or own) | avatar_id |
| language | nullable\|string\|max:12 | language (defaults to avatar.language) |

Lifecycle is action-driven, not update-driven:
- `POST /conversations` → creates `active` session, returns conversation.
- `POST /conversations/{id}/realtime-token` → short-lived signed token for the realtime
  media gateway (STT/LLM/TTS loop). Token scope: this conversation only.
- `POST /conversations/{id}/end` → sets `ended`, computes `duration_seconds`, triggers
  title generation + `ConversationEnded` event (→ usage module writes a UsageRecord,
  avatar stats increment).
