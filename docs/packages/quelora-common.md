# @quelora/common — Developer Overview

> Package documentation · [↑ Docs index](../README.md) · Source: `quelora-common/.ai/overview.md`

**Type:** Shared internal library (npm workspace package)
**Consumers:** `quelora-public-api`, `quelora-dashboard-api`, `quelora-worker`, `quelora-jobs`
**Import pattern:** Direct path imports — `require('@quelora/common/models/Profile')`, `require('@quelora/common/services/authService')`, etc. No barrel `index.js`.

---

## Directory Tree

```
quelora-common/
├── db.js                           # MongoDB singleton connection
├── package.json
│
├── models/                         # 30 Mongoose schemas
│   ├── Profile.js                  # Core user entity
│   ├── Post.js                     # Discussion thread / content node
│   ├── Comment.js                  # Comment on a post
│   ├── Activity.js                 # User action log (TTL 30 days)
│   ├── Client.js                   # Tenant configuration document
│   ├── Report.js                   # Moderation report
│   ├── ReputationConfig.js         # Per-client reputation weights
│   ├── ReputationLog.js            # Reputation change audit trail
│   ├── JobExecutionLog.js          # Background job run records
│   ├── Stats.js                    # Hourly/daily aggregate counters
│   ├── PostStats.js                # Per-post counters
│   ├── GeoStats.js                 # Geographic aggregate counters
│   ├── GeoPostStats.js             # Post-level geo counters
│   ├── ProfileStats.js             # Per-profile aggregate counters
│   ├── ProfileStatsDaily.js        # Daily profile snapshots
│   ├── ProfileFollower.js          # Follow relationship (follower side)
│   ├── ProfileFollowing.js         # Follow relationship (following side)
│   ├── ProfileFollowRequest.js     # Pending follow requests
│   ├── ProfileLike.js              # Like relationship
│   ├── ProfileShare.js             # Share relationship
│   ├── ProfileBookmark.js          # Bookmark relationship
│   ├── ProfileComment.js           # Comment relationship record
│   ├── ProfileBlock.js             # Block relationship
│   ├── ProfileNotInterested.js     # "Not interested" signal
│   ├── ProfileSuggestion.js        # Suggested profile record
│   ├── CommentAudio.js             # Audio transcription for comment
│   └── TokenUsageStats.js          # AI API token consumption
│
├── services/                       # 27 business logic services
│   ├── authService.js
│   ├── cacheService.js
│   ├── clientConfigService.js
│   ├── emailService.js
│   ├── pushService.js
│   ├── reputationService.js
│   ├── reputationProcessorService.js
│   ├── userEventService.js
│   ├── profileService.js
│   ├── statsService.js
│   ├── statsRollupService.js
│   ├── activityService.js
│   ├── activityProcessorService.js
│   ├── notificationAggregatorService.js
│   ├── onboardingService.js
│   ├── suggestService.js
│   ├── activeUsersService.js
│   ├── captchaService.js
│   ├── toxicityService.js
│   ├── moderateService.js
│   ├── commentProcessingService.js
│   ├── contentQualityService.js
│   ├── geoService.js
│   ├── languageService.js
│   ├── i18nService.js
│   ├── translateService.js
│   ├── ssoService.js
│   ├── loggerService.js
│   └── tokenUsageService.js
│
├── middlewares/                    # 11 Express middlewares
│   ├── authMiddleware.js
│   ├── optionalAuthMiddleware.js
│   ├── rateLimiterMiddleware.js
│   ├── captchaMiddleware.js
│   ├── extractGeoDataMiddleware.js
│   ├── validateClientHeaderMiddleware.js
│   ├── validatePasswordResetTokenMiddleware.js
│   ├── globalErrorHandler.js
│   ├── cacheInvalidator.js
│   ├── requestLogger.js
│   ├── responseCompressor.js
│   └── trackUserPresence.js
│
├── utils/                          # 17 utility modules
│   ├── cipher.js                   # AES-256-CBC encryption
│   ├── password.js                 # Password strength validation
│   ├── textUtils.js                # Text transformations
│   ├── notificationUtils.js        # Notification dispatch helpers
│   ├── profileUtils.js
│   ├── geoUtils.js
│   ├── rankingUtils.js
│   ├── imageHelper.js
│   ├── formatComment.js
│   ├── recordProfileActivity.js
│   ├── recordStatsActivity.js
│   ├── deepMerge.js
│   ├── pluginRegistry.js           # Plugin path constructor + module registry
│   └── featureLoader.js            # Load optional enterprise module
│
├── moderationProviders/            # LLM moderation adapters
│   ├── ModerationProvider.js       # Abstract base class
│   ├── OpenAIModerationProvider.js
│   ├── GeminiModerationProvider.js
│   ├── GrokModerationProvider.js
│   └── DeepSeekModerationProvider.js
│
├── toxicityProviders/              # Toxicity scoring adapters
│   ├── ToxicityProvider.js         # Abstract base class
│   ├── PerspectiveToxicityProvider.js   # Google Perspective API
│   └── DetoxifiToxicityProvider.js
│
├── ssoProviders/                   # External auth providers
│   ├── GoogleProvider.js
│   ├── FacebookProvider.js
│   ├── AppleProvider.js
│   ├── XProvider.js
│   └── QueloraProvider.js          # Internal SSO
│
├── config/
│   ├── helmetConfig.js             # Security headers (HSTS, MIME, XSS…)
│   ├── dynamicCorsConfig.js
│   └── corsClientConfig.js
│
├── infrastructure/
│   └── bullmq.js                   # BullMQ queue/worker factory
│
├── constants/
│   └── queues.js                   # Queue name registry
│
├── templates/emails/
│   ├── notificationTemplate.js
│   └── verificationTemplate.js
│
└── locale/                         # i18n strings (12 languages)
    └── {en,es,fr,de,it,pt,ru,ar,he,hi,ja,zh}.json
```

