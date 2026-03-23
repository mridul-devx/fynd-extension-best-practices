# Webhooks — Rules and context

Rules for webhook handling in both directions: Fynd→extension (outbound) and external→extension (inbound).

## Fynd → extension (outbound)

1. **Registration:** Every handled event must be in `webhook_config.event_map` with `{ handler, version }`. The `webhook_config.api_path` must match the mounted route that calls `webhookRegistry.processWebhook(req)`.

2. **Handler signature:** `(event_name, request_body, company_id, application_id)` — use all four arguments. Scope DB writes and API calls by `company_id` and `application_id`.

3. **Error boundary:** Catch all errors inside the handler. Log failures but do not throw — Fynd expects a timely 200. Unhandled throws may trigger retries and duplicate processing. See [error-handling.md](error-handling.md) for the pattern.

4. **Non-blocking:** If work is heavy (API calls, large DB writes), acknowledge the webhook immediately and process asynchronously via a job queue.

5. **Event names and payloads:** Depend on extension type. Always refer to Fynd Partner docs for correct event names and payload shapes.

## External → extension (inbound)

1. **Mount before FDK:** Routes receiving external callbacks must be mounted **before** `fdkHandler`. Otherwise the FDK catch-all consumes the request.

2. **Verify authenticity:** If the external service supports HMAC signatures, API key headers, or IP allowlisting, verify before processing. See [security.md](security.md) for the HMAC pattern.

3. **Resolve companyId first:** Before calling Fynd APIs, resolve `companyId` from the payload, header, or a stored mapping in your DB. If unresolvable, return 400.

4. **Use getPlatformClient(companyId):** After resolving company, `await fdkExtension.getPlatformClient(companyId)`. If null or throws, return 401.

5. **Return 200 on success:** Return 200 after processing so the external system does not retry. Use 4xx/5xx for genuine failures.

## Context

- **Resolving companyId for inbound webhooks:** From a header (`x-company-id`), the request body, or a mapping stored in your DB when handling a prior Fynd webhook. The extension needs a deterministic way to map external callbacks to Fynd companies.
- **Common pattern:** When Fynd sends an outbound webhook, store a `{ external_ref, fynd_company_id }` mapping. When the external service calls back, look up `fynd_company_id` by `external_ref`.
