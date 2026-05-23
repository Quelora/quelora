# 🧩 quelora-widget-community — Overview

> Package documentation · [↑ Docs index](../README.md) · Source: `quelora-widget-community/.ai/overview.md`
> Related: [widget/cdn-release](../widget/cdn-release.md) · [widget/sentinel-agent-guide](../widget/sentinel-agent-guide.md)

## 1. What It Is

An embeddable community engagement widget written in ES6+ JavaScript. It is distributed as a set of static build artifacts (`dist/`) consumed by:
- `quelora-demo-api` (Node.js demo server)
- `quelora-wp-plugin` (WordPress plugin)
- Any integrator page via a `<script>` tag + `window.QUELORA_CONFIG`

The widget is **offline-first**, **modular**, and splits execution across three contexts: Main Thread, Web Worker, and Service Worker.

---

## 2. Repository Structure

```
quelora-widget-community/
├── js/
│   ├── quelora.js                   # Entry point — singleton bootstrap
│   ├── sw.js                        # Service Worker — push notifications + offline cache
│   ├── rollup.dir.config.js         # Build configuration
│   │
│   ├── core/                        # System foundations (12 modules)
│   │   ├── quelora-app.js           # App orchestrator — lazy-loaded on first interaction
│   │   ├── conf.js                  # Config reader (window.QUELORA_CONFIG)
│   │   ├── core.js                  # State & lifecycle (WebSocket/SSE session)
│   │   ├── event.js                 # EventBus singleton
│   │   ├── session.js               # JWT lifecycle
│   │   ├── storage.js               # LocalStorage / SessionStorage / IndexedDB abstraction
│   │   ├── security.js              # CSRF / XSS helpers
│   │   ├── guard.js                 # Access control & permission checks
│   │   ├── logs.js                  # Centralized logger (handleLog)
│   │   ├── debug.js                 # Debug harness (tree-shaken in production)
│   │   ├── utils.js                 # DOM, timing, debounce utilities
│   │   ├── i18n.js                  # Internationalization
│   │   └── scaffold.js              # DOM root element creation
│   │
│   ├── worker/                      # Web Worker context
│   │   ├── queloraWorker.js         # Worker entry — all network & data processing
│   │   ├── endpointsWorker.js       # Action → HTTP request dispatch map
│   │   ├── utilsWorker.js           # Worker-side utilities
│   │   ├── modules/notifications/
│   │   │   └── activitiesWorker.db.js  # IndexedDB activity log
│   │   └── pkg/                     # Pre-compiled WASM bindings
│   │       ├── quelora_image_processor.js / .wasm
│   │       └── quelora_markdown_parser.js / .wasm
│   │
│   ├── modules/                     # Feature modules
│   │   ├── posts/                   # Entity discovery, likes, shares, stats
│   │   ├── comments/                # Thread rendering, virtualization, nested replies
│   │   ├── profile/                 # User profiles, auth, follow system
│   │   ├── notifications/           # Push notifications, activity feed, SSE routing
│   │   ├── registration/            # Signup flow, email verification, OAuth
│   │   └── features/                # mention.js, audio.js, quote.js, live.ui.js
│   │
│   ├── ui/                          # UI layer
│   │   ├── ui.js                    # Central UI controller
│   │   ├── modal.js                 # Modal management
│   │   ├── drawer.js / drawers.js   # Side panel system
│   │   ├── toast.js                 # Toast notifications
│   │   ├── anchor.js                # Deep-link anchor protocol
│   │   ├── icons.js                 # Icon system
│   │   └── components/              # emoji.js, gif.js, cropper.js, progressInput.js
│   │
│   ├── plugins/                     # Native plugins (always loaded)
│   │   ├── sentinel/                # Debug bridge (inspector, recorder, protocol)
│   │   └── placer/                  # Interaction placement optimizer
│   │
│   ├── enterprise/                  # Optional enterprise plugins
│   │   ├── resilience/              # Offline fallback + IndexedDB persistence
│   │   ├── p2p/                     # Torrent/Nostr peer networking
│   │   ├── sse/                     # Server-sent Events client
│   │   ├── chat/                    # P2P / server-backed chat
│   │   ├── live/                    # Live streaming UI
│   │   ├── gamification/            # Points, badges, leaderboards
│   │   ├── survey/                  # Polls and feedback forms
│   │   └── banana/                  # Internal module (TBD)
│   │
│   ├── services/
│   │   ├── captcha.js               # reCAPTCHA / hCaptcha wrapper
│   │   └── geoStorage.js            # Geolocation provider integration
│   │
│   ├── vendors/                     # Pre-built third-party libs
│   │   ├── pako.min.js              # GZIP compression
│   │   ├── qrcode.min.js            # QR code generation
│   │   ├── trystero-torrent.min.js  # P2P via WebTorrent
│   │   └── trystero-nostr.min.js    # P2P via Nostr
│   │
│   └── locales/                     # i18n JSON bundles (12 languages)
│       └── {en,pt,de,es,fr,ar,he,hi,ja,ru,zh}/
│           └── {common,profile,notifications,chat,gamification,survey,live,placer}.json
│
├── css/
│   ├── quelora.css                  # Main CSS manifest (imports all modules)
│   ├── variables.css                # 80+ CSS custom properties (design tokens)
│   ├── community.css, profile.css, modal.css, animations.css, responsive.css, ...
│   └── enterprise/
│       └── gamification.css, livechat.css, survey.css, ads.css, chat/
│
└── dist/                            # Build output (consumed by integrators)
    ├── quelora.js                   # Main bundle
    ├── chunks/                      # Lazy-loaded code-split chunks
    ├── worker/
    │   ├── queloraWorker.js
    │   ├── chunks/
    │   └── pkg/                     # WASM files
    ├── sw.js
    ├── vendors/
    ├── locales/
    └── css/
```

