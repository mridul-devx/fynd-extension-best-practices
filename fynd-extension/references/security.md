# Security — Rules and patterns

Security rules for Fynd extension backends. Apply these when handling webhooks, API routes, and configuration.

## Rules

1. **Inbound webhook verification:** When receiving webhooks from external services (payment gateways, shipping providers, etc.), verify the request authenticity using HMAC signature, API key header, or IP allowlisting — depending on what the external service supports. Do not trust inbound payloads without verification.

2. **Input validation:** Validate all incoming data from webhooks and API requests before processing. Check required fields, data types, and value ranges. Do not pass unvalidated external input directly to database queries or Fynd API calls.

3. **Secret management:** Never hardcode `EXTENSION_API_KEY`, `EXTENSION_API_SECRET`, `FP_API_DOMAIN`, or any credentials in source code. Always read from environment variables. Never log secrets, tokens, or full request headers that may contain auth data.

4. **CORS configuration:** If the extension frontend is served from a different origin than the backend, configure CORS to allow only your extension's origin. Do not use `cors({ origin: "*" })` in production — it allows any site to call your extension's API.

5. **Session storage is not a database:** FDK's session storage (e.g., SQLiteStorage) holds OAuth tokens. Never read, modify, or query it for business logic. Treat it as opaque infrastructure managed by FDK.

6. **Company isolation:** Every database query must be scoped to `fynd_company_id`. Never return data from one company to another. When building list/search endpoints, always include the company filter in the query — do not filter in application code after fetching all records.

## HMAC signature verification

```js
const crypto = require("crypto");

function verifyWebhookSignature(req, secret, headerName = "x-webhook-signature") {
  const signature = req.headers[headerName];
  if (!signature) return false;

  const payload = JSON.stringify(req.body);
  const expected = crypto
    .createHmac("sha256", secret)
    .update(payload)
    .digest("hex");

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected)
  );
}

// Usage in inbound webhook route
router.post("/payment-callback", (req, res, next) => {
  if (!verifyWebhookSignature(req, process.env.PAYMENT_WEBHOOK_SECRET)) {
    return res.status(401).json({ message: "Invalid webhook signature" });
  }
  next();
}, async (req, res) => {
  // ... process verified webhook
});
```

## Input validation

```js
function validateRequired(body, fields) {
  const missing = fields.filter((f) => body[f] === undefined || body[f] === null);
  if (missing.length > 0) {
    return { valid: false, message: `Missing required fields: ${missing.join(", ")}` };
  }
  return { valid: true };
}

// Usage
router.post("/configure", async (req, res) => {
  const { valid, message } = validateRequired(req.body, ["name", "webhook_url"]);
  if (!valid) {
    return res.status(400).json({ message });
  }
  // ... proceed with validated data
});
```

## CORS configuration

```js
const cors = require("cors");

// Only allow your extension's frontend origin
app.use(cors({
  origin: process.env.EXTENSION_BASE_URL,
  credentials: true,
}));
```

## Logging — what to log and what not to

```js
// GOOD — log identifiers, event types, status
console.log(`[webhook] Processing ${event_name} for company=${company_id}`);
console.log(`[api] GET /products company=${companyId} count=${products.length}`);
console.error(`[error] getPlatformClient failed company=${companyId}`);

// BAD — never log these
console.log(req.headers);              // may contain auth tokens
console.log(process.env);              // contains all secrets
console.log(req.body.api_secret);      // explicit secret
console.log(platformClient);           // contains auth internals
```

## Company-scoped database queries

```js
// CORRECT — company filter in the query itself
const configs = await db.find("configurations", {
  fynd_company_id: companyId,
  application_id: applicationId,
});

// WRONG — fetches all, filters in JS (leaks data, poor performance)
const allConfigs = await db.find("configurations", {});
const configs = allConfigs.filter((c) => c.fynd_company_id === companyId);
```
