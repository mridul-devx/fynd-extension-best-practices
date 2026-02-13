# Webhooks — Rules and context

Use these rules when adding or changing webhook handling (Fynd → extension or external → extension). They give the AI the context needed to suggest correct behavior.

## Rules: Fynd → extension (outbound)

1. **Registration:** Every Fynd webhook event the extension handles must be listed in `webhook_config.event_map` with `{ handler, version }`. The path in `webhook_config.api_path` must be mounted in the server and must call `fdkExtension.webhookRegistry.processWebhook(req)` and return 200 on success. Do not register events without a handler; do not mount the webhook path after `fdkHandler`.

2. **Handler signature:** Every event handler must have exactly the signature `(event_name, request_body, company_id, application_id)`. Use `company_id` and `application_id` for all DB writes and API calls inside the handler; do not ignore them or use global state for company scope.

3. **Non-blocking:** Do not perform long-running or blocking work inside the handler before responding. If work is heavy, acknowledge the webhook and process asynchronously so Fynd receives a timely 200.

4. **Event names and payloads:** They depend on the extension type. When adding or debugging a webhook, refer to Fynd Partner docs for the correct event name and payload shape; do not assume event names or payload structure without documentation.

## Rules: External service → extension (inbound)

1. **Mount before FDK:** Any route that receives POST callbacks from an external system must be mounted **before** `app.use("/", fdkExtension.fdkHandler)`. Otherwise the FDK catch-all will handle the request and the webhook handler will never run.

2. **Resolve companyId first:** Before calling any Fynd API from an inbound webhook handler, resolve `companyId` from the request (payload, header, or DB lookup). If `companyId` cannot be resolved, return 400 or 401 and do not call `getPlatformClient` or Fynd APIs.

3. **Use getPlatformClient(companyId):** After resolving `companyId`, call `const platformClient = await fdkExtension.getPlatformClient(companyId)`. If the result is null or throws, return 401 or 400 (e.g. company not authenticated); do not proceed with platform calls.

4. **Return 200 on success:** Return 200 (and optionally a JSON body) after processing so the external system does not retry unnecessarily. Use appropriate 4xx/5xx for failures.

## Context (reference)

- **Resolving companyId:** From a header (e.g. `x-company-id`), from the request body, or from your DB (e.g. a mapping stored when handling an outbound Fynd webhook). The extension must have a deterministic way to know which Fynd company an inbound webhook belongs to.
