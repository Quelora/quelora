# 02 — Coding Standards & Rules

📏 **Quelora SDK coding standards.** These rules apply to **all packages, always**.

> Source: `.ai/coding-standards.md`

---

## 1. Technical quality & completeness

- **No ellipses / no fragments** — Always output the complete file or the complete
  function block. Never use `// ... rest of code` or `// ...existing code...`.
- **Production ready** — Code must be strictly for production. Absolutely no `// TODO`,
  `// FIXME`, or `// FIX` comments allowed.
- **Skepticism first** — Do not assume existing patterns are flawless. Verify
  performance, scaling, and UX implications before implementing.

## 2. Documentation

- **JSDoc mandatory** — Every module, class, and public function must be documented
  using strict JSDoc.
- **English only** — All JSDoc, inline comments, and commit messages MUST be in English.

## 3. Logging

- **Never use `console.log` directly** in production code.
- **`quelora-widget-community` only:** use the internal logger
  `handleLog(message, context, level, icon)`.
  Example: `handleLog('User auth failed', 'AuthModule', 'error', '🔒');`
- Other packages (`quelora-dashboard`, `quelora-dashboard-api`, `quelora-public-api`)
  use their own logging conventions — check each package's overview doc first.

## 4. Error handling & defensive programming

- **Optional chaining** — Use `?.` carefully. Do not use it for boolean evaluations
  without explicit existence checks. Prefer
  `if (obj && obj.readyState === obj.OPEN)` over `if (obj?.readyState === obj?.OPEN)`.
- **Safe parsing** — Always wrap `JSON.parse` and dynamic imports in `try/catch` blocks.
- **Graceful degradation** — If a dynamic plugin fails to load, degrade silently
  (Community Edition fallback) rather than crashing the thread.

## 5. Architecture constraints

- **Worker offloading** — Any cryptographic operation, heavy data parsing, or WASM
  execution MUST be done in the Worker (`queloraWorker.js`), never in the Main Thread.
- **Prop drilling** — Avoid prop drilling. Use the `eventBus` for cross-component
  communication.