---

## Core Infrastructure

### Database — `db.js`

Mongoose singleton. Prevents multiple connections across hot reloads.

```js
const { connectDB } = require('@quelora/common/db');
await connectDB();
```

- `connectTimeoutMS: 10000`, `serverSelectionTimeoutMS: 5000`
- Auto-index disabled in production
- Graceful shutdown on `SIGINT` / `SIGTERM`

---

### Redis — `services/cacheService.js`

IORedis client with exponential retry (`Math.min(attempts * 50, 2000)` ms).

```js
const { cacheService, cacheClient } = require('@quelora/common/services/cacheService');
```

| Method | Signature | Notes |
|--------|-----------|-------|
| `get` | `(key)` | JSON.parse on hit; returns `null` on miss |
| `set` | `(key, data, ttl?)` | JSON.stringify; optional EX seconds |
| `delete` | `(key)` | DEL |
| `deleteByPattern` | `(pattern)` | SCAN + DEL loop |
| `flush` | `()` | FLUSHALL |
| `increment` | `(key, ttl?)` | Atomic INCR; sets EX on first call |

`cacheClient` — raw IORedis instance (use for hashes, sets, lists via `hGet`, `sAdd`, `lPush`, etc.)

---

### BullMQ — `infrastructure/bullmq.js`

Singleton producer connection shared across all queues. Workers get dedicated connections (required for blocking commands).

```js
const { createQueue, createWorker } = require('@quelora/common/infrastructure/bullmq');
const { QUEUES } = require('@quelora/common/constants/queues');

const q = createQueue(QUEUES.EMAILS);
const w = createWorker(QUEUES.EMAILS, async (job) => { /* ... */ });
```

**Default job options:**
- `removeOnComplete: 100` — keep last 100 completed
- `removeOnFail: 500`
- Retry: 3 attempts, exponential backoff `1s * 2^attempt`

**Queue names (`QUEUES`):**

