# fynd-extension-best-practices

Agent skill for **Fynd extension (FDK)** backend and partner panel development. Guides AI agents and developers on FDK setup, route layout, webhooks, and patterns for catalog, logistics, and custom extensions.

## When to use

Use this skill when working on:

- Any Fynd extension backend (Node/Express + FDK)
- FDK, Fynd Platform APIs, or partner panel
- Extension webhooks (Fynd → extension or external → extension)
- Catalog, custom, or platform-scoped extensions

## Install

```bash
npx skills add <your-username>/fynd-extension-best-practices --skill fynd-extension
```

Replace `<your-username>` with your GitHub username. Example:

```bash
npx skills add mridul/fynd-extension-best-practices --skill fynd-extension
```

## How it's useful

- **For developers:** Encodes FDK semantics (getPlatformClient, event_map, route mount order, company scope) so the agent suggests correct routes, webhook registration, and where to add code.
- **For AI agents (Cursor, Claude Code, etc.):** Description triggers on "Fynd extension," "FDK," "webhooks," "partner panel"; SKILL.md and references keep answers aligned with FDK and extension layout.
- **For onboarding:** Reference docs cover FDK setup, webhooks (inbound/outbound), and patterns by type (catalog, custom, platform-scoped).


Use this skill when editing extension APIs or webhooks; use fynd-theme-best-practices for theme/storefront work.

## Contents

- `fynd-extension/` — Skill directory (SKILL.md + references). Do not add README inside this folder (per skills.sh best practices).

## Links

- [skills.sh](https://skills.sh) — Agent Skills directory
- [Fynd Partner / FDK docs](https://docs.fynd.com) — Platform and extension documentation
