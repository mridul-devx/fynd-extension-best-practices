# Fynd Extension Skill — Analysis & Enhancement Plan

## Overview

The `fynd-extension` skill guides AI agents when working on Fynd Platform extensions built with the FDK (Fynd Development Kit). It ensures AI-generated code follows FDK conventions, security patterns, and architectural constraints.

---

## Current Structure

```
.agents/skills/fynd-extension/
├── SKILL.md                          # Main entry — 7 core rules, quick context, behavior guide
└── references/
    ├── fdk-setup.md                  # FDK config, env vars, route mount order (6 rules)
    ├── extension-patterns.md         # Rules by extension type: catalog, platform, custom
    └── webhooks.md                   # Fynd→extension and external→extension webhook rules
```

### How It Works

1. **SKILL.md** is loaded first by the AI agent — contains the 7 non-negotiable rules
2. **Reference files** are loaded on-demand when the AI needs deeper context (e.g., adding a webhook, changing FDK setup)
3. Rules are numbered and bold-keyed for quick scanning

---

## What the Skill Covers Today

| Area | Coverage | File(s) |
|---|---|---|
| Company scoping | Full | SKILL.md, all references |
| Route mount order | Full | SKILL.md, fdk-setup.md |
| getPlatformClient usage | Full | SKILL.md, fdk-setup.md |
| Fynd→extension webhooks | Full | SKILL.md, webhooks.md |
| External→extension webhooks | Full | SKILL.md, webhooks.md |
| Extension entity filtering | Full | SKILL.md, extension-patterns.md |
| Single FDK entrypoint | Full | SKILL.md, fdk-setup.md |
| Extension types (catalog/platform/custom) | Good | extension-patterns.md |
| Environment variables | Basic | fdk-setup.md |
| Session storage vs business DB | Basic | fdk-setup.md, extension-patterns.md |
| Auth callback | Basic | fdk-setup.md |

---

## What the Skill Does NOT Cover

| Area | Impact | Notes |
|---|---|---|
| Code examples | **High** | Zero code snippets — AI agents guess patterns |
| Error handling patterns | **High** | Says "return 4xx" but no standard shape |
| Frontend / extension panel | **High** | 100% backend — no panel UI guidance |
| Lifecycle callbacks (install/uninstall) | **Medium** | Only `auth` callback mentioned |
| Application-scoped vs company-scoped APIs | **Medium** | One mention, no rule |
| Security (HMAC, input validation, CORS) | **Medium** | Only company scoping covered |
| Anti-patterns / common mistakes | **Medium** | No explicit "don't do this" list |
| Database & multi-tenancy patterns | **Medium** | "Use your own DB" with no guidance |
| Logging & debugging | **Low-Med** | No structured logging rules |
| Pagination & rate limits | **Low** | No guidance on paginated APIs |
| Middleware patterns | **Low** | No common middleware guidance |
| Partner API usage | **Low** | Mentioned but not explained |

---

## Enhancement Plan

### Phase 1: High-Impact Additions

#### 1.1 Add Code Examples to Existing Rules

Add a new reference file `references/code-examples.md` with concrete patterns:

```
references/code-examples.md (NEW)
├── setupFdk() — full config shape
├── Server file — route mount order in Express
├── Webhook handler — complete implementation
├── getPlatformClient — with error handling
├── Entity filtering — before returning to client
└── Application-scoped API — when to use
```

**Why:** AI agents generate 3-5x more accurate code when they have patterns to match against. The current skill tells agents *what* to do but not *how* it looks in code.

#### 1.2 Add Error Handling Reference

Add `references/error-handling.md`:

```
references/error-handling.md (NEW)
├── Standard error response shape
├── getPlatformClient failure handling
├── Webhook error boundaries (try/catch + still return 200)
├── Express error middleware
└── Validation errors
```

#### 1.3 Add Frontend / Panel Reference

Add `references/frontend-panel.md`:

```
references/frontend-panel.md (NEW)
├── Extension launch URL structure
├── Company context in frontend
├── Frontend → backend API calls (with companyId header)
├── Application context propagation
└── Panel framework patterns (React/Vue)
```

---

### Phase 2: Medium-Impact Additions

#### 2.1 Add Lifecycle Callbacks to fdk-setup.md