---

## 3. Build Pipeline

**Tool:** Rollup (`js/rollup.dir.config.js`)

Two independent Rollup builds run in parallel:

| Build | Input | Output | Format |
|---|---|---|---|
| Main app | `js/quelora.js` | `dist/quelora.js` + `dist/chunks/` | ESM |
| Web Worker | `js/worker/queloraWorker.js` | `dist/worker/queloraWorker.js` + `dist/worker/chunks/` | ESM |

**Key Rollup plugins:**

| Plugin | Purpose |
|---|---|
| `@rollup/plugin-resolve` | Node module resolution |
| `@rollup/plugin-commonjs` | CJS compatibility |
| `@rollup/plugin-json` | JSON imports |
| `rollup-plugin-replace` | Env variable substitution |
| `rollup-plugin-copy` | Copy locales, CSS, WASM to `dist/` |
| `rollup-plugin-terser` | 3-pass aggressive minification (`passes: 3`, `drop_console: true`, `unsafe: true`) |
| `rollup-plugin-visualizer` | Bundle analysis → `dist/stats.html` |
| `exterminateDebug` (custom) | Tree-shakes all debug/sentinel modules out of production builds |

**Code splitting:** Dynamic imports are NOT inlined; they produce named chunks (`chunks/[name]-[hash].js`). Enterprise plugins are always lazy-loaded.

**Approximate output sizes (gzipped):**
- `quelora.js` ~200 KB
- `queloraWorker.js` ~180 KB
- WASM modules ~180 KB combined
- CSS ~80 KB, Locales (all langs) ~150 KB
- **Total ~300–350 KB**

---

## 4. Execution Contexts

```
┌─────────────────────────────────────────────────────────────────┐
│  MAIN THREAD                                                    │
│                                                                 │
│  quelora.js ──(lazy)──► quelora-app.js                         │
│      │                       │                                  │
│      │                  Plugin Registry                         │
│      │                  (enterprise/*.js)                       │
│      │                       │                                  │
│      └── core/ modules/  ui/ services/                         │
│                                                                 │
│  Communication: eventBus.emit/on  │  CoreModule.postWorkerMessage│
└─────────────────────────────────────────────────────────────────┘
           ▲ postMessage          ▼ postMessage
┌─────────────────────────────────────────────────────────────────┐
│  WEB WORKER                                                     │
│                                                                 │
│  queloraWorker.js                                               │
│      ├── endpointsWorker.js  (action → fetch dispatch)         │
│      ├── PluginRegistry      (enterprise worker plugins)        │
│      ├── WASM: image processor, markdown parser                 │
│      └── Resilience routing  (HYBRID / P2P / fallback)         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  SERVICE WORKER  (sw.js)                                        │
│  Push notifications, offline cache (ql-notifications-v3)       │
│  Deep-link routing on notification click                        │
└─────────────────────────────────────────────────────────────────┘
```

**Rule:** Any cryptographic operation, WASM execution, or API call goes to the Worker. The Main Thread only handles UI and event orchestration.

---

## 5. Startup Flow

