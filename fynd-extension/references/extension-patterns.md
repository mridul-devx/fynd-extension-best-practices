# Extension patterns — Rules and context

Rules and context by extension type. Apply these when the extension uses catalog, platform APIs, or custom routes so the AI keeps suggestions consistent with FDK.

## Rules: Catalog extensions

1. **Platform client:** Always obtain `companyId` from the request (`x-company-id` header or query/body) and call `getPlatformClient(companyId)` before using `platformClient.catalog` or `platformClient.application(...).catalog`. Do not attach a platform client to `req` without ensuring it is company-scoped.

2. **Routes:** Catalog routes (e.g. products, app products) belong on the platform router mounted under `/api`. Use `platformClient.catalog.getProducts()` or `platformClient.application(application_id).catalog.getAppProducts()` as needed; do not call catalog APIs without a platform client.

3. **Filtering:** When the platform API returns entities that can be owned by an extension (e.g. schemes), always filter the list by `extension_id === process.env.EXTENSION_API_KEY` before returning to the client so only this extension’s resources are shown.

## Rules: Platform-scoped extensions

1. **All platform calls:** Use `getPlatformClient(companyId)` for every Fynd Platform API call (catalog, order, application, serviceability, or any other). Do not mix company scopes or call platform APIs without a client.

2. **Extension-owned entities:** When listing or returning entities that are tied to an extension (e.g. schemes, accounts), always filter by `extension_id === process.env.EXTENSION_API_KEY`. Store company-scoped config keyed by `fynd_company_id` (and optionally `application_id`).

3. **Webhooks:** Event names and payloads depend on the extension type. When adding or changing webhook handlers, use Fynd Partner docs for event names and payload structure; do not assume without documentation.

## Rules: Custom extensions

1. **Routes:** Use `platformApiRoutes` (or a router under `/api`) for platform API usage, `partnerApiRoutes` for partner-scoped operations, and a `basicRouter` (or similar) under a dedicated prefix (e.g. `/apibasic`) for config, file uploads, or other custom logic. Always resolve `companyId` (from header, query, or body) and use `getPlatformClient(companyId)` when calling Fynd from any of these routes.

2. **Storage:** Use your own DB for config, logs, or entity data; key by `fynd_company_id` (and optionally `application_id`). Do not use FDK session storage for business data; it is for OAuth tokens only.

3. **Webhooks:** Fynd → extension: add events to `webhook_config.event_map` and implement handlers with signature `(event_name, request_body, company_id, application_id)`. External → extension: add POST routes and mount them before `fdkHandler`; resolve `companyId` then call `getPlatformClient(companyId)` and Fynd APIs. See [webhooks.md](webhooks.md) for rules.

4. **Filtering:** When listing platform entities that can be owned by an extension, always filter by `extension_id === process.env.EXTENSION_API_KEY` so only this extension’s resources are returned.
