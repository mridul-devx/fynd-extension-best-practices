# Error handling — Rules and patterns

Rules for consistent error handling across Fynd extension backends. Apply these when writing or reviewing route handlers, webhook handlers, and middleware.

## Rules

1. **Standard error shape:** All error responses must use a consistent JSON shape: `{ message: string, code?: string }`. Do not return plain strings, HTML, or inconsistent objects.

2. **getPlatformClient failures:** If `getPlatformClient(companyId)` returns null or throws, return 401 with `{ message: "Company not authenticated. Reinstall the extension." }`. Do not proceed with platform API calls. Do not swallow the error silently.

3. **Platform API errors:** Wrap Fynd Platform API calls in try/catch. If the API returns a 4xx/5xx, forward a meaningful message to the client — do not expose raw Fynd error internals. Log the full error server-side.

4. **Webhook error boundaries:** In Fynd→extension webhook handlers, catch all errors inside the handler. Log the error but do not throw — Fynd expects a timely 200 response. If the handler throws, the webhook delivery may be retried, causing duplicate processing.

5. **Inbound webhook errors:** In external→extension webhook handlers, always return an appropriate HTTP status so the external system knows the result. Return 200 on success, 400 for bad payloads, 401 for unresolvable companies, 500 for unexpected failures.

6. **Validation before processing:** Validate required fields in the request (companyId, applicationId, payload fields) before performing any work. Return 400 with a clear message listing what is missing.

## Error response shape

```js
// Success
res.status(200).json({ success: true, data: { ... } });

// Client error
res.status(400).json({ message: "Missing required field: company_id" });

// Auth error
res.status(401).json({ message: "Company not authenticated. Reinstall the extension." });

// Not found
res.status(404).json({ message: "Scheme not found" });

// Server error
res.status(500).json({ message: "Internal server error" });
```

## Express error middleware

Mount this **after** all routes as the final middleware:

```js
// Error handling middleware — mount last
app.use((err, req, res, _next) => {
  const companyId = req.headers["x-company-id"] || "unknown";
  console.error(`[error] company=${companyId} path=${req.path}`, err);

  // Don't leak internal errors to the client
  const status = err.status || 500;
  const message = status < 500 ? err.message : "Internal server error";
  res.status(status).json({ message });
});
```

## getPlatformClient error wrapper

```js
async function getClient(req, res) {
  const companyId = req.headers["x-company-id"] || req.query.company_id;

  if (!companyId) {
    res.status(400).json({ message: "Missing company_id" });
    return null;
  }

  try {
    const client = await fdkExtension.getPlatformClient(companyId);
    if (!client) {
      res.status(401).json({ message: "Company not authenticated. Reinstall the extension." });
      return null;
    }
    return client;
  } catch (err) {
    console.error(`getPlatformClient failed for company=${companyId}:`, err.message);
    res.status(401).json({ message: "Company not authenticated. Reinstall the extension." });
    return null;
  }
}
```

## Platform API error wrapper

```js
async function callPlatformApi(apiCall, res, fallbackMessage) {
  try {
    return await apiCall();
  } catch (err) {
    console.error(`[platform-api] ${fallbackMessage}:`, err.message);
    const status = err.status || 502;
    res.status(status).json({ message: fallbackMessage });
    return null;
  }
}

// Usage
router.get("/products", async (req, res) => {
  const client = await getClient(req, res);
  if (!client) return;

  const result = await callPlatformApi(
    () => client.catalog.getProducts({ pageSize: 10 }),
    res,
    "Failed to fetch products from Fynd"
  );
  if (!result) return;

  res.json(result);
});
```

## Webhook handler error boundary

```js
async function handleOrderPlaced(event_name, request_body, company_id, application_id) {
  try {
    // ... process the webhook
  } catch (err) {
    // Log but NEVER throw — Fynd expects a 200 response
    console.error(`[webhook] ${event_name} failed for company=${company_id}:`, err);
    // Optionally store the failure for retry via your own job queue
    await db.insert("failed_webhooks", {
      event_name, company_id, application_id,
      error: err.message,
      payload: JSON.stringify(request_body),
    });
  }
}
```