```
DOMContentLoaded
  └─► Quelora.getInstance()  [quelora.js]
        1. Validate config (cid format: QU-XXXXXXXX-XXXXX)
        2. Instantiate Web Worker (blob or external quelora-worker.js)
        3. worker.postMessage('init', { cid, apiUrl, token, plugins, features, ... })
        4. Initialize light UI — entity DOM observer, anchor handler
        5. Register i18n (browser lang → saved pref → 'en')
        6. Register Service Worker
        7. Set up online/offline listeners
        8. Schedule lazy app load (1 s delay or on first interaction)

On first interaction:
  └─► dynamic import('core/quelora-app.js')
        1. Load enterprise UI plugins via Plugin Registry
        2. Initialize modules: Comments, Profile, Registration, Notifications
        3. Set up 40+ worker message handlers
        4. Load Emoji picker (desktop only)
        5. Initialize Captcha if enabled
        6. Wire event bus (drawers, session, resilience events)
```

---

## 6. Inter-Context Communication

### Main → Worker
```js
CoreModule.postWorkerMessage({ action: 'like', payload: { entityId } })
// Auth token and CID are auto-injected
```

### Worker → Main
```js
self.postMessage({ action: 'likeUpdated', payload: { likesCount, liked }, originalPayload })
// Caught by workerMessageHandlers in quelora-app.js
```

### Intra-Main (UI components)
```js
eventBus.emit('SHOW_TOAST', { type: 'success', message: '...' })
eventBus.on('SESSION_ESTABLISHED', callback)
```

**Registered eventBus events:**
`SESSION_ESTABLISHED`, `SESSION_CLEARED`, `SHOW_TOAST`, `DRAWER_STATE_CHANGE`,
`RESILIENCE_MODE_CHANGED`, `OFFLINE_MODE`, `ONLINE_MODE`, `SSE_MESSAGE`,
`CHAT_MESSAGE_RECEIVED`, `P2P_CONNECTION_ESTABLISHED`, `USER_BLOCKED`,
`GAMIFICATION_UI_OPENED`, and others.

---

## 7. Plugin System

Both Main Thread and Worker use the same **dynamic registry pattern**. Plugins are declared in `window.QUELORA_CONFIG.plugins` and injected at runtime. Missing plugins (Community Edition) are silently skipped — they never throw.

**Main Thread (quelora-app.js):**
```js
for (const plugin of conf.get('plugins.ui')) {
    const mod = await import(plugin.path);
    if (mod.register) await mod.register(context);  // context = { worker, eventBus, modules, ... }
}
```

**Worker (queloraWorker.js):**
```js
PluginRegistry.register(payload.plugins);  // Populates manifest on init
PluginRegistry.load('ResilienceManager', extract);  // Lazy, deduplicated
```

### Enterprise modules

| Plugin | Path | Purpose |
|---|---|---|
| Resilience | `enterprise/resilience/` | Offline fallback, mode switching, IndexedDB cache |
| P2P | `enterprise/p2p/` | Torrent/Nostr peer networking via Trystero |
| SSE | `enterprise/sse/` | Server-sent Events real-time client |
| Chat | `enterprise/chat/` | P2P and server-backed messaging |
| Live | `enterprise/live/` | Live streaming / broadcast UI |
| Gamification | `enterprise/gamification/` | Points, badges, leaderboards |
| Survey | `enterprise/survey/` | Polls and feedback forms |

---

## 8. Resilience Subsystem

Mode is set by the server via `X-Resilience-Bootstrap` response header and consumed by `queloraWorker.js` to route every request:

| Mode | Behavior |
|---|---|
| `HYBRID` | Server + P2P fallback — normal operation |
| `PASSIVE` | Existing sessions work, new logins blocked |
| `SERVER_ONLY` | P2P disabled; protected write actions blocked |
| `P2P_ONLY` | Anonymous users routed through P2P; server reserved for auth users |

**Offline layers:**
1. In-memory cache (module state)
2. IndexedDB (`fallbackWorker.db.js`, `activitiesWorker.db.js`)
3. P2P artifact ingestion (enterprise)
4. Circuit breaker in `GuardModule` / `CoreModule.isSystemOffline()` — write actions blocked with offline placeholder UI

---

## 9. WASM Modules

Located in `js/worker/pkg/` and loaded inside the Worker:

| Module | File | Function |
|---|---|---|
| Image Processor | `quelora_image_processor_bg.wasm` | Resize / compress images before upload |
| Markdown Parser | `quelora_markdown_parser_bg.wasm` | Convert user Markdown → sanitized HTML (XSS-safe) |

Both are initialized asynchronously at Worker startup via dynamic import of the JS bindings.

---

## 10. UI Layer

### Anchor (Deep-link) Protocol

```
#QUELORA-Q-{entityId}-{commentId}   → load entity + thread
#QUELORA-U-{memberId}               → open user profile
#QUELORA-E-{entityId}               → load entity
#QUELORA-L-{entityId}-{commentId}   → load entity + highlight comment
#QUELORA-R-{memberId}               → mention user
```

`anchor.js` parses the hash and delegates to `QueloraApp.handleAnchor(code, params)`.

