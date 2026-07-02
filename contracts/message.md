# Data contract — Message  (module: conversations)

> THE source of truth for this entity. One row per spoken turn (user or avatar).

- **Table:** `conversation_messages`
- **Resource casing:** `snake_case` (transactional; no money fields)
- **Type:** transactional (child of Conversation; no standalone top-level routes)
- **Tenancy:** inherited via `conversation_id` (no own `product_id` column needed beyond scaffold default; **no** `organization_id`).

## Fields
| Canonical name | Type | Null? | Money/unit | Enum values | Filter? | Sort? | Notes |
|----------------|------|-------|-----------|-------------|---------|-------|-------|
| id | bigint (auto) | no | — | — | — | — | PK |
| conversation_id | bigint FK | no | — | — | ✓ (route) | — | → conversations.id |
| sequence | int | no | — | — | — | ✓ (default asc) | turn order within conversation, unique per conversation |
| role | string→enum | no | — | user\|avatar | ✓ | — | cast to `MessageRole` |
| transcript | text | no | — | — | ✓ search | — | STT text (user) or LLM reply text (avatar) |
| audio_url | string | yes | — | — | — | — | stored TTS/recorded clip; null if not retained |
| audio_duration_ms | int | yes | **ms** | — | — | — | length of the audio clip |
| latency_ms | int | yes | **ms** | — | — | — | avatar turns only: request→first-audio; null for user turns |
| created_at | timestamp | no | — | — | — | ✓ | IST ISO-8601 in Resource |

## Relationships
| Relation | Kind | Target | Serializes as | Loaded via |
|----------|------|--------|---------------|-----------|
| conversation | belongsTo | Conversation | id only (route context) | — |

## Per-surface projection (fill once, then generate)
| Surface | Representation | Status |
|---------|----------------|--------|
| DB column (migration) | `$table->foreignId('conversation_id'); $table->unsignedInteger('sequence'); $table->string('role'); $table->text('transcript'); $table->unique(['conversation_id','sequence']);` | ✅ live |
| API Resource field | `'transcript' => $this->transcript, 'latency_ms' => $this->latency_ms` | ✅ live |
| TS interface (`lib/data/types/conversations.ts`) | `audio_duration_ms: number \| null;` | 🔜 product-web |
| Mock JSON key (`public/data/conversations/messages.json`) | `"audio_duration_ms": 4200` | 🔜 product-web |
| Kotlin `@Serializable` | `val audioDurationMs: Int?` | 🔜 product-mobile |
| GraphQL field | `audioDurationMs: Int` | 🔜 product-mobile |

## Filters object (drives list endpoints + the web hook)
`GET /conversations/{id}/messages` → `{ role?, search?, page?, limit? }`, default sort
`sequence asc`. Scoped to conversations owned by `me` (or `admin.usage.view`).
`limit` → `per_page` on the backend.

## Create/Update input (DTO + FormRequest + web Input type)
Messages are **appended only by the realtime gateway** through an internal
service-authenticated endpoint (`POST /internal/conversations/{id}/messages` using the
realtime-token / service credential) — never by the end-user client directly. Immutable
after creation (no update/delete in v1).

| Input field | Validation | Maps to |
|-------------|-----------|---------|
| role | required\|in:user,avatar | role |
| transcript | required\|string\|max:16000 | transcript |
| audio_url | nullable\|url | audio_url |
| audio_duration_ms | nullable\|integer\|min:0 | audio_duration_ms |
| latency_ms | nullable\|integer\|min:0 (avatar turns only) | latency_ms |

`sequence` is assigned server-side (max+1 per conversation); appending also bumps the
parent's `message_count` and `last_message_at`.