| Constant | Queue name | Consumer |
|----------|-----------|---------|
| `REPUTATION` | `reputation-jobs` | quelora-jobs |
| `SUGGESTION` | `suggestion-jobs` | quelora-jobs |
| `SYSTEM` | `system-jobs` | quelora-jobs |
| `ENTERPRISE` | `enterprise-jobs` | quelora-jobs |
| `ACTIVITY` | `activity-jobs` | quelora-worker |
| `GRAVITY_DECAY` | `gravity-decay` | quelora-jobs |
| `EMAILS` | `emails` | quelora-worker |
| `NOTIFICATIONS` | `notifications` | quelora-worker |
| `AGGREGATION` | `aggregation` | quelora-worker |

---

## Models

### `Profile` — core user entity

**Key fields:**

| Field | Type | Notes |
|-------|------|-------|
| `cid` | String | Tenant ID, required, indexed |
| `author` | String | SHA-256(email) — stable unique identity |
| `name` | String | 3–15 chars, auto-generated, unique per CID |
| `given_name`, `family_name` | String | Display names |
| `email` | String | Unique per CID |
| `password` | String | bcrypt(cost=10), hashed pre-save |
| `picture`, `background` | String | Avatar / banner URLs |
| `locale` | String | Language preference |
| `trust.score` | Number | Reputation score (indexed) |
| `trust.level` | Number | Trust tier 0–5 |
| `isBanned`, `isDeleted` | Boolean | Moderation flags (indexed) |
| `isVerified` | Boolean | |
| `location` | GeoJSON Point | `{ type:'Point', coordinates:[lon,lat] }` + country/city strings |
| `geohash` | String | Precision-6 geohash (≈1.2 km), indexed |
| `geoHistory` | Array | Last 5 locations with timestamps |
| `lastActivityDate` | Date | Indexed for activity sorting |
| `pushSubscriptions` | Array(max 3) | `{ endpoint, keys:{p256dh,auth}, platform, permissionGranted }` |
| `settings` | Object | notifications, privacy, interface, session |

**Counter fields** (denormalized, maintained by post-save hooks on relation models):
`bookmarksCount`, `commentsCount`, `followersCount`, `followingCount`, `blockedCount`, `likesCount`, `sharesCount`

**Indexes:** `{ email,cid }`, `{ author,cid }` (unique), `{ location:'2dsphere' }`, `{ cid,followersCount:-1 }`, `{ cid,geohash,followersCount:-1 }`, text index on name/given_name/family_name/author

**Pre-save hooks:** Generate `author` (SHA-256 of email), unique `name`, bcrypt password, compute geohash, rotate `geoHistory` (max 5), cap `pushSubscriptions` (max 3)

**Static methods:**
- `ensureProfileExists(user, cid, geoData, options)` — find-or-create with optional geo update
- `updateSettings(cid, author, key, value)` — validated single-setting update
- `updateProfileLocation(cid, author, geoData)` — atomic geo update + history rotation
- `addPushSubscription(cid, author, subscription)` — push + auto-trim to 3
- Keyset-paginated list methods: `getMoreLikes`, `getMoreComments`, `getMoreShares`, `getMoreBookmarks`, `getMoreFollowers`, `getMoreFollowing`

---

### `Post` — discussion thread / content node

**Key fields:**

| Field | Type | Notes |
|-------|------|-------|
| `cid` | String | Tenant ID |
| `entity` | ObjectId | External entity reference (unique per CID) |
| `reference` | String | External string reference (500 chars) |
| `title`, `link`, `type`, `description`, `image` | String | Content metadata |
| `commentCount`, `sharesCount`, `likesCount`, `viewsCount` | Number | Denormalized counters |
| `config` | Mixed | Full configuration object (validated pre-save) |
| `deletion.status` | String | `active` \| `trash` \| `purged` |
| `deletion.scheduledPurgeAt` | Date | TTL 60 days after trash |

