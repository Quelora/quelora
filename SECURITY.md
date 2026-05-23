# Security policy

## Reporting a vulnerability

**Please do not open a public issue for security reports.**

Email: **security@quelora.org**

Include:

- A description of the issue and where it lives (which repo, which file/area).
- Reproduction steps or a proof-of-concept.
- Affected versions / commit hashes if known.
- Whether you'd like to be credited in the advisory (and how).

A PGP key is available on request.

## Response expectations — honest

Quelora is currently maintained by a single developer with limited time.
Best-effort timeline:

- **Acknowledgement:** within 5 business days.
- **Triage and severity assessment:** within 14 days.
- **Fix and disclosure:** depends on severity; coordinated with the reporter.

We do not yet have a bug-bounty program.

## In scope

- Authentication / authorization bypass in `quelora-public-api` or `quelora-dashboard-api`.
- Tenant isolation bypass (cross-`cid` data access).
- Decryption of at-rest encrypted fields without the key.
- Stored XSS in the widget or dashboard.
- CSRF in dashboard or admin endpoints.
- RCE in any service.
- Sensitive data leakage in API responses, logs, or default configs.

## Out of scope (for now)

- Findings on the **demo** server ([demo.quelora.org](https://demo.quelora.org))
  that depend on the demo's specific data — please reproduce in a self-hosted
  instance and report against the codebase, not the demo.
- Self-XSS or social-engineering scenarios that require the victim to paste
  hostile code into their own browser console.
- Issues that require physical access to the host machine.
- Best-practice nags (TLS configuration of `demo.quelora.org`, header policies)
  unless they enable a concrete attack.

## Disclosure

Once a fix is shipped, we publish a security advisory via GitHub's advisories
mechanism on the affected repo. Reporters who opt in are credited.

---

*This policy will get more formal as the project matures. PRs to tighten it
are welcome.*
