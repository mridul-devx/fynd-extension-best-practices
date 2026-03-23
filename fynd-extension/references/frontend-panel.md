# Frontend panel — Rules and patterns

Rules for the extension's frontend panel — the UI merchants interact with inside the Fynd Partner/Company dashboard. Apply these when working on the extension's frontend code.

## Rules

1. **Launch URL structure:** The extension panel is loaded in an iframe at `<EXTENSION_BASE_URL>/company/:company_id`. If the extension is application-scoped, the URL includes the application: `<EXTENSION_BASE_URL>/company/:company_id/application/:application_id`. The frontend must extract `company_id` (and `application_id`) from the URL and pass them to all backend API calls.

2. **Company context in API calls:** Every API call from the frontend to the extension backend must include `company_id` via the `x-company-id` header (or as a query/path parameter). The backend relies on this to scope all Fynd Platform API calls. Never hardcode or cache company IDs.

3. **Application context:** When the extension operates at the application level, also pass `application_id` in API calls. Extract it from the URL path or query string.

4. **Session and auth:** The FDK handler manages OAuth sessions automatically. The frontend does not handle tokens directly — cookies set by `fdkHandler` authenticate requests. Do not implement custom auth flows or store tokens in localStorage.

5. **Relative API paths:** The frontend should call the extension backend at relative paths (e.g., `/api/products`, `/api/configurations`). Do not hardcode `EXTENSION_BASE_URL` in frontend API calls — the iframe base URL handles routing.

6. **No direct Fynd API calls:** The frontend must never call Fynd Platform APIs directly. All platform data must flow through the extension backend, which adds company scoping and auth via `getPlatformClient`.

## URL pattern

```
# Company-level extension
https://your-extension.com/company/12345

# Application-level extension
https://your-extension.com/company/12345/application/abc123
```

## Frontend context extraction

```js
// React / Vue — extract from URL
function getExtensionContext() {
  const pathParts = window.location.pathname.split("/");
  const companyIndex = pathParts.indexOf("company");
  const appIndex = pathParts.indexOf("application");

  return {
    companyId: companyIndex !== -1 ? pathParts[companyIndex + 1] : null,
    applicationId: appIndex !== -1 ? pathParts[appIndex + 1] : null,
  };
}
```

## API call pattern

```js
const { companyId, applicationId } = getExtensionContext();

// Every backend call includes company context
async function fetchFromBackend(path, options = {}) {
  const headers = {
    "Content-Type": "application/json",
    "x-company-id": companyId,
    ...(applicationId && { "x-application-id": applicationId }),
    ...options.headers,
  };

  const response = await fetch(path, { ...options, headers });

  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.message || "Request failed");
  }

  return response.json();
}

// Usage
const products = await fetchFromBackend("/api/products");
const config = await fetchFromBackend("/api/configurations");
```

## Frontend project structure

```
frontend/
├── src/
│   ├── App.jsx / App.vue       # Root — extracts company/application context
│   ├── context/
│   │   └── extension.js        # Provides companyId, applicationId to all components
│   ├── services/
│   │   └── api.js              # Backend API client (includes x-company-id header)
│   ├── pages/
│   │   ├── Dashboard.jsx       # Main extension panel
│   │   ├── Settings.jsx        # Configuration page
│   │   └── ...
│   └── components/
├── dist/                       # Built output — served by Express catch-all
└── package.json
```

## Context provider pattern (React)

```jsx
import { createContext, useContext } from "react";

const ExtensionContext = createContext(null);

export function ExtensionProvider({ children }) {
  const context = getExtensionContext(); // from URL

  return (
    <ExtensionContext.Provider value={context}>
      {children}
    </ExtensionContext.Provider>
  );
}

export function useExtension() {
  const ctx = useContext(ExtensionContext);
  if (!ctx) throw new Error("useExtension must be used within ExtensionProvider");
  return ctx;
}
```

## Context provider pattern (Vue 3)

```js
// composables/useExtension.js
import { reactive, readonly } from "vue";

const state = reactive(getExtensionContext());

export function useExtension() {
  return readonly(state);
}
```