**Config sub-object structure** (validated by `validateConfig()`):

| Section | Key settings |
|---------|-------------|
| `visibility` | `public` \| `private` \| `followers` |
| `comment_status` | `open` \| `closed` |
| `interaction` | booleans: comments, likes, shares, replies, bookmarks, quotes |
| `moderation` | toxicity filter, content moderation, banned words, custom prompt |
| `limits` | post_text(1000), comment_text(200), reply_text(200) |
| `audio` | mic_transcription, max_recording_seconds, bitrate |
| `liveMode` | isLiveActive, startTime, maxClients(300) |
| `editing` | allow_delete, allow_edits, edit_time_limit |
| `expiration` | enable, expire_at |
| `language` | post_language, auto_translate |

**Static methods:** `addShare`, `moveToTrash`, `restoreFromTrash`, `incrementComment`, `decrementComment`, `getDefaultConfig`

---

### `Comment` — individual comment

**Key fields:** `post` (ref Post), `entity` (ObjectId), `profile_id` (ref Profile), `author`, `text`, `parent`, `root` (thread structure), `language`, `visible` (soft-hide), `ranking_score`, `translates[]`, `hasAudio`, `trust_snapshot`

**Static methods:** `incrementLikes`, `decrementLikes`, `incrementReplies`, `decrementReplies`, `hide(id)`, `unhide(id)` (also updates parent `repliesCount`)

**Indexes:** `{ post,parent,_id:-1 }` (thread fetch), `{ ranking_score:-1,_id:-1 }`, text on `text`

---

### `Activity` — user action log

Auto-expires after **30 days** (MongoDB TTL index on `created_at`).

`action_type` enum: `like`, `comment`, `reply`, `share`, `follower`, `follower-request`, `follow-approval`

---

### `Client` — tenant configuration

**Key fields:** `cid` (unique), `users[]` (refs), `description`, `apiUrl`, `siteUrl`, `config` (Mixed — main tenant config), `postConfig`, `vapid`, `email`, `turn`, `nostr`, `p2p`, `resilience`, `jobsConfig`, `enterpriseModules` (String[]), `communityPlugins` (String[])

- `enterpriseModules` — array of enabled enterprise module keys (`surveys`, `gamification`, `advertising`, `network`, `resilience`, `push`, `liveMode`). Managed by god users per client via `PATCH /client/:cid/modules`.
- `communityPlugins` — array of enabled community plugin keys (`sentinel`, `placer`). Managed by admin+ users per client.

**`entityConfig` sub-object** (validated by `validateEntityConfig(entityConfig)`):

| Field | Type | Notes |
|-------|------|-------|
| `interactionPlacement.deterministic` | Boolean | When `true`, enables deterministic placement mode (see below) |
| `selector` | String (max 100) | CSS selector for content entities — **required** in standard mode, **omitted** in deterministic mode |
| `entityIdAttribute` | String (max 100) | Attribute holding the entity's unique ID — **required** in standard mode, **omitted** in deterministic mode |
| `interactionPlacement.position` | `'before'` \| `'after'` \| `'inside'` | **Required** in standard mode, **omitted** in deterministic mode |
| `interactionPlacement.relativeTo` | String (max 100) | Optional sub-selector — standard mode only |
| `goTo` | Boolean | Whether clicking the bar navigates to the entity's URL — present in both modes |
| `hrefAttribute` | String (max 100) | Attribute holding the entity URL — standard mode only; deterministic mode uses `data-href` on the span |

In **deterministic mode** (`deterministic: true`) the admin places `<span class="ql-deterministic" data-entity="..." data-href="...">` markers in the page HTML. The widget discovers these spans, hides them, and injects the interaction bar immediately before each one.

