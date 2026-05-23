# Quelora

> **Open-source, self-hosted comments and community engagement.**
> A privacy-first alternative to Disqus — with AI moderation, real-time chat,
> gamification, offline-first resilience, and a WordPress plugin.

[![License: AGPL-3.0](https://img.shields.io/badge/License-AGPL--3.0-blue.svg)](./LICENSE)
[![Status: first public release](https://img.shields.io/badge/status-first%20public%20release-orange.svg)](#status--honest)

**[Website](https://quelora.org)** · **[Live demo](https://demo.quelora.org)** · **[Docs](./docs)** · **[Discussions](https://github.com/Quelora/quelora/discussions)**

---

## What Quelora is

Quelora is not "yet another Disqus alternative". It is a full **engagement
layer** you drop on any website or web app — comments, profiles, follows,
real-time chat, push notifications, AI-moderated discussions, gamification,
surveys, P2P resilience.

Embed it with a single `<script>` tag. Self-host it on a small VPS. Own the
data. No tracker, no ads, no third party.

### Highlights

- 💬 **Discussions** — threads, replies, reactions, mentions, GIFs, audio.
- 👤 **Community layer** — user profiles, follows, blocks, reputation / trust scores.
- 🤖 **AI moderation** — automatic toxicity scoring (Perspective · Detoxify) and LLM moderation (OpenAI · Gemini · Grok · DeepSeek).
- ⚡ **Real-time** — live comment streams, chat, live-mode broadcast threads.
- 🎮 **Gamification & surveys** — points, badges, levels, polls.
- 🌐 **Offline-first** — IndexedDB cache, optional P2P resilience via WebTorrent / Nostr.
- 🔌 **WordPress plugin** — drop-in, no code.
- 🌍 **12 languages** — automatic detection (en, es, fr, de, it, pt, ru, zh, ja, ar, hi, he).
- 🔐 **Privacy-first** — per-tenant configuration encrypted at rest (AES-256-CBC), JWT auth, optional 2FA.

## Try it in 5 minutes

```bash
git clone https://github.com/Quelora/quelora-dx-env
cd quelora-dx-env
make setup     # ~5 min: clones every component, SSL via mkcert, DNS, deps, DB seed
make up

# Then open:
#   https://dashboard.quelora.dev   →   admin SPA (default: quelora / Quelora26*/ — change it)
#   https://demo.quelora.dev        →   the widget live
```

For production deployments, see [docs/architecture.md](./docs/architecture.md)
and the [`quelora-dx-env`](https://github.com/Quelora/quelora-dx-env) repo —
its Makefile is currently the canonical source of truth while a production
self-hosting guide is being written.

## Embed the widget on any site

```html
<script>
  window.QUELORA_CONFIG = {
    cid: 'QU-XXXXXXXX-XXXXX',
    apiUrl: 'https://api.your-domain.tld'
  };
</script>
<script type="module" src="https://cdn.quelora.org/quelora.js"></script>
```

Create a client in the dashboard, copy the `cid`, paste this snippet — done.

## Components

Quelora ships as ~14 independent repositories under
[github.com/Quelora](https://github.com/Quelora):

### Core platform

| Repo | Role |
|------|------|
| [quelora-widget-community](https://github.com/Quelora/quelora-widget-community) | The embeddable widget — ES6, Web Worker, Service Worker, WASM. |
| [quelora-public-api](https://github.com/Quelora/quelora-public-api) | Community-facing REST API (port 3000). |
| [quelora-dashboard-api](https://github.com/Quelora/quelora-dashboard-api) | Multi-tenant admin API + Sentinel debug broker (port 3010). |
| [quelora-dashboard](https://github.com/Quelora/quelora-dashboard) | Admin SPA — React 18 + MUI v7. |
| [quelora-worker](https://github.com/Quelora/quelora-worker) | BullMQ worker — emails, notifications, activity. |
| [quelora-jobs](https://github.com/Quelora/quelora-jobs) | Scheduled BullMQ jobs — reputation, gravity decay, suggestions. |
| [quelora-common](https://github.com/Quelora/quelora-common) | Shared library — models, services, providers. |

### Optional modules

| Repo | Role |
|------|------|
| [quelora-enterprise](https://github.com/Quelora/quelora-enterprise) | Backend modules — gamification, surveys, ads, SSE, chat, P2P, live mode. |
| [quelora-widget-enterprise](https://github.com/Quelora/quelora-widget-enterprise) | Client modules paired with the above. |
| [quelora-wasm](https://github.com/Quelora/quelora-wasm) | Rust → WASM (image processor + Markdown parser) used inside the widget worker. |
| [quelora-detoxify](https://github.com/Quelora/quelora-detoxify) | Self-hosted FastAPI wrapper around the Detoxify multilingual toxicity model. |

> *Some modules (P2P resilience, live mode) are marked **experimental** — they
> work but have seen less production exposure. Help especially welcome there.*

### Integrations & operations

| Repo | Role |
|------|------|
| [quelora-wp-plugin](https://github.com/Quelora/quelora-wp-plugin) | WordPress plugin — embed the widget + sync posts and users. |
| [quelora-demo-api](https://github.com/Quelora/quelora-demo-api) | The live demo at [demo.quelora.org](https://demo.quelora.org). |
| [quelora-dx-env](https://github.com/Quelora/quelora-dx-env) | Orchestration — docker-compose, Makefile, setup scripts. |

## License

Released under [**AGPL-3.0**](./LICENSE). Modify, run, self-host, fork freely.
If you offer Quelora as a hosted service to third parties, you must publish
your modifications under the same license. No "open core" — every package is
AGPL, including the optional modules.

## Status — honest

This is the **first public release** of a project that has been running in
production for about a year as a **single-developer effort**. The codebase is
solid because it has been dogfooded daily — not because it has formal test
coverage (it doesn't have any yet; that's `good-first-issue` territory).

The maintainer (**Germán Zelaya**) is openly looking for co-maintainers — see
[MAINTAINERS.md](./MAINTAINERS.md). If any subsystem here interests you and
you want to own it, open a [discussion](https://github.com/Quelora/quelora/discussions).

Quelora is being released **now**, intentionally, because the alternative is
the project dying on a disk drive. The goal of the launch is to find people
who care enough to keep it alive. If that's you, welcome.

> *Behind the scenes:* the final push to release — repo cleanup, documentation,
> the website, the launch plan — was done in collaboration with Claude (Anthropic).
> The architecture and code are human; the homestretch had an AI co-pilot.
> Mentioned here for transparency, not as marketing.

## Contributing

PRs welcome. See [CONTRIBUTING.md](./CONTRIBUTING.md). The on-ramp is the
[`good first issue`](https://github.com/search?q=org%3AQuelora+label%3A%22good+first+issue%22+state%3Aopen&type=issues)
label across all repos.

For security disclosures: [SECURITY.md](./SECURITY.md).
By contributing you agree to the [Code of Conduct](./CODE_OF_CONDUCT.md).
