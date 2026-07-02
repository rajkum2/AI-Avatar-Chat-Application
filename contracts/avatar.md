# Data contract — Avatar  (module: avatars)

> THE source of truth for this entity. Every surface is generated from / checked
> against this table.

- **Table:** `avatars`
- **Resource casing:** **DUAL** — public catalog Resource = `camelCase`; studio/admin
  Resource = `snake_case` (no money fields on this entity, so no `_formatted`). See
  "Dual-casing" section below for which route serves which.
- **Type:** public catalog **+** transactional (studio CRUD, admin moderation)
- **Tenancy:** `product_id` (nullable) — yes. `organization_id` — **no** (single-tenant).

## Fields
| Canonical name | Type | Null? | Money/unit | Enum values | Filter? | Sort? | Notes |
|----------------|------|-------|-----------|-------------|---------|-------|-------|
| id | bigint (auto) | no | — | — | — | — | PK |
| product_id | bigint | yes | — | — | — | — | scaffold tenancy |
| slug | string | no | — | — | ✓ (by) | — | unique natural key |
| name | string | no | — | — | ✓ search | ✓ | |
| tagline | string | yes | — | — | — | — | one-liner for cards |
| description | text | yes | — | — | ✓ search | — | |
| category | string→enum | no | — | companion\|tutor\|coach\|assistant\|storyteller | ✓ | — | cast to `AvatarCategory` |
| personality_prompt | text | no | — | — | — | — | LLM system prompt. **NEVER in public catalog Resource** |
| greeting_text | string | yes | — | — | — | — | first line avatar speaks on session start |
| voice_id | bigint FK | no | — | — | ✓ | — | → voices.id |
| language | string | no | — | — | ✓ | — | BCP-47, default `en-IN` |
| visual_style | string→enum | no | — | realistic\|stylized\|cartoon | ✓ | — | cast to `AvatarVisualStyle` |
| portrait_url | string | yes | — | — | — | — | card/detail image |
| visibility | string→enum | no | — | public\|private | ✓ | — | private = creator-only |
| status | string→enum | no | — | draft\|pending_review\|published\|archived | ✓ | — | cast to `AvatarStatus`; public catalog serves `published` only |
| creator_user_id | bigint FK | yes | — | — | ✓ (mine) | — | null = official/staff-made |
| is_featured | bool | no | — | — | ✓ | ✓ | staff-curated |
| conversation_count | int | no | — | — | — | ✓ (popular) | denormalized stat |
| total_seconds_talked | bigint | no | seconds | — | — | ✓ | denormalized stat |
| created_at | timestamp | no | — | — | — | ✓ (default desc) | IST ISO-8601 in Resource |

## Relationships
| Relation | Kind | Target | Serializes as | Loaded via |
|----------|------|--------|---------------|-----------|
| voice | belongsTo | Voice | nested object (catalog: `voice: { name, sampleUrl, gender, language }`) | `whenLoaded('voice')` |
| creator | belongsTo | User | nested object (studio/admin only) | `whenLoaded('creator')` |
| conversations | hasMany | Conversation | count only (never embedded) | `withCount` |

## Dual-casing (PropYaar-Projects pattern)
| Surface | Route group | Resource | Field set |
|---------|------------|----------|-----------|
| Public catalog | public group, root `routes/api.php` (`GET /avatars`, `GET /avatars/{slug}`, `GET /avatars/{slug}/similar`) | `AvatarCatalogResource` (camelCase) | slug, name, tagline, description, category, language, visualStyle, portraitUrl, isFeatured, conversationCount, greetingText, voice{…} — filtered to `status=published AND visibility=public`; **excludes** personality_prompt, creator_user_id, status, visibility |
| Studio (owner CRUD) | avatars module `Routes/api.php`, authed | `AvatarResource` (snake_case) | full field set incl. personality_prompt |
| Admin moderation | `AdminAvatarController` + `guard('admin.avatars.moderate')` | `AvatarResource` (snake_case) | full field set + creator |

## Per-surface projection (fill once, then generate)
| Surface | Representation | Status |
|---------|----------------|--------|
| DB column (migration) | `$table->string('slug')->unique(); $table->text('personality_prompt'); $table->foreignId('voice_id'); $table->string('status')->default('draft');` | ✅ live |
| API Resource field (catalog) | `'portraitUrl' => $this->portrait_url, 'conversationCount' => $this->conversation_count` | ✅ live |
| API Resource field (studio) | `'personality_prompt' => $this->personality_prompt, 'status' => $this->status` | ✅ live |
| TS interface (`lib/data/types/avatars.ts`) | `Avatar` (camelCase, catalog) + `StudioAvatar` (snake_case) | 🔜 product-web |
| Mock JSON key (`public/data/avatars/avatars.json`) | `"portraitUrl": "/images/avatars/asha.png"` (catalog shape) | 🔜 product-web |
| Kotlin `@Serializable` | `val portraitUrl: String?` (catalog shape) | 🔜 product-mobile |
| GraphQL field | `portraitUrl: String` | 🔜 product-mobile |

## Filters object (drives list endpoints + the web hook)
- Catalog: `{ search?, category?, language?, featured?, sort? ('popular'|'newest'), page?, limit? }`
- Studio: `{ search?, status?, visibility?, sort?, page?, limit? }` (implicitly scoped to `creator_user_id = me`)
- `limit` → `per_page` on the backend.

## Create/Update input (DTO + FormRequest + web Input type)
| Input field | Validation | Maps to |
|-------------|-----------|---------|
| name | required\|string\|max:100 | name (slug auto-generated, uniquified) |
| tagline | nullable\|string\|max:160 | tagline |
| description | nullable\|string\|max:2000 | description |
| category | required\|in:companion,tutor,coach,assistant,storyteller | category |
| personality_prompt | required\|string\|max:8000 | personality_prompt |
| greeting_text | nullable\|string\|max:300 | greeting_text |
| voice_id | required\|exists:voices,id (stock or own) | voice_id |
| language | required\|string\|max:12 | language |
| visual_style | required\|in:realistic,stylized,cartoon | visual_style |
| portrait_url | nullable\|url | portrait_url |
| visibility | required\|in:public,private | visibility |

Status transitions are **actions**, not update fields: submit-for-review
(`draft → pending_review`, owner; auto-`published` if `visibility=private`), approve/reject
(`pending_review → published`/`draft`, staff), archive (owner/staff).