**Methods:**
- `validateEntityConfig(entityConfig)` — throws descriptive errors on invalid fields; relaxes selector/entityIdAttribute/position/relativeTo validation when `deterministic: true`
- `decryptConf()` — decrypts cipher subfields (`login.providerDetails.clientSecretCipher`, `moderation.apiKeyCipher`, `toxicity.apiKeyCipher`, `geolocation.apiKeyCipher`) and returns plaintext config
- `invalidateClientCache(cid)` — static, clears all Redis cache keys for the client including `:modules`

---

### Interaction models (all follow same pattern)

`ProfileFollower`, `ProfileFollowing`, `ProfileLike`, `ProfileShare`, `ProfileBookmark`, `ProfileComment`, `ProfileBlock`, `ProfileFollowRequest`, `ProfileNotInterested`, `ProfileSuggestion`

All have **post-save / post-remove hooks** that atomically update the corresponding counter on `Profile` (e.g. `ProfileFollower` saves → `Profile.followersCount++`).

---

## Services

### `authService`

```js
const authService = require('@quelora/common/services/authService');
```

| Function | Signature | Notes |
|----------|-----------|-------|
| `generateToken` | `(userId, author, clientIp, isAdmin, cid)` | Tenant users use client's JWT secret; admins use `JWT_ADMIN_SECRET` |
| `validateToken` | `(token, clientIp, cid, isAdmin)` | Returns decoded payload or `{}` — never throws |
| `renewAdminToken` | `(expiredToken, clientIp)` | Only for admins within 5 min after expiry |

---

### `clientConfigService`

```js
const clientConfigService = require('@quelora/common/services/clientConfigService');
```

All methods cache to Redis with TTL 3600s. Cache keys: `client:config:{cid}:{suffix}`.

| Function | Returns |
|----------|---------|
| `getClientCached(cid)` | Full Client document |
| `getClientConfig(cid, path?)` | Decrypted main config (dot-path: `'login.jwtSecret'`) |
| `getClientPostConfig(cid, path?)` | Post config |
| `getClientVapidConfig(cid)` | Decrypted VAPID keys |
| `getClientEmailConfig(cid)` | Decrypted email config |
| `getClientTurnConfig(cid)` | Decrypted TURN credentials |
| `getClientNostrConfig(cid)` | Decrypted Nostr config |
| `getClientP2pConfig(cid)` | Decrypted P2P config |
| `getClientResilienceConfig(cid)` | Decrypted resilience config |
| `getClientGiphyConfig(cid)` | Decrypted Giphy config |
| `getClientWidgetConfig(cid)` | Public widget config (login, captcha, geolocation, features, plugins manifest) — cached under `:widget` key |

`getClientWidgetConfig` calls `buildPluginManifest(client.enterpriseModules, client.communityPlugins)` to build the `features` and `plugins` sections of the widget config. Cache key: `client:config:{cid}:widget`.

Cache key `KEYS.MODULES(cid)` = `client:config:{cid}:modules` — cleared on module changes alongside all other keys by `clearClientConfigCache`.

---

### `emailService`

```js
const { emailService, addEmailJob } = require('@quelora/common/services/emailService');
```

- `addEmailJob(cid, author, subject, body, to, options)` — enqueue to `QUEUES.EMAILS`
- `sendEmailNotification(cid, recipientAuthor, subjectKey, messageKey, data, options)` — localized, HTML-templated, fire-and-forget
- `sendEmailBroadcastToFollowers(cid, author, subjectKey, messageKey, data, options)` — enqueues with type `broadcast_followers`

---

### `reputationService`

```js
const { queueReputationEvent } = require('@quelora/common/services/reputationService');
```

- `queueReputationEvent({ cid, target_profile_id, source_profile_id, event_type, entity_id, source_trust_level, quality_score })`
  - Pushes to Redis list `queue:reputation:events`
  - UUID for idempotency; skips self-events

---

### `userEventService`

```js
const userEventService = require('@quelora/common/services/userEventService');
```

High-level action handlers that orchestrate notifications, reputation, activity, geo, and gamification (enterprise):

