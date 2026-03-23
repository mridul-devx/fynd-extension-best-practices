# FDK setup — Rules and context

Rules for FDK configuration, server layout, and lifecycle callbacks. Apply when editing or suggesting changes to an extension backend.

## Rules

1. **Single FDK entrypoint:** One backend module (e.g. `backend/fdk.js`) calls `setupFdk({ ... })` and exports the result. The server imports this once. Do not create multiple FDK instances.

2. **Environment variables:** Required: `EXTENSION_API_KEY`, `EXTENSION_API_SECRET`, `EXTENSION_BASE_URL`, `FP_API_DOMAIN`. Never hardcode these. Verify all are set before startup.

3. **getPlatformClient(companyId):** Always `await` — it returns a Promise. If it returns null or throws, return 401 to the client. Never proceed with platform calls on failure. See [error-handling.md](error-handling.md) for the wrapper pattern.

4. **Route mount order (strict):** In the server file, mount in this order:
   1. Inbound webhook routes (external → extension)
   2. Extension API routes (e.g. `/api/checkout`, `/api/configurations`)
   3. Fynd webhook processing route (`webhookRegistry.processWebhook(req)`)
   4. `app.use("/", fdkExtension.fdkHandler)`
   5. `platformApiRoutes`, `partnerApiRoutes`, `basicRouter` under their prefixes
   6. SPA catch-all (`app.get("*", ...)`)

   See [code-examples.md](code-examples.md) for the full server file pattern.

5. **Auth callback:** `callbacks.auth` returns the redirect URL after OAuth (e.g. `${req.extension.base_url}/company/${req.query.company_id}`). This is where the merchant lands after authorizing.

6. **Install callback:** `callbacks.install` fires when a company installs the extension. Use it to create initial config records or store company mappings. Do **not** call `getPlatformClient` here — the OAuth token may not be persisted yet.

7. **Uninstall callback:** `callbacks.uninstall` fires when a company removes the extension. Clean up company-specific data, revoke external service access, and remove stored mappings.

8. **Webhook config:** `webhook_config.api_path` must match the mounted route that calls `webhookRegistry.processWebhook(req)`. Every handled event must be in `webhook_config.event_map` with `{ handler, version }`.

## Context

- **Package:** `@gofynd/fdk-extension-javascript/express`.
- **Exports:** `fdkHandler`, `getPlatformClient(companyId)`, `platformApiRoutes`, `partnerApiRoutes`, `webhookRegistry`.
- **Storage:** Session storage (e.g. `SQLiteStorage`) is for OAuth tokens only. Extension business data belongs in your own DB keyed by `fynd_company_id`.
- **Access modes:** `"online"` — token refreshed per session; `"offline"` — long-lived token for background jobs.
