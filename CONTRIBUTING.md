# Contributing to Quelora

Thanks for being here. Quelora is open precisely because the project needs
contributors to live. Anything from a typo fix to taking ownership of a whole
subsystem is welcome.

## What we welcome

- **Bug reports** — open an issue in the affected repo with reproduction steps.
- **Pull requests** — small, focused, with a clear "before/after".
- **Documentation** — gaps, fixes, translations, examples. Often the highest-leverage contribution.
- **Tests** — there are none yet. Anywhere you can add a passing test is a real contribution.
- **Co-maintainers** — see [MAINTAINERS.md](./MAINTAINERS.md). If a subsystem interests you, open a discussion.

## Setting up a local environment

The fastest path is the orchestration repo:

```bash
git clone https://github.com/Quelora/quelora-dx-env
cd quelora-dx-env
make setup
make up
```

This pulls every component, sets up local SSL via `mkcert`, maps DNS for
`*.quelora.dev`, installs Node dependencies, and restores a seeded MongoDB.
See [quelora-dx-env](https://github.com/Quelora/quelora-dx-env) for prerequisites.

If you only want to work on one repo, clone it standalone and follow its
`README.md`.

## Code style

Per-package conventions live in each repo's `README.md`. Project-wide
standards (logging, error handling, multi-tenancy patterns) are in
[docs/coding-standards.md](./docs/coding-standards.md).

Quick summary:

- **JSDoc** on every module / class / public function.
- **English** for all code, comments and commit messages.
- **No `console.log`** in production code — packages have their own loggers.
- **Optional chaining** carefully (read the standards doc).
- **Multi-tenant first** — every query filters by `cid`. No exceptions.

## Commit messages

Conventional Commits encouraged but not enforced:

```
feat(widget): add reply collapse animation
fix(public-api): handle null geo on /sso/verify
docs(meta): clarify the rate-limit defaults
```

## Pull-request process

1. Fork the repo, branch from `main`.
2. Keep PRs small and focused. One concern per PR.
3. Update relevant docs/READMEs in the same PR.
4. CI must pass (or, if there is no CI, explain how you verified locally).
5. A maintainer reviews. If we're slow, ping in the PR — we won't take offence.

## Becoming a maintainer

Trusted contributors become committers. The path:

1. Land 2-3 meaningful PRs on a given subsystem (widget, dashboard, API, plugin…).
2. Open a discussion in `Quelora/quelora` mentioning you'd like to maintain that
   area.
3. Get push access, plus your name in [MAINTAINERS.md](./MAINTAINERS.md).

There is no CLA — your PR is AGPL by virtue of being applied to an AGPL project.

## Security

For vulnerability reports, please **do not open a public issue**.
See [SECURITY.md](./SECURITY.md).

## Code of conduct

By participating you agree to the [Code of Conduct](./CODE_OF_CONDUCT.md).
The short version: be kind, assume good faith, no harassment.
