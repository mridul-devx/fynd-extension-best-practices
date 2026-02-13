# FDK setup — Rules and context

Use this as the source of rules and context for FDK configuration and server layout. Apply these when editing or suggesting changes to an extension backend.

## Rules

1. **Single FDK entrypoint:** The extension must have one backend module (e.g. `backend/fdk.js`) that calls `setupFdk({ ... })` and exports the result. The server must require this module once and use the same instance for `fdkHandler`, `getPlatformClient`, `platformApiRoutes`, `partnerApiRoutes`, and `webhookRegistry`. Do not create multiple FDK instances or duplicate setup.

2. **Environment variables:** The FDK expects these env vars: `EXTENSION_API_KEY`, `EXTENSION_API_SECRET`, `EXTENSION_BASE_URL`, `FP_API_DOMAIN`. When suggesting or reviewing config, ensure all are set; do not hardcode secrets or base URLs.

3. **getPlatformClient(companyId):** Always use `await fdkExtension.getPlatformClient(companyId)` for any Fynd Platform API call. Never call platform APIs without a company-scoped client. If the client is null or throws (e.g. company not authenticated), return 401 or 400 with a clear message; do not proceed with platform calls.

4. **Route mount order (strict):** In the server file, mount routes in this order:
   - First: Inbound webhook routes (external service → extension).
   - Second: Other extension API routes (e.g. `/api/checkout`, `/api/configurations`).
   - Third: The route that calls `fdkExtension.webhookRegistry.processWebhook(req)` (e.g. `/api/webhook-events`).
   - Fourth: `app.use("/", fdkExtension.fdkHandler)`.
   - Fifth: Mount `platformApiRoutes`, `partnerApiRoutes`, and any `basicRouter` under their prefixes (`/api`, `/apipartner`, `/apibasic`).
   - Last: Catch-all for SPA (e.g. `app.get("*", ...)` serving `index.html`).
   Do not mount custom or webhook routes after `fdkHandler`; they would never be reached.

5. **Auth callback:** `callbacks.auth` must return the URL to redirect the user after OAuth (e.g. `${req.extension.base_url}/company/${req.query.company_id}` or with `application_id`). Do not call `getPlatformClient` inside `callbacks.install`; the token may not be stored yet.

6. **Webhook config:** `webhook_config.api_path` must match the path mounted in the server that calls `webhookRegistry.processWebhook(req)`. Every event the extension handles must be listed in `webhook_config.event_map` with `{ handler, version }`.

## Context (reference)

- **Package:** `@gofynd/fdk-extension-javascript` (Express: `@gofynd/fdk-extension-javascript/express`).
- **Exports from FDK module:** `fdkHandler`, `getPlatformClient(companyId)`, `platformApiRoutes`, `partnerApiRoutes`, `webhookRegistry`.
- **Storage:** Session storage (e.g. SQLite via `SQLiteStorage`) is for OAuth tokens. Extension business data (config, logs) should live in your own DB keyed by `fynd_company_id` or similar.
