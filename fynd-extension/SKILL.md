---
name: fynd-extension
description: |
  Guides development and debugging of Fynd extensions using the FDK (Fynd Development Kit).
  Covers backend layout (Express, platform/partner/basic routes), webhooks, company-scoped config,
  and common patterns for catalog, custom, and platform-scoped extensions. Use when working on any
  Fynd extension, FDK, Fynd Platform APIs, partner panel, or extension webhooks.
---

# Fynd Extension (FDK) — Rules and context for the AI

When working on a Fynd extension codebase, follow these rules and use this context so your suggestions stay correct and consistent with FDK.

## Rules (always apply)

1. **Company context:** Every Fynd Platform API call must be scoped to a company. Always obtain `companyId` from the request (`x-company-id` header, `company_id` in query/body, or your DB) and use `getPlatformClient(companyId)` — never call platform APIs without a company-scoped client.

2. **Route mount order:** Custom webhooks and extension API routes must be mounted **before** `fdkExtension.fdkHandler`. If you add a new route, place it before the FDK handler in the server file so it is not swallowed by the catch-all.

3. **Fynd API access:** Use only `fdkExtension.getPlatformClient(companyId)` to get a client for Fynd Platform APIs (catalog, order, application, etc.). Do not construct API clients manually.

4. **Webhook handlers (Fynd → extension):** Handlers registered in `webhook_config.event_map` must have the signature `(event_name, request_body, company_id, application_id)`. Always use `company_id` and `application_id` to scope any DB writes or API calls inside the handler.

5. **Inbound webhooks (external → extension):** Any POST route that receives callbacks from an external system must be mounted before the FDK handler. Resolve `companyId` from the payload or your DB before calling `getPlatformClient(companyId)`; if you cannot resolve it, return 4xx and do not call Fynd APIs.

6. **Extension-owned entities:** When listing or returning platform entities that belong to an extension (e.g. schemes, accounts), always filter by `extension_id === process.env.EXTENSION_API_KEY` so only this extension’s resources are shown.

7. **Backend entrypoint:** FDK is configured in a single backend module (e.g. `backend/fdk.js`) via `setupFdk()`. The server mounts `fdkHandler`, `platformApiRoutes`, `partnerApiRoutes`, and the webhook route; do not duplicate FDK setup or mount the same paths twice.

## Quick context

- **FDK module:** Exports `fdkHandler`, `getPlatformClient(companyId)`, `platformApiRoutes`, `partnerApiRoutes`, `webhookRegistry`. Use the same instance everywhere.
- **Routers:** Platform routes (e.g. catalog) under `/api`; partner routes under `/apipartner`; extension-specific routes under `/apibasic` or your own prefix. All need `companyId` and `getPlatformClient(companyId)` when calling Fynd.
- **Event names and payloads:** Depend on extension type. Refer to Fynd Partner docs when adding or debugging webhooks.

## When adding or changing behavior

- **New API route:** Add a router in the backend, mount it in the server **before** the FDK handler; in the route, read `companyId` (e.g. from `x-company-id`), call `getPlatformClient(companyId)`, then call Fynd APIs.
- **New Fynd webhook:** Add the event to `webhook_config.event_map` in the FDK module with `{ handler, version }`; implement the handler with signature `(event_name, request_body, company_id, application_id)`.
- **New inbound webhook:** Add a POST route and mount it before `fdkHandler`; in the handler, resolve `companyId`, then `getPlatformClient(companyId)` and Fynd APIs as needed.
- **Extension-specific logic:** Attach to the appropriate router (platform/partner/basic); when returning lists from the platform that are extension-scoped, filter by `extension_id === process.env.EXTENSION_API_KEY`.

## References (load when needed)

- [references/fdk-setup.md](references/fdk-setup.md) — FDK setup rules, env vars, route order.
- [references/webhooks.md](references/webhooks.md) — Webhook registration and inbound webhook rules.
- [references/extension-patterns.md](references/extension-patterns.md) — Rules by extension type (catalog, platform-scoped, custom).