Add rules for `callbacks.install` and `callbacks.uninstall`:
- What to do on install (create company config, initial sync)
- What to do on uninstall (cleanup, revoke access)
- Warning: no `getPlatformClient` in `install` (token not stored yet)

#### 2.2 Add Application-Scoped API Rule to SKILL.md

Add rule 8:
- Use `platformClient.catalog.*` for company-wide operations
- Use `platformClient.application(appId).catalog.*` for application-specific data
- Always resolve `application_id` from the request when using app-scoped APIs

#### 2.3 Add Security Reference

Add `references/security.md`:

```
references/security.md (NEW)
├── Inbound webhook signature verification (HMAC)
├── Input validation on webhook payloads
├── Environment variable handling (never log secrets)
├── CORS configuration for extension panel
└── Rate limiting inbound endpoints
```

#### 2.4 Add Anti-Patterns Section to SKILL.md

Add a "Common mistakes" section:

```markdown
## Common mistakes (avoid)

- Forgetting `await` on `getPlatformClient(companyId)` — it's async
- Storing business data in FDK session storage — it's for OAuth only
- Hardcoding company IDs — always read from the request
- Creating multiple FDK instances — use the single exported module
- Mounting routes after fdkHandler — they'll never be reached
- Calling getPlatformClient in callbacks.install — token not stored yet
```

---

### Phase 3: Low-Impact Clean-ups

#### 3.1 Reduce Redundancy

Rules repeated across files:
- `extension_id === process.env.EXTENSION_API_KEY` — **4 times**
- `getPlatformClient(companyId)` rules — **every file**
- Webhook handler signature — **2 times**

**Fix:** SKILL.md states each rule once. Reference files add depth (examples, edge cases, type-specific nuance) instead of repeating the same text.

#### 3.2 Improve Description for Better Triggering

Current:
```yaml
description: |
  Guides development and debugging of Fynd extensions using the FDK...
```

Enhanced:
```yaml
description: |
  Guides development and debugging of Fynd extensions using the FDK (Fynd Development Kit)
  with Node.js and Express. Covers backend layout, webhooks, company-scoped config,
  frontend panel patterns, and common patterns for catalog, custom, and platform-scoped
  extensions. Use when working on any Fynd extension, FDK, Fynd Platform APIs, partner
  panel, extension webhooks, or @gofynd/fdk-extension-javascript.
```

#### 3.3 Add Database Patterns

Add to `extension-patterns.md` under each type:
- Schema keying: `fynd_company_id` + `application_id`
- Multi-tenant isolation patterns
- Config storage model

---

## Proposed Final Structure

```
.agents/skills/fynd-extension/
├── SKILL.md                              # 9 core rules + anti-patterns + quick context
└── references/
    ├── fdk-setup.md                      # FDK config, env vars, route order, lifecycle callbacks
    ├── extension-patterns.md             # Rules by type + DB patterns (enhanced)
    ├── webhooks.md                       # Webhook rules (existing, unchanged)
    ├── code-examples.md                  # NEW — concrete code patterns
    ├── error-handling.md                 # NEW — error shapes, middleware, boundaries
    ├── frontend-panel.md                 # NEW — panel UI, launch URL, frontend API calls
    └── security.md                       # NEW — HMAC, validation, CORS, secrets
```

---

## Priority Order

| Priority | Enhancement | Effort | Impact |
|---|---|---|---|
| **P0** | Code examples reference | Medium | Very High |
| **P0** | Anti-patterns section in SKILL.md | Low | High |
| **P1** | Error handling reference | Medium | High |
| **P1** | Frontend panel reference | Medium | High |
| **P1** | Application-scoped API rule | Low | Medium |
| **P2** | Lifecycle callbacks (install/uninstall) | Low | Medium |
| **P2** | Security reference | Medium | Medium |
| **P2** | Description keyword improvement | Low | Low |
| **P3** | Redundancy cleanup | Low | Low |
| **P3** | Database patterns | Low | Medium |

---

## Summary

The skill has a **strong foundation** — clear rules, good structure, correct FDK guidance. The biggest gap is **no code examples**, which forces AI agents to guess implementation patterns. The second gap is **backend-only coverage** — Fynd extensions always have a frontend panel that the skill completely ignores. Adding these two would significantly improve AI code generation quality for Fynd extension work.