| Function | Triggers |
|----------|---------|
| `onCommentAdded(...)` | Quality score → reputation event, notification, activity log, geo stats, gamification hook |
| `onReplyAdded(...)` | Same + notifies parent comment author |
| `onCommentLiked(...)` | Reputation event, notification |
| `onCommentUnliked(...)` | Reputation event |
| `onShareAdded(...)` | Stats, notification |
| `onFollowAdded(...)` | Activity, notification |

All orchestration runs via `runBackgroundTasks(promises)` → `Promise.allSettled` (failures logged, never rethrow).

---

### `profileService`

1320-line service — most complex in the codebase.

Key functions:
- `getProfile(cid, author)` — versioned cache hit → MongoDB fallback
- `invalidateProfileCache(cid, author)` — bump version + delete cache
- `searchProfiles(cid, query, profileId, excludeBlocked, limit)` — full-text with block filter
- `hydrateUserRelations(cid, profileIds, userId)` — bulk check follower/following/block status
- `getProfileSuggestions(cid, profileId, geoEnabled)` — ranked suggestions (geo or popularity)
- `findIdsByText(cid, query)` — text index query on Profile

---

### Other services (summary)

| Service | Purpose |
|---------|---------|
| `statsService` | `recordLikeAdded/Removed`, `recordShareAdded`, `recordCommentAdded` |
| `captchaService` | reCAPTCHA Enterprise + Turnstile verification |
| `toxicityService` | Load toxicity provider from client config + `score(text, lang)` |
| `moderateService` | Load LLM moderation provider + `moderate(text)` |
| `pushService` | Web Push via web-push library |
| `notificationAggregatorService` | Bundle + deduplicate notifications |
| `activityService` | Persist activity log entries |
| `activityProcessorService` | Batch activity persistence consumer |
| `onboardingService` | Generate initial profile suggestions for new users |
| `suggestService` | Profile recommendation engine (geo + social graph weighted) |
| `reputationProcessorService` | Consume `queue:reputation:events` → apply weights → update Profile.trust |
| `activeUsersService` | Track online users via Redis |
| `geoService` | Geospatial MongoDB queries ($near, $geoWithin) |
| `languageService` | Language detection |
| `i18nService` | `getLocalizedMessage(key, locale, vars?, path?)` |
| `translateService` | Machine translation (Google Translate API) |
| `ssoService` | Dispatch to appropriate SSO provider |
| `commentProcessingService` | Comment create/update orchestration |
| `contentQualityService` | Text quality score 0.01–1.0 (length, vocabulary, structure) |
| `loggerService` | In-memory console log capture for Sentinel |
| `tokenUsageService` | Track AI API token spend per CID in Redis |

---

## Middlewares

### `authMiddleware` (required auth)

```js
app.use('/protected', require('@quelora/common/middlewares/authMiddleware'));
```

- Extracts `Authorization: Bearer <token>`
- Reads `cid` from request or `X-Client-ID` header
- Calls `authService.validateToken(token, ip, cid, false)`
- Attaches decoded payload to `req.user`
- Returns `401` if token missing/invalid; `400` if CID missing

### `optionalAuthMiddleware`

Same logic but continues on failure — `req.user` is `undefined` if not authenticated.

### `rateLimiterMiddleware`

Three exported limiters:

| Export | Window | Limit | Use |
|--------|--------|-------|-----|
| `globalRateLimiter` | 1 min | 600 req/IP | All public routes |
| `strictRateLimiter` | 2 sec | 10 req/IP | Mutation endpoints |
| `loginRateLimiter` | 5 min | 5 req/IP | `/auth/generate-token`, 2FA |

### `captchaMiddleware`

- Reads `x-captcha-token` header
- Skips if not enabled in client config
- Returns `400` if token missing when enabled; `401` if verification fails

### `extractGeoDataMiddleware`

Resolution order:
1. HTTP headers (`x-ip`, `x-country`, `x-city`, `x-lat`, `x-lon`, etc.) — set by nginx
2. MaxMind GeoLite2 database lookup
3. Fallback: "Unknown"

