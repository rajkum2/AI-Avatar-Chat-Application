# Data contract — Voice  (module: voices)

> THE source of truth for this entity. Stock voices are seeded; cloned voices are
> contract-ready but the upload/processing pipeline is deferred (see PRODUCT_SPEC §8).

- **Table:** `voices`
- **Resource casing:** `snake_case` (transactional; no money fields)
- **Type:** transactional (authed read for the studio voice picker; user CRUD only for own cloned voices — deferred)
- **Tenancy:** `product_id` (nullable) — yes. `organization_id` — **no** (single-tenant).

## Fields
| Canonical name | Type | Null? | Money/unit | Enum values | Filter? | Sort? | Notes |
|----------------|------|-------|-----------|-------------|---------|-------|-------|
| id | bigint (auto) | no | — | — | — | — | PK |
| product_id | bigint | yes | — | — | — | — | scaffold tenancy |
| name | string | no | — | — | ✓ search | ✓ | e.g. "Asha — warm, Indian English" |
| kind | string→enum | no | — | stock\|cloned | ✓ | — | cast to `VoiceKind`; cloned = deferred pipeline |
| provider | string→enum | no | — | elevenlabs\|azure\|google\|custom | ✓ | — | TTS provider adapter key |
| provider_voice_id | string | no | — | — | — | — | provider-side identifier; **never exposed in any Resource** |
| language | string | no | — | — | ✓ | — | BCP-47, e.g. `en-IN`, `hi-IN`, `te-IN` |
| gender | string→enum | no | — | male\|female\|neutral | ✓ | — | cast to `VoiceGender` |
| sample_url | string | yes | — | — | — | — | short preview clip for the picker |
| owner_user_id | bigint FK | yes | — | — | ✓ (mine) | — | null = stock; set = user's cloned voice |
| status | string→enum | no | — | active\|processing\|failed | ✓ | — | stock always `active`; cloned lifecycle |
| created_at | timestamp | no | — | — | — | ✓ (default desc) | IST ISO-8601 in Resource |

## Relationships
| Relation | Kind | Target | Serializes as | Loaded via |
|----------|------|--------|---------------|-----------|
| owner | belongsTo | User | nested object (admin only) | `whenLoaded('owner')` |
| avatars | hasMany | Avatar | count only (block delete while in use) | `withCount` |

## Per-surface projection (fill once, then generate)
| Surface | Representation | Status |
|---------|----------------|--------|
| DB column (migration) | `$table->string('kind')->default('stock'); $table->string('provider'); $table->string('provider_voice_id');` | ✅ live |
| API Resource field | `'sample_url' => $this->sample_url, 'status' => $this->status` (provider_voice_id omitted) | ✅ live |
| TS interface (`lib/data/types/voices.ts`) | `sample_url: string \| null;` | 🔜 product-web |
| Mock JSON key (`public/data/voices/voices.json`) | `"sample_url": "/audio/voices/asha-sample.mp3"` | 🔜 product-web |
| Kotlin `@Serializable` | `val sampleUrl: String?` | 🔜 product-mobile |
| GraphQL field | `sampleUrl: String` | 🔜 product-mobile |

## Filters object (drives list endpoints + the web hook)
`{ search?, kind?, language?, gender?, status?, page?, limit? }` — list returns stock
voices + own cloned voices (`owner_user_id = me OR null`). `limit` → `per_page`.

## Create/Update input (DTO + FormRequest + web Input type)
Stock voices: seeded + `admin.voices.manage` CRUD. User voice cloning is **deferred**;
the contract for it (so no schema change later):

| Input field | Validation | Maps to |
|-------------|-----------|---------|
| name | required\|string\|max:100 | name |
| language | required\|string\|max:12 | language |
| gender | required\|in:male,female,neutral | gender |
| sample_upload | required\|file\|mimes:mp3,wav\|max:10240 (deferred) | → provider clone job → provider_voice_id; status `processing` → `active`/`failed` |
