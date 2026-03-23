# Extension patterns — Rules by type

Rules by extension type. Apply the relevant section based on what the extension does.

## Catalog extensions

1. **Platform client:** Obtain `companyId` from the request and call `getPlatformClient(companyId)`. Use `platformClient.catalog.getProducts()` for company-wide catalog or `platformClient.application(applicationId).catalog.getAppProducts()` for storefront-specific products.

2. **Routes:** Catalog routes belong on the platform router under `/api`. Do not call catalog APIs without a platform client.

3. **Filtering:** Filter results by `extension_id === process.env.EXTENSION_API_KEY` when the API returns extension-owned entities.

## Platform-scoped extensions

1. **All platform calls:** Use `getPlatformClient(companyId)` for every Fynd API call — catalog, order, application, serviceability, etc. Do not mix company scopes.

2. **Extension-owned entities:** Filter by `extension_id === process.env.EXTENSION_API_KEY`. Store config keyed by `fynd_company_id` (and optionally `application_id`).

3. **Webhooks:** Event names and payloads depend on the extension type. Refer to Fynd Partner docs — do not assume event names without documentation.

## Custom extensions

1. **Routes:** Use `platformApiRoutes` under `/api` for platform operations, `partnerApiRoutes` under `/apipartner` for partner-scoped work, and a `basicRouter` under `/apibasic` for config, uploads, or other custom logic.

2. **Storage:** Use your own DB for business data, keyed by `fynd_company_id` (and `application_id` when relevant). FDK session storage is for OAuth tokens only.

3. **Webhooks:** For Fynd→extension, add events to `webhook_config.event_map`. For external→extension, mount POST routes before `fdkHandler` and verify signatures. See [webhooks.md](webhooks.md) and [security.md](security.md).

4. **Filtering:** Filter platform entities by `extension_id` before returning to the client.

## Database patterns (all types)

1. **Primary key:** Key all business data by `fynd_company_id`. Add `application_id` as a secondary key when the data is application-specific.

2. **Company isolation:** Always include the company filter in database queries — never fetch all records and filter in application code. This prevents data leaks and improves performance.

3. **Common models:**

```
configurations
├── fynd_company_id (indexed)
├── application_id (optional, indexed)
├── settings (JSON)
└── updated_at

company_mappings
├── fynd_company_id (indexed)
├── external_service_id
├── external_ref
└── created_at

webhook_events
├── event_name
├── company_id (indexed)
├── application_id
├── payload (JSON)
├── status (pending | processed | failed)
└── created_at
```

4. **Migrations:** When adding new fields or tables, ensure they are company-scoped by default. Never create tables without a `fynd_company_id` column for business data.