Attaches to request: `req.clientIp`, `req.clientCountry`, `req.clientCountryCode`, `req.clientRegion`, `req.clientRegionCode`, `req.clientCity`, `req.clientLatitude`, `req.clientLongitude`, `req.geoData`

MaxMind lookup instances are cached in-memory (load once per process). Invalidate via `extractGeoData.invalidateLookup(dbPath)`.

### `globalErrorHandler`

Mount last in Express chain:
```js
app.use(require('@quelora/common/middlewares/globalErrorHandler'));
```
Returns `{ error: message }` with the error's status code (default 500). Hides internal details in production.

### Other middlewares

| Middleware | Purpose |
|-----------|---------|
| `validateClientHeaderMiddleware` | Validates `X-Client-ID` header presence |
| `validatePasswordResetTokenMiddleware` | Validates reset token |
| `cacheInvalidator` | Purges Redis cache on POST/PUT/PATCH/DELETE |
| `requestLogger` | Request/response logging |
| `responseCompressor` | Dictionary-based payload compression |
| `trackUserPresence` | Records online status in Redis |

---

## Utilities

### `cipher.js`

AES-256-CBC via Node.js `crypto`.

```js
const { encrypt, decrypt, encryptJSON, decryptJSON, generateKeyFromString } = require('@quelora/common/utils/cipher');
```

| Function | Notes |
|----------|-------|
| `encrypt(text, key)` | Returns `"<ivHex>:<cipherHex>"` |
| `decrypt(str, key)` | Reverses above |
| `encryptJSON(obj, key)` | `JSON.stringify` → `encrypt` |
| `decryptJSON(str, key)` | `decrypt` → `JSON.parse` |
| `generateKeyFromString(str)` | SHA-256 → 64-char hex (32 bytes) |
| `validateEncryptionKey(key)` | Must be 64-char hex |

IV is 16 random bytes per call — ciphertext is non-deterministic.

### `password.js`

```js
const { validatePasswordStrength } = require('@quelora/common/utils/password');
// → { valid: boolean, message: string }
```

Requirements: min 8 chars, 1 uppercase, 1 lowercase, 1 digit, 1 special character.

### `textUtils.js`

- `toUnicodeBold(text)` — ASCII → Unicode bold characters
- `validateSearchQuery(query)` → alphanumeric + accents, max 15 chars
- `decodeHtmlEntities(text)` → unescape HTML entities

### `notificationUtils.js`

- `buildHashUrl(siteUrl, type, { entity, commentId, replyId, profileId })` — client-side hash routing URL
- `queueActivityLog(activityData, requiresFanout)` — enqueue for batch persistence
- `sendNotificationAndLogActivity(...)` — fan-out dispatch (>2000 followers → background worker)

### `pluginRegistry.js`

Central path constructor that maps logical module names to feature flags and plugin file paths. Used by `clientConfigService.getClientWidgetConfig` to build the complete `{ features, plugins }` manifest returned by `GET /config`.

```js
const { buildPluginManifest, VALID_ENTERPRISE_MODULES, VALID_COMMUNITY_PLUGINS } = require('@quelora/common/utils/pluginRegistry');

const { features, plugins } = buildPluginManifest(
    ['network', 'resilience'],  // enterprise modules
    ['sentinel']                // community plugins
);
// → { features: { sse: true, chat: true, p2p: true }, plugins: { ui: [...], worker: [...] } }
```

**Enterprise module registry:**

| Key | Features | UI plugins | Worker plugins |
|-----|---------|-----------|----------------|
| `surveys` | — | Survey | — |
| `gamification` | — | Gamefication | — |
| `advertising` | — | AdsModule | — |
| `network` | `sse`, `chat` | SSEService, Chat | ActivitiesWorkerDB, SSEWorker, ChatWorker |
| `resilience` | `p2p` | P2P, TrackerBridge, Resilience | ResilienceManager, ResilienceCrypto, FallbackDB |
| `push` | — | — | ActivitiesWorkerDB (shared, deduped) |
| `liveMode` | `sse` | Live, LiveService | — |

