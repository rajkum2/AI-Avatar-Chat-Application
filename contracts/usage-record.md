# Data contract — UsageRecord  (module: usage)

> THE source of truth for this entity. One row written per conversation-end (listener on
> `ConversationEnded`). Feeds the user's own usage view, the admin dashboard, and future
> billing enforcement — without schema change when billing launches.

- **Table:** `usage_records`
- **Resource casing:** `snake_case` (transactional; no money fields)
- **Type:** transactional (read-only over the API; rows are written by the system, never by clients)
- **Tenancy:** `product_id` (nullable) — yes. `organization_id` — **no** (single-tenant).

## Fields
| Canonical name | Type | Null? | Money/unit | Enum values | Filter? | Sort? | Notes |
|----------------|------|-------|-----------|-------------|---------|-------|-------|
| id | bigint (auto) | no | — | — | — | — | PK |
| product_id | bigint | yes | — | — | — | — | scaffold tenancy |
| user_id | bigint FK | no | — | — | ✓ (implicit: mine; admin: any) | — | who consumed |
| conversation_id | bigint FK | no | — | — | ✓ | — | source session; unique (one record per conversation) |
| avatar_id | bigint FK | no | — | — | ✓ | — | denormalized for per-avatar analytics |
| seconds_used | int | no | **seconds** | — | — | ✓ | = conversation.duration_seconds at end |
| messages_count | int | no | — | — | — | — | turns in the session |
| used_on | date | no | — | — | ✓ (from/to) | ✓ (default desc) | IST date of conversation end; monthly totals aggregate on this |
| created_at | timestamp | no | — | — | — | — | IST ISO-8601 in Resource |

## Relationships
| Relation | Kind | Target | Serializes as | Loaded via |
|----------|------|--------|---------------|-----------|
| user | belongsTo | User | nested object (admin only) | `whenLoaded('user')` |
| conversation | belongsTo | Conversation | id + title | `whenLoaded('conversation')` |
| avatar | belongsTo | Avatar | `{ slug, name }` | `whenLoaded('avatar')` |

## Per-surface projection (fill once, then generate)
| Surface | Representation | Status |
|---------|----------------|--------|
| DB column (migration) | `$table->unsignedInteger('seconds_used'); $table->date('used_on'); $table->foreignId('conversation_id')->unique();` | ✅ live |
| API Resource field | `'seconds_used' => $this->seconds_used, 'used_on' => $this->used_on->toDateString()` | ✅ live |
| TS interface (`lib/data/types/usage.ts`) | `seconds_used: number;` | 🔜 product-web |
| Mock JSON key (`public/data/usage/usage-records.json`) | `"seconds_used": 312` | 🔜 product-web |
| Kotlin `@Serializable` | `val secondsUsed: Int` | 🔜 product-mobile |
| GraphQL field | `secondsUsed: Int!` | 🔜 product-mobile |

## Filters object (drives list endpoints + the web hook)
`{ avatar_id?, from?, to?, sort? (default used_on desc), page?, limit? }` — implicitly
`user_id = me`; `admin.usage.view` may pass `user_id`. A summary endpoint
(`GET /usage/summary?month=YYYY-MM`) returns `{ total_seconds, total_conversations,
by_avatar: [...] }` for the settings page and admin dashboard. `limit` → `per_page`.

## Create/Update input (DTO + FormRequest + web Input type)
None over the public API — records are created by the `ConversationEnded` listener:

| Source field | Maps to |
|--------------|---------|
| conversation.user_id | user_id |
| conversation.id | conversation_id (unique — idempotent on event replay) |
| conversation.avatar_id | avatar_id |
| conversation.duration_seconds | seconds_used |
| conversation.message_count | messages_count |
| conversation.ended_at (IST date) | used_on |
