# Quelora — Documentation

Developer and architecture documentation for the
[Quelora](https://github.com/Quelora/quelora) platform.

For an overview of what Quelora is, see the [meta-repo README](../README.md).
For per-component code, see each repo on the
[Quelora organization](https://github.com/Quelora).

## Index

### Cross-cutting

| Doc | Contents |
|-----|----------|
| [architecture.md](./architecture.md) | High-level view — monorepo layout, runtime services, data flow, multi-tenancy, encryption, RBAC. **Start here.** |
| [coding-standards.md](./coding-standards.md) | Quality, documentation, logging, error handling, architecture constraints — applies to all packages. |

### Per-package

| Package | Doc |
|---------|-----|
| [`@quelora/common`](https://github.com/Quelora/quelora-common) | [packages/quelora-common.md](./packages/quelora-common.md) |
| [`quelora-public-api`](https://github.com/Quelora/quelora-public-api) | [packages/quelora-public-api.md](./packages/quelora-public-api.md) |
| [`quelora-dashboard-api`](https://github.com/Quelora/quelora-dashboard-api) | [packages/quelora-dashboard-api.md](./packages/quelora-dashboard-api.md) |
| [`quelora-dashboard`](https://github.com/Quelora/quelora-dashboard) | [packages/quelora-dashboard.md](./packages/quelora-dashboard.md) |
| [`quelora-widget-community`](https://github.com/Quelora/quelora-widget-community) | [packages/quelora-widget-community.md](./packages/quelora-widget-community.md) |

### Widget operations

| Doc | Contents |
|-----|----------|
| [widget-cdn-release.md](./widget-cdn-release.md) | CDN versioning — Cloudflare R2 + Worker, release workflow, enterprise access control. |

### Operations

A dedicated self-hosting / production-deployment guide is **being written**.
In the meantime, the canonical source is the
[`quelora-dx-env`](https://github.com/Quelora/quelora-dx-env) repository — its
Makefile orchestrates the full stack and its `scripts/` directory documents
every step of the install.

---

If something is missing or wrong, please open an issue — documentation gaps
are often the highest-leverage contribution.