**Community plugin registry:**

| Key | Features | UI plugins |
|-----|---------|-----------|
| `sentinel` | — | Sentinel |
| `placer` | `interactionPlacer` | Placer |

Plugin deduplication: if both `network` and `push` are active, `ActivitiesWorkerDB` appears only once (first-seen wins).

`VALID_ENTERPRISE_MODULES` and `VALID_COMMUNITY_PLUGINS` are exported for API-layer validation.

---

### `featureLoader.js`

Loads optional `@quelora/enterprise` module without crashing if absent:
```js
const enterprise = require('@quelora/common/utils/featureLoader');
enterprise?.gamification?.awardXP(cid, author, event);
```

---

## Providers

### Moderation providers

All extend `ModerationProvider` base class:
```js
const provider = new OpenAIModerationProvider(apiKey, configJson, cid);
const result = await provider.moderate(text, type);
```

- `moderate(text, type)` — returns flagging result
- Base class auto-tracks token usage to Redis: `token_usage:{cid}` → `{provider}:{model}:promptTokens`, etc.

Implementations: `OpenAIModerationProvider`, `GeminiModerationProvider`, `GrokModerationProvider`, `DeepSeekModerationProvider`

### Toxicity providers

All extend `ToxicityProvider`:
```js
const result = await provider.analyze(text, language);
// → { toxicity, severe_toxicity, obscene, threat, insult, identity_attack } (all 0.0–1.0)
```

Implementations: `PerspectiveToxicityProvider`, `DetoxifiToxicityProvider`

### SSO providers

All implement `verify(credential)` → `{ status, token, expires_in, error }`:

```js
const GoogleProvider = require('@quelora/common/ssoProviders/GoogleProvider');
const p = new GoogleProvider({ googleClientId: '...' });
const result = await p.verify(idToken);
```

`author` field in JWT set to `SHA-256(sub)` — never raw external ID.

Implementations: `GoogleProvider`, `FacebookProvider`, `AppleProvider`, `XProvider`, `QueloraProvider`

---

## Config

### `helmetConfig.js`

Exports configured Helmet middleware:
```js
app.use(require('@quelora/common/config/helmetConfig'));
```

- HSTS: 1 year, includeSubDomains, preload
- Referrer: no-referrer
- CORP: same-origin
- CSP: disabled by default (configured at nginx level)
- Frameguard: disabled by default

---

## Key Patterns & Conventions

**Multi-tenancy first:** Every model includes `cid`. Every query MUST filter by `cid`. Never query without it.

**Keyset pagination:** Use `_id` or `created_at` cursors (`lastId`), not offset/skip. All `getMore*` statics on Profile follow this pattern.

**Denormalized counters:** Profile maintains `followersCount`, `likesCount`, etc. updated atomically by post-save/remove hooks on the relation models. Never compute these by counting documents.

**Fire-and-forget background tasks:** `userEventService` wraps side effects in `Promise.allSettled` — a notification failure never blocks the main response.

**Versioned cache keys:** Profile cache uses a version counter. `invalidateProfileCache` bumps the version; old keys expire naturally. Avoids cache stampede on invalidation.

**Soft deletes:** Post uses `deletion.status` (`active` / `trash` / `purged`) with a scheduled purge date. Profile uses `isDeleted` boolean. Never hard-delete without explicit intent.

**Encryption boundary:** `cipher.js` is the only encryption code. All sensitive config fields go through it. Key derivation via `generateKeyFromString(cid)` is deterministic — the same CID always produces the same AES key.

**Enterprise isolation:** `featureLoader.js` loads `@quelora/enterprise` dynamically. If the package is absent the return value is `null` — callers must guard with optional chaining.
