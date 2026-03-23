# Code examples — Reference patterns

Concrete code patterns for Fynd extension development. Use these as templates when generating or suggesting code.

## FDK setup (`backend/fdk.js`)

```js
const { setupFdk } = require("@gofynd/fdk-extension-javascript/express");
const { SQLiteStorage } = require("@gofynd/fdk-extension-javascript/express/storage");

const fdkExtension = setupFdk({
  api_key: process.env.EXTENSION_API_KEY,
  api_secret: process.env.EXTENSION_API_SECRET,
  base_url: process.env.EXTENSION_BASE_URL,
  callbacks: {
    auth: async (req) => {
      // Return redirect URL after OAuth completes
      return `${req.extension.base_url}/company/${req.query.company_id}`;
    },
    install: async (req) => {
      // Company just installed the extension
      // Safe: create config records, store company mapping
      // Unsafe: do NOT call getPlatformClient here — token may not be stored yet
    },
    uninstall: async (req) => {
      // Company uninstalled the extension
      // Clean up company-specific data, revoke external access
    },
  },
  storage: new SQLiteStorage(
    process.env.DB_PATH || "./db.sqlite",
    "sessions"
  ),
  access_mode: "online",
  cluster: process.env.FP_API_DOMAIN || "https://api.fynd.com",
  webhook_config: {
    api_path: "/api/webhook-events",
    notification_email: "dev@example.com",
    event_map: {
      "company/product/create": {
        handler: handleProductCreate,
        version: "1",
      },
      "application/order/placed": {
        handler: handleOrderPlaced,
        version: "1",
      },
    },
  },
});

module.exports = fdkExtension;
```

## Server file — route mount order (`server.js`)

```js
const express = require("express");
const fdkExtension = require("./backend/fdk");
const inboundWebhookRouter = require("./backend/routes/inbound-webhooks");
const extensionApiRouter = require("./backend/routes/extension-api");

const app = express();
app.use(express.json());

// ── 1. Inbound webhooks (external service → extension) ──
// Must come FIRST so they are not swallowed by fdkHandler
app.use("/api/webhooks", inboundWebhookRouter);

// ── 2. Extension API routes ──
app.use("/api", extensionApiRouter);

// ── 3. Fynd webhook processing ──
app.post("/api/webhook-events", async (req, res) => {
  try {
    await fdkExtension.webhookRegistry.processWebhook(req);
    return res.status(200).json({ success: true });
  } catch (err) {
    console.error("Webhook processing failed:", err);
    return res.status(500).json({ message: "Webhook processing failed" });
  }
});

// ── 4. FDK handler (OAuth, session, catch-all) ──
app.use("/", fdkExtension.fdkHandler);

// ── 5. Platform, partner, and basic routes ──
app.use("/api", fdkExtension.platformApiRoutes);
app.use("/apipartner", fdkExtension.partnerApiRoutes);

// ── 6. SPA catch-all (last) ──
app.get("*", (req, res) => {
  res.sendFile(path.join(__dirname, "frontend", "dist", "index.html"));
});

app.listen(process.env.PORT || 8080);
```

## getPlatformClient — with error handling

```js
async function getClient(req, res) {
  const companyId = req.headers["x-company-id"] || req.query.company_id;

  if (!companyId) {
    return res.status(400).json({ message: "Missing company_id" });
  }

  try {
    const platformClient = await fdkExtension.getPlatformClient(companyId);
    return platformClient;
  } catch (err) {
    console.error(`getPlatformClient failed for company ${companyId}:`, err.message);
    res.status(401).json({ message: "Company not authenticated. Reinstall the extension." });
    return null;
  }
}

// Usage in a route handler
router.get("/products", async (req, res) => {
  const platformClient = await getClient(req, res);
  if (!platformClient) return; // response already sent

  const products = await platformClient.catalog.getProducts({ pageSize: 10 });
  return res.json(products);
});
```

## Application-scoped vs company-scoped API

```js
// Company-scoped — affects all applications under the company
const products = await platformClient.catalog.getProducts();
const orders = await platformClient.order.getOrders();

// Application-scoped — affects one specific storefront application
const applicationId = req.query.application_id;
const appProducts = await platformClient
  .application(applicationId)
  .catalog.getAppProducts();
const appConfig = await platformClient
  .application(applicationId)
  .configuration.getAppBasicDetails();

// Rule: Use application-scoped when the data belongs to a specific storefront.
// Use company-scoped when the data spans all storefronts (e.g. master catalog).
```

## Webhook handler

```js
async function handleProductCreate(event_name, request_body, company_id, application_id) {
  // Always use company_id and application_id from the arguments — not globals
  console.log(`[webhook] ${event_name} for company=${company_id} app=${application_id}`);

  try {
    // For heavy work, acknowledge and process async
    await db.insert("webhook_events", {
      event_name,
      company_id,
      application_id,
      payload: JSON.stringify(request_body),
      status: "pending",
      created_at: new Date(),
    });

    // Optionally call Fynd APIs using company_id
    const platformClient = await fdkExtension.getPlatformClient(company_id);
    if (platformClient) {
      // ... process with platform client
    }
  } catch (err) {
    // Log but don't throw — Fynd expects a timely 200
    console.error(`[webhook] Error handling ${event_name}:`, err);
  }
}
```

## Extension entity filtering

```js
router.get("/schemes", async (req, res) => {
  const platformClient = await getClient(req, res);
  if (!platformClient) return;

  const allSchemes = await platformClient.serviceability.getSchemes();

  // Filter to only this extension's schemes
  const mySchemes = allSchemes.items.filter(
    (scheme) => scheme.extension_id === process.env.EXTENSION_API_KEY
  );

  return res.json({ items: mySchemes });
});
```

## Inbound webhook (external → extension)

```js
// backend/routes/inbound-webhooks.js
const router = require("express").Router();
const fdkExtension = require("../fdk");
const db = require("../db");

router.post("/payment-callback", async (req, res) => {
  // 1. Resolve companyId from payload or DB
  const externalRef = req.body.reference_id;
  const mapping = await db.findOne("company_mappings", { external_ref: externalRef });

  if (!mapping) {
    return res.status(400).json({ message: "Cannot resolve company for this callback" });
  }

  // 2. Get platform client
  const platformClient = await fdkExtension.getPlatformClient(mapping.fynd_company_id);
  if (!platformClient) {
    return res.status(401).json({ message: "Company not authenticated" });
  }

  // 3. Process and respond
  await platformClient.order.updatePaymentStatus({ ... });
  return res.status(200).json({ success: true });
});

module.exports = router;
```