### Drawer IDs (for Sentinel `OPEN_DRAWER`)

`ql-comments`, `ql-notification-list`, `ql-community-profile`,
`ql-community-settings`, `likes-list`, `ql-member-profile-drawer`,
`ql-search-follow-request`, `ql-follow-request`, `ql-community-general-settings`

### Components

| Component | File | Notes |
|---|---|---|
| Emoji picker | `ui/components/emoji.js` | EmojiMart — desktop only |
| GIF picker | `ui/components/gif.js` | Server-side Giphy proxy, race-condition safe |
| Image cropper | `ui/components/cropper.js` | Avatar upload, touch-friendly |
| Progress input | `ui/components/progressInput.js` | Char counter, auto-resize |

---

## 11. Styling

**Architecture:** CSS custom properties (dark-first theme)

- `css/variables.css` — 80+ design tokens (colors, spacing, typography, nesting depth colors up to `--ql-level-9`)
- `css/quelora.css` — manifest that imports all module stylesheets
- `css/responsive.css` — mobile/tablet breakpoints
- `css/animations.css` — keyframes and transitions
- `css/enterprise/` — optional overrides (gamification, livechat, survey, ads)

---

## 12. Internationalization

**12 languages:** `en`, `pt`, `de`, `es`, `fr`, `ar`, `he`, `hi`, `ja`, `ru`, `zh`

**Namespace files per language:** `common`, `profile`, `notifications`, `chat`, `gamification`, `survey`, `live`, `placer`

**Init priority:** saved user preference → browser `navigator.language` → `en`

**DOM translation pattern:**
```html
<button class="t">button_like</button>
<input placeholder="t:input_placeholder" class="ql-comment-input" />
```

Drawers lazy-load their namespace on open via `I18n.loadModuleTranslations('notifications')`.

---

## 13. Security

- **CSRF:** Token validation on all state-changing requests (`core/security.js`)
- **XSS:** User-generated content rendered only through the WASM Markdown parser — raw HTML injection is never allowed
- **Captcha:** reCAPTCHA v3 or hCaptcha, configurable per action (registration, comments)
- **Optional chaining rule:** `if (obj && obj.readyState === obj.OPEN)` — not `obj?.readyState === obj?.OPEN` (avoids silent boolean evaluation bugs)

---

## 14. Logging

All logging goes through a single function — never `console.log` directly:

```js
handleLog('User auth failed', 'AuthModule', 'error', '🔒');
```

In production builds, `handleLog` is declared pure in Terser config and all calls are dropped (`drop_console: true`, `pure_funcs: ['handleLog', ...]`).

---

## 15. Configuration Reference

The integrator sets `window.QUELORA_CONFIG` before loading `quelora.js`:

```js
window.QUELORA_CONFIG = {
    cid: 'QU-XXXXXXXX-XXXXX',          // Required — client identifier
    apiUrl: 'https://api.example.com',  // Required — backend base URL

    login: {
        queloraSession: true,           // false = external session (provide loginUrl/logoutUrl)
    },

    features: {
        comments: true, likes: true, bookmarks: true,
        share: true, follow: true, audio: true, gifs: true, mentions: true
    },

    plugins: {
        ui: [
            { name: 'Gamification', path: './js/enterprise/gamification/gamification.js' }
        ],
        worker: [
            { name: 'ResilienceManager', path: './js/enterprise/resilience/worker/resilienceManager.js' }
        ]
    },

    captcha: { enabled: false, provider: 'recaptcha', siteKey: '' },
    geolocation: { enabled: false, provider: 'ip-api', apiKey: '' },

    entityConfig: {
        selector: '[data-entity]',      // DOM selector for content entities
        goTo: false,
        interactionPlacement: { position: 'inside', relativeTo: '[data-interactions]' }
    }
};
```

---

## 16. Key `context` Object (accessible via Sentinel `EXEC_JS`)

| Object | Notable members |
|---|---|
| `context.CoreModule` | `postWorkerMessage`, `disconnectWebSocket`, `isSystemOffline` |
| `context.ConfModule` | `get(key, default)` |
| `context.SessionModule` | `getToken`, `getTokenIfAvailable`, `performLogin`, `logout` |
| `context.PostsModule` | `setLike`, `handleShare`, `handleBookmark`, `loadThread`, `fetchStats` |
| `context.ProfileModule` | `getCurrentProfile`, `logout`, `saveMyProfile`, `updateSetting` |
| `context.eventBus` | `emit(event, payload)`, `on(event, cb)`, `off(event, cb)` |
| `context.worker` | Direct Worker reference |
| `window.QueloraApp` | `handleAnchor(code, params)`, `Modules` (full registry) |
