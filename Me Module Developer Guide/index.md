---
title: ShareRing Me Modules Developer Guide
post_type: page
menu_order: 3
post_status: publish
post_excerpt: Guide for building embedded web applications (Modules) inside the ShareRing Me app.
taxonomy:
    category:
        - developer
---

# ShareRing Me Modules Developer Guide

This guide is for **external developers** who want to build a **Me Module** for the ShareRing Me app.

- A **Me Module** is a **web application** that runs inside the ShareRing Me mobile app (in an embedded WebView).
- A Me Module communicates with the ShareRing Me app **only** via a **message bridge** (`postMessage` and `addEventListener`).
- This document covers the **Me Module API**.

---

## What you build (high-level)

- A static web app (React/Vue/Svelte/Vanilla JS—your choice)
- Hosted at a public URL (typically `https://...`)
- With a required `manifest.json` at the **domain root**
- Optionally packaged for **offline caching** using a zip bundle

---

## Quickstart (recommended scaffolding)

You can use any stack. For best results (TypeScript + fast iteration + predictable build output) use **Vite + React + TypeScript**.

### 1) Scaffold a module

```bash
npm create vite@latest sharering-me-module -- --template react-ts
cd sharering-me-module
npm install
```

### 2) Make the build work from `file://` (required for offline mode)

When ShareRing Me loads an offline bundle it loads your `index.html` from a local file path, so **asset URLs must be relative**.

Update `vite.config.ts`:

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  // Critical: ensures assets resolve when loaded from a local file path
  base: "./",
});
```

If you use client-side routing, note that:
- **Online mode**: You can use either hash routing (e.g. `/#/route`) or path routing.
- **Offline mode**: You must use hash routing (e.g. `/#/route`) because your module will be loaded from local file paths.

### 3) Add a small bridge helper

Create `src/shareringMeBridge.ts`:

```ts
export type MeModuleResponse = {
  type: string; // ALWAYS returned uppercase by the app
  payload: any;
  error?: unknown;
};

/**
 * Creates a safe bridge for Me Modules.
 *
 * Important constraints of the platform:
 * - Responses can arrive out of order (async handlers).
 *
 * This bridge enforces ONE in-flight request at a time.
 */
export function createShareRingMeBridge() {
  let inFlight:
    | {
        expectedType: string;
        resolve: (value: any) => void;
        reject: (err: Error) => void;
        timeoutId: number;
      }
    | null = null;

  const queue: Array<{
    expectedType: string;
    payload: any;
    timeoutMs: number;
    resolve: (value: any) => void;
    reject: (err: Error) => void;
  }> = [];

  function parseIncoming(data: any): MeModuleResponse | null {
    if (!data) return null;
    try {
      return JSON.parse(data);
    } catch {
      return null;
    }
  }

  function onMessage(event: MessageEvent) {
    if (event.type !== "message") return;
    const msg = parseIncoming((event as any).data);
    if (!msg || !msg.type) return;

    const msgType = String(msg.type).toUpperCase();
    if (!inFlight) return;
    if (msgType !== inFlight.expectedType) return;

    window.clearTimeout(inFlight.timeoutId);
    const { resolve, reject } = inFlight;
    inFlight = null;

    if (msg.error) reject(new Error(String(msg.error)));
    else resolve(msg.payload);

    flush();
  }

  window.addEventListener("message", onMessage, true);

  function flush() {
    if (inFlight || queue.length === 0) return;
    const next = queue.shift()!;
    inFlight = {
      expectedType: next.expectedType,
      resolve: next.resolve,
      reject: next.reject,
      timeoutId: window.setTimeout(() => {
        inFlight = null;
        next.reject(new Error(`Timeout waiting for ${next.expectedType}`));
        flush();
      }, next.timeoutMs),
    };

    (window as any).ReactNativeWebView?.postMessage(
      JSON.stringify({
        type: next.expectedType,
        payload: next.payload,
      })
    );
  }

  function send(type: string, payload?: any, opts?: { timeoutMs?: number }) {
    const expectedType = String(type).toUpperCase();
    const timeoutMs = opts?.timeoutMs ?? 30_000;

    return new Promise<any>((resolve, reject) => {
      queue.push({ expectedType, payload, timeoutMs, resolve, reject });
      flush();
    });
  }

  function destroy() {
    window.removeEventListener("message", onMessage, true);
  }

  return { send, destroy };
}
```

### 4) Use the bridge in your UI

Example `src/App.tsx`:

```tsx
import { useMemo, useState } from "react";
import { createShareRingMeBridge } from "./shareringMeBridge";

export default function App() {
  const mm = useMemo(() => createShareRingMeBridge(), []);
  const [result, setResult] = useState<any>(null);
  const [error, setError] = useState<string>("");

  async function readAppInfo() {
    setError("");
    setResult(null);
    try {
      const payload = await mm.send("COMMON_APP_INFO");
      setResult(payload);
    } catch (e: any) {
      setError(e?.message ?? String(e));
    }
  }

  return (
    <div style={{ padding: 16, fontFamily: "system-ui" }}>
      <h2>Hello Me Module (V2)</h2>
      <button onClick={readAppInfo}>COMMON_APP_INFO</button>
      {error ? <pre style={{ color: "crimson" }}>{error}</pre> : null}
      {result ? <pre>{JSON.stringify(result, null, 2)}</pre> : null}
    </div>
  );
}
```

### 5) Add `manifest.json` (required)

Your module must serve a **ShareRing manifest** at `/<manifest.json>`.

If you use Vite, create `public/manifest.json` so it is available at:
- dev: `http://localhost:5173/manifest.json`
- prod build: `dist/manifest.json`

Minimal example (online-only):

```json
{
  "version": "0.0.1",
  "offline_mode": false,
  "isMaintenance": false,
  "enable_secure_screen": false
}
```

### 6) Build & host

```bash
npm run build
```

Host the `dist/` folder (any static hosting works).

---

## Testing inside ShareRing Me (Developer Mode → Custom dApps)

To test a module during development:

1. Ensure your ShareRing Me user has **Developer Mode** enabled (this is typically enabled per user/account).
2. In the app, go to **Settings → Developer Tool → Add Custom dApps**.
3. Paste your module URL and save.
   - If you’re using a local dev server, use a URL reachable from the device (LAN IP or a tunnel).
4. Open it from the same area (or wherever your build exposes it in the app UI).

---

## Required hosting

**Your module MUST serve both `index.html` and `manifest.json` at the domain root** of the host.

**Important**: Me modules do not support hosting in subpaths. You must use a dedicated domain or subdomain for your module.

Examples:

- `https://example.com` → `index.html` and `manifest.json` at `https://example.com/`
- `https://module.example.com` → `index.html` and `manifest.json` at `https://module.example.com/`

If `manifest.json` is missing or invalid, the ShareRing Me app can refuse to load the module (especially on first open).

### Manifest schema

The ShareRing Me app uses a PWA manifest with some additional attributes.

Recommended fields:

- `version` (string): version identifier. Bump this when you deploy a new build.
- `offline_mode` (boolean): enable offline zip caching.
- `zip_name` (string): zip file name (required when `offline_mode: true`).
- `checksum` (string, optional): checksum for the zip file (see below).
- `isMaintenance` (boolean, optional): if true, the app may show a maintenance page.
- `enable_secure_screen` (boolean, optional): request screenshot prevention while your module is open.

### Minimal example (online-only)

```json
{
  "version": "1.0.0",
  "offline_mode": false,
  "isMaintenance": false,
  "enable_secure_screen": false
}
```

### Offline-enabled example

```json
{
  "version": "1.0.3",
  "offline_mode": true,
  "zip_name": "build-1.0.3.zip",
  "checksum": "PUT_SHA256_OF_BASE64_ZIP_HERE",
  "isMaintenance": false,
  "enable_secure_screen": true
}
```

---

## Offline mode (zip bundle) (optional — use only when you need it)

Offline mode lets the ShareRing Me app download a zip build and load it locally.

Offline mode is powerful, but it has real trade-offs:
- It increases **first-run bandwidth** (the zip must be downloaded).
- It increases **device storage usage** (the extracted bundle is cached locally).
- It can increase **memory pressure** (unzipping + loading larger local assets).

### When offline mode is useful

- **Low / unreliable bandwidth**: users can still open and use the module after a successful initial cache.
- **No mobile service**: modules that must work in-flight/underground/remote areas.
- **Stateful/offline-first workflows**: e.g. ticketing / scanning / check-in flows that must keep working and sync later.

### When to avoid offline mode

- The module is used infrequently (downloading a zip is wasted bandwidth/storage).
- The module changes often (users will download many versions over time).
- The module is mostly static content that works fine online.
- The module requires real-time data or frequent API calls (offline caching provides no benefit).

### 1) Zip download URL

The app downloads the zip from:

```
<moduleUrl>/<zip_name>
```

Examples:
- module URL `https://example.com` + `zip_name: "build-1.0.3.zip"`
  → `https://example.com/build-1.0.3.zip`
- module URL `https://example.com/my-module` + `zip_name: "build-1.0.3.zip"`
  → `https://example.com/my-module/build-1.0.3.zip`

### 2) Zip file contents (critical)

When unzipped, the app expects:
- `index.html` at the **root of the extracted folder**
- your static assets referenced by **relative paths**

Do **not** ship a zip that nests everything inside another folder level.

Good zip (root):
```
index.html
assets/...
manifest.json   (recommended to include too)
```

Bad zip (nested):
```
my-build/
  index.html
  assets/...
```

### 3) Checksum format

If you provide `checksum`, the app verifies it as:

1. read the zip file as a **base64 string**
2. compute `sha256(base64String)` → hex

Example command (Node.js) to generate this checksum:

```bash
node -e "const fs=require('fs'); const crypto=require('crypto'); const b64=fs.readFileSync('build-1.0.3.zip').toString('base64'); console.log(crypto.createHash('sha256').update(b64).digest('hex'));"
```

### 4) Update process (recommended)

When deploying a new version:

1. Build your web app (`npm run build`).
2. Zip the build output with `index.html` at the zip root.
3. Upload the zip to `<moduleUrl>/<zip_name>`.
4. Update the domain-root `manifest.json`:
   - bump `version`
   - set `zip_name` to the new file
   - set `checksum` if used

---

## Messaging protocol

### Message envelopes

All messages MUST be JSON objects.

**Request (WebView → App)**

```js
type Request = { type: string; payload?: unknown };
```

**Response (App → WebView)**

```js
type Response = { type: string; payload: unknown; error?: unknown };
```

### Conventions

- `type` is an **event name** (string literal), e.g. `'COMMON_APP_INFO'`.
- `payload` varies by event:
  - scalar (e.g. `string`, `boolean`)
  - object
  - array
- `error` is optional and only present when the native handler fails.

### Web → App (request)

```js
window.ReactNativeWebView?.postMessage(JSON.stringify({
  type: 'EVENT_TYPE',
  payload: { /* your data (optional) */ }
}));
```

### App → Web (response)

```js
const handleMessage = (event: MessageEvent) => {
  if (event.type !== "message") return;
  const msg = JSON.parse(event.data);
  // msg.type, msg.payload, msg.error 
}
window.addEventListener("message", handleMessage, true);

// later on remove the listener to avoid memory leaks and/or collisions
// window.removeEventListener('message', handleMessage);
```

---

## User confirmation (PIN) for sensitive operations

Some calls require the ShareRing Me app to show a **PIN confirmation** UI to the user. Your module must handle:
- a delay (user is interacting)
- the user cancelling (you’ll receive an error)

PIN confirmation is required for:
- `CRYPTO_DECRYPT`
- `CRYPTO_SIGN`
- `WALLET_SIGN_TRANSACTION`
- `WALLET_SIGN_AND_BROADCAST_TRANSACTION`
- `VAULT_EXEC_QUERY_SILENT`

---

## API Reference

### Conventions used below

- "Request" means your module sends `{ type, payload }`.
- "Response payload" means what you’ll receive as `msg.payload` in the response.
- On failure you’ll receive `msg.error` (string).
- Some events are **fire-and-forget** and do not respond.

### Events index

| Category | Event `type` | Direction |
|---|---|---:|
| COMMON | `COMMON_APP_INFO` | MM → App |
| COMMON | `COMMON_DEVICE_INFO` | MM → App |
| COMMON | `COMMON_READ_ASYNC_STORAGE` | MM → App |
| COMMON | `COMMON_WRITE_ASYNC_STORAGE` | MM → App |
| COMMON | `COMMON_STATUS_BAR_DIMENSIONS` | MM → App |
| COMMON | `COMMON_SET_STATUS_BAR_STYLE` | MM → App |
| COMMON | `COMMON_COPY_TO_CLIPBOARD` | MM → App |
| COMMON | `COMMON_OPEN_BROWSER` | MM → App |
| NAVIGATION | `NAVIGATE_TO` | MM → App |
| NAVIGATION | `NAVIGATE_BACK` | MM → App |
| NAVIGATION | `NAVIGATE_IS_FOCUSED` | MM → App |
| NAVIGATION | `NAVIGATE_OPEN_DEVICE_SETTINGS` | MM → App |
| NAVIGATION | `NAVIGATE_OPEN_LINK` | MM → App |
| VAULT | `VAULT_DOCUMENTS` | MM → App |
| VAULT | `VAULT_EMAIL` | MM → App |
| VAULT | `VAULT_AVATAR` | MM → App |
| VAULT | `VAULT_ADD_DOCUMENT` | MM → App |
| VAULT | `VAULT_ADD_CUSTOM_VALUE` | MM → App |
| VAULT | `VAULT_EXEC_QUERY` | MM → App |
| VAULT | `VAULT_EXEC_QUERY_SILENT` | MM → App |
| WALLET | `WALLET_MAIN_ACCOUNT` | MM → App |
| WALLET | `WALLET_CURRENT_ACCOUNT` | MM → App |
| WALLET | `WALLET_BALANCE` | MM → App |
| WALLET | `WALLET_ACCOUNTS` | MM → App |
| WALLET | `WALLET_SWITCH_ACCOUNT` | MM → App |
| WALLET | `WALLET_SIGN_TRANSACTION` | MM → App |
| WALLET | `WALLET_SIGN_AND_BROADCAST_TRANSACTION` | MM → App |
| WALLET | `WALLET_SWAP_ACCOUNT` | MM → App |
| NFT | `NFT_NFTS` | MM → App |
| CRYPTOGRAPHY | `CRYPTO_ENCRYPT` | MM → App |
| CRYPTOGRAPHY | `CRYPTO_DECRYPT` | MM → App |
| CRYPTOGRAPHY | `CRYPTO_SIGN` | MM → App |
| CRYPTOGRAPHY | `CRYPTO_VERIFY` | MM → App |
| GOOGLE_WALLET | `GOOGLE_WALLET_CAN_ADD_PASSES` | MM → App |
| GOOGLE_WALLET | `GOOGLE_WALLET_ADD_PASS` | MM → App |
| APPLE_WALLET | `APPLE_WALLET_CAN_ADD_PASSES` | MM → App |
| APPLE_WALLET | `APPLE_WALLET_ADD_PASS` | MM → App |
| APPLE_WALLET | `APPLE_WALLET_HAS_PASS` | MM → App |
| APPLE_WALLET | `APPLE_WALLET_REMOVE_PASS` | MM → App |
| APPLE_WALLET | `APPLE_WALLET_VIEW_PASS` | MM → App |

### Error handling

When the App cannot fulfill a request, it will respond with:

```json
{
  "type": "<EVENT_NAME>",
  "error": "..."
}
```

Notes:
- `error` is intentionally `unknown` to avoid constraining native implementations. It can be a message string or a generic error object.

---

### Event details

## COMMON

### `COMMON_APP_INFO`

Get basic app context for localization/theme.

#### Request

```json
{ "type": "COMMON_APP_INFO" }
```

#### Response

```js
{
  type: "COMMON_APP_INFO";
  payload: {
    language: string;
    version: string;
    id: string;
    darkMode: boolean;
  };
  error?: unknown;
}
```

**Response payload fields**

| Field | Type | Description |
|---|---|---|
| `language` | `string` | Current app language/locale. |
| `version` | `string` | App version string. |
| `id` | `string` | App-specific identifier. |
| `darkMode` | `boolean` | Whether dark mode is enabled. |

### `COMMON_DEVICE_INFO`

#### Request

```json
{ "type": "COMMON_DEVICE_INFO" }
```

#### Response

```js
{
  type: "COMMON_DEVICE_INFO";
  payload: {
    identifier: string;
    brand: string;
    model: string;
    os: string;
    country: string;
    timezone: string;
  };
  error?: unknown;
}
```

**Response payload fields**

| Field | Type | Description |
|---|---|---|
| `identifier` | `string` | Device identifier. |
| `brand` | `string` | Device brand/manufacturer. |
| `model` | `string` | Device model. |
| `os` | `string` | OS name/version. |
| `country` | `string` | Device country. |
| `timezone` | `string` | IANA timezone, e.g. `Asia/Ho_Chi_Minh`. |

### `COMMON_STATUS_BAR_DIMENSIONS`

Use to position UI under safe areas.

#### Request

```json
{ "type": "COMMON_STATUS_BAR_DIMENSIONS" }
```

#### Response

```js
{
  type: "COMMON_STATUS_BAR_DIMENSIONS";
  payload: {
    top: number;
    left: number;
    width: number;
    height: number;
  };
  error?: unknown;
}
```

### `COMMON_SET_STATUS_BAR_STYLE`

#### Request

```js
{
  type: "COMMON_SET_STATUS_BAR_STYLE";
  payload: "light" | "dark";
}
```

#### Response

```js
{
  type: "COMMON_SET_STATUS_BAR_STYLE";
  payload: null;
  error?: unknown;
}
```

### `COMMON_COPY_TO_CLIPBOARD`

#### Request

```js
{
  type: "COMMON_COPY_TO_CLIPBOARD";
  payload: {
    content: string;
    showToastNotification?: boolean;
  };
}
```

#### Response

```js
{
  type: "COMMON_COPY_TO_CLIPBOARD";
  payload: null;
  error?: unknown;
}
```

**Request payload fields**

| Field | Type | Description |
|---|---|---|
| `content` | `string` | Text to copy. |
| `showToastNotification` | `boolean` | Whether to show the default "copied" toast/notification. Defaults to `true`. |

### `COMMON_OPEN_BROWSER`

Opens an in-app browser for a URL. To pass data back, redirect to `sharering://close?...`.

#### Request

```js
{
  type: "COMMON_OPEN_BROWSER";
  payload: string; // URL
}
```

#### Response

```js
{
  type: "COMMON_OPEN_BROWSER";
  payload: Record<string, any>;
  error?: unknown;
}
```

**Notes**
- Response `payload` contains the parsed query string parameters as a key-value object when data is passed back on browser close event

### `COMMON_READ_ASYNC_STORAGE`

Read app-managed key/value storage **scoped to your module domain**.

#### Request

```js
{
  type: "COMMON_READ_ASYNC_STORAGE";
  payload: string | string[];
}
```

#### Response

```js
{
  type: "COMMON_READ_ASYNC_STORAGE";
  payload: Record<string, any>;
  error?: unknown;
}
```

**Notes**
- `payload` can be a string key or an array of keys.
- Response `payload` is a key-value object; missing/unavailable keys are omitted.

### `COMMON_WRITE_ASYNC_STORAGE`

Write app-managed key/value storage **scoped to your module domain**.

#### Request

```js
{
  type: "COMMON_WRITE_ASYNC_STORAGE";
  payload: {
    [key: string]: any;
  };
}
```

#### Response

```js
{
  type: "COMMON_WRITE_ASYNC_STORAGE";
  payload: null;
  error?: unknown;
}
```

---

## NAVIGATION

### `NAVIGATE_TO`

Navigate within the ShareRing Me app.

#### Request

```js
{
  type: "NAVIGATE_TO";
  payload: {
    to: string;
    params?: Record<string, any>;
    mode?: "replace" | "push";
  };
}
```

**Notes**
- `to` can be an in-app screen name or another Me Modules (by its domain name).
- `mode`:
  - `replace`: replaces the current route
  - `push`: adds a new route to the navigation stack
  - `pop`: navigates back to the previous route specified by `to`. If the route is not found in the stack, it replaces the current route with `to`

### `NAVIGATE_BACK`

#### Request

```js
{
  type: "NAVIGATE_BACK";
  payload: {
    steps?: number;
  };
}
```

#### Response

```js
{
  type: "NAVIGATE_BACK";
  payload: null;
  error?: unknown;
}
```

**Notes**
- `steps` defaults to `1` and can be used to go back multiple routes.

### `NAVIGATE_IS_FOCUSED`

#### Request

```json
{ "type": "NAVIGATE_IS_FOCUSED" }
```

#### Response

```js
{
  type: "NAVIGATE_IS_FOCUSED";
  payload: boolean;
  error?: unknown;
}
```

### `NAVIGATE_OPEN_DEVICE_SETTINGS`

#### Request

```js
{
  type: "NAVIGATE_OPEN_DEVICE_SETTINGS";
  payload: string;
}
```

**Notes**
- `payload` can be one of: `general`, `location`, `bluetooth`, `network`, `wifi`, `cellular`, `app`, `apps`.

### `NAVIGATE_OPEN_LINK`

Opens a URL or deeplink via the OS.

#### Request

```js
{
  type: "NAVIGATE_OPEN_LINK";
  payload: string; // deeplink or URL
}
```

**Notes**
- The app triggers `Linking.openURL(...)`; OS handles the target scheme.

---

## VAULT

### `VAULT_DOCUMENTS`

#### Request

```json
{ "type": "VAULT_DOCUMENTS" }
```

#### Response

```js
{
  type: "VAULT_DOCUMENTS";
  payload: Array<{
    id: string;
    type: string;
    country: string;
  }>;
  error?: unknown;
}
```

**Response payload fields (per item)**

| Field | Type | Description |
|---|---|---|
| `id` | `string` | Document id. |
| `type` | `string` | Document type. |
| `country` | `string` | Document country. |

### `VAULT_EMAIL`

#### Request

```json
{ "type": "VAULT_EMAIL" }
```

#### Response

```js
{
  type: "VAULT_EMAIL";
  payload: string;
  error?: unknown;
}
```

### `VAULT_AVATAR`

#### Request

```json
{ "type": "VAULT_AVATAR" }
```

#### Response

```js
{
  type: "VAULT_AVATAR";
  payload: string;
  error?: unknown;
}
```

**Notes**
- `payload` is the base64 encoded image string

### `VAULT_ADD_DOCUMENT`

Add a document to the user’s vault.

#### Request

```js
{
  type: "VAULT_ADD_DOCUMENT";
  payload: {
    image: string;
    photo: string;
    metadata: {
      type: string;
      expiryDate: string;
      issueDate: string;
      number: string;
      country: string;
      countryCode: string;
      fullName: string;
      dob: string;
      address: string;
      nationality: string;
      placeOfBirth: string;
      issueBy: string;
    };
  };
}
```

#### Response

```js
{
  type: "VAULT_ADD_DOCUMENT";
  payload: null;
  error?: unknown;
}
```

### `VAULT_ADD_CUSTOM_VALUE`

#### Request

```js
{
  type: "VAULT_ADD_CUSTOM_VALUE";
  payload: {
    [key: string]: any;
  };
}
```

#### Response

```js
{
  type: "VAULT_ADD_CUSTOM_VALUE";
  payload: null;
  error?: unknown;
}
```

### `VAULT_EXEC_QUERY`

Starts a vault query / data sharing flow.

#### Request

```js
{
  type: "VAULT_EXEC_QUERY";
  payload: {
    queryId: string;
    clientId: string;
    ownerAddress: string;
    sessionId?: string;
    customValue?: Array<{ name: string; value: string }>;
  };
}
```

#### Response

```js
{
  type: "VAULT_EXEC_QUERY";
  payload: Array<{
    name: string;
    label: string;
    value: string;
  }>;
  error?: unknown;
}
```

### `VAULT_EXEC_QUERY_SILENT`

Same request shape as `VAULT_EXEC_QUERY`, but the app may require PIN confirmation.

#### Request

```js
{
  type: "VAULT_EXEC_QUERY_SILENT";
  payload: {
    queryId: string;
    clientId: string;
    ownerAddress: string;
    sessionId?: string;
    customValue?: Array<{ name: string; value: string }>;
  };
}
```

#### Response

```js
{
  type: "VAULT_EXEC_QUERY_SILENT";
  payload: Array<{
    name: string;
    label: string;
    value: string;
  }>;
  error?: unknown;
}
```

---

## WALLET

### `WALLET_MAIN_ACCOUNT`

#### Request

```json
{ "type": "WALLET_MAIN_ACCOUNT" }
```

#### Response

```js
{
  type: "WALLET_MAIN_ACCOUNT";
  payload: {
    address: string;
    pubKey: string;
  };
  error?: unknown;
}
```

### `WALLET_CURRENT_ACCOUNT`

#### Request

```json
{ "type": "WALLET_CURRENT_ACCOUNT" }
```

#### Response

```js
{
  type: "WALLET_CURRENT_ACCOUNT";
  payload: {
    address: string;
    pubKey: string;
  };
  error?: unknown;
}
```

### `WALLET_BALANCE`

#### Request

```json
{ "type": "WALLET_BALANCE" }
```

#### Response

```js
{
  type: "WALLET_BALANCE";
  payload: Array<{
    amount: string;
    denom: string;
  }>;
  error?: unknown;
}
```

### `WALLET_ACCOUNTS`

#### Request

```json
{ "type": "WALLET_ACCOUNTS" }
```

#### Response

```js
{
  type: "WALLET_ACCOUNTS";
  payload: Array<{
    address: string;
    pubKey: string;
  }>;
  error?: unknown;
}
```

### `WALLET_SWITCH_ACCOUNT`

#### Request

```json
{ "type": "WALLET_SWITCH_ACCOUNT" }
```

#### Response

```js
{
  type: "WALLET_SWITCH_ACCOUNT";
  payload: {
    address: string;
    pubKey: string;
  };
  error?: unknown;
}
```

**Notes**
- Response `payload` is the account that was switched to.

### `WALLET_SIGN_TRANSACTION`

Signs a ShareLedger transaction.

#### Request

```js
{
  type: "WALLET_SIGN_TRANSACTION";
  payload: {
    messages: string; // hex string of TX messages
    memo?: string;
    fee: {
      amount?: Array<{ amount: string; denom: string }>;
      gas?: number;
      gasPrice?: string; // e.g. "1000nshr"
      granter?: string;
      payer?: string;
    };
  };
}
```

#### Response

```js
{
  type: "WALLET_SIGN_TRANSACTION";
  payload: string; // signature (or signed result)
  error?: unknown;
}
```

### `WALLET_SIGN_AND_BROADCAST_TRANSACTION`

Signs a ShareLedger transaction then commit it to the blockchain.

#### Request

```js
{
  type: "WALLET_SIGN_AND_BROADCAST_TRANSACTION";
  payload: {
    messages: string; // hex string of TX messages
    memo?: string;
    fee: {
      amount?: Array<{ amount: string; denom: string }>;
      gas?: number;
      gasPrice?: string;
      granter?: string;
      payer?: string;
    };
  };
}
```

#### Response

```js
{
  type: "WALLET_SIGN_AND_BROADCAST_TRANSACTION";
  payload: {
    height: number;
    transactionHash: string;
    gasUsed: number;
    gasWanted: number;
    code: string;
  };
  error?: unknown;
}
```

**How to create a ShareLedger transaction**

For `WALLET_SIGN_TRANSACTION` and `WALLET_SIGN_AND_BROADCAST_TRANSACTION`, the Me Module needs to create a transaction and pass it to the app for signing. Here's a code example showing how to create a transaction:

```js
import { ShareledgerSigningClient } from "@shareledgerjs/client";

// 1. Create a client instance
const client = await ShareledgerSigningClient.connect("https://rpc.explorer.shareri.ng"); // RPC endpoint
// 2. Create a transaction message (example: transfer transaction)
const msg = client.bank.send("<from address>", "<to address>", [{ amount: "1000000000", denom: "nshr" }]);
// 3. Encode the transaction to get a buffer
const msgEncoded = client.registry.encodeTxBody({ messages: [msg] });
// 4. Convert to hex string for passing to the app
const msgEncodedHex = Buffer.from(msgEncoded).toString('hex'); // Pass this hex string to the app

```


### `WALLET_SWAP_ACCOUNT`

#### Request

```js
{
  type: "WALLET_SWAP_ACCOUNT";
  payload: {
    network: "eth" | "bsc";
  };
}
```

#### Response

```js
{
  type: "WALLET_SWAP_ACCOUNT";
  payload: {
    network: "eth" | "bsc";
    address: string;
  };
  error?: unknown;
}
```

---

## NFT

### `NFT_NFTS`

#### Request

```json
{ "type": "NFT_NFTS" }
```

#### Response

```js
{
  type: "NFT_NFTS";
  payload: any[];
}
```

---

## CRYPTOGRAPHY

### `CRYPTO_ENCRYPT`

#### Request

```js
{
  type: "CRYPTO_ENCRYPT";
  payload: string;
}
```

#### Response

```js
{
  type: "CRYPTO_ENCRYPT";
  payload: string;
  error?: unknown;
}
```

### `CRYPTO_DECRYPT`

#### Request

```js
{
  type: "CRYPTO_DECRYPT";
  payload: string;
}
```

#### Response

```js
{
  type: "CRYPTO_DECRYPT";
  payload: string;
  error?: unknown;
}
```

### `CRYPTO_SIGN`

#### Request

```js
{
  type: "CRYPTO_SIGN";
  payload: {
    data: string;
    signOptions: {
      delimiter: string;
      expiration: {
        enabled: boolean;
        salt?: string;
      };
    };
  };
}
```

#### Response

```js
{
  type: "CRYPTO_SIGN";
  payload: {
    signature: string;
    data: string;
  };
  error?: unknown;
}
```

### `CRYPTO_VERIFY`

#### Request

```js
{
  type: "CRYPTO_VERIFY";
  payload: {
    data: string;
    verifyOptions: {
      signature?: string;
      delimiter: string;
      expiration: {
        enabled: boolean;
        time: number; // seconds
        salt?: string;
      };
    };
  };
}
```

#### Response

```js
{
  type: "CRYPTO_VERIFY";
  payload: {
    valid: boolean;
    data: string[];
  };
  error?: unknown;
}
```

---

## GOOGLE_WALLET

### `GOOGLE_WALLET_CAN_ADD_PASSES`

#### Request

```json
{ "type": "GOOGLE_WALLET_CAN_ADD_PASSES" }
```

#### Response

```js
{
  type: "GOOGLE_WALLET_CAN_ADD_PASSES";
  payload: boolean;
  error?: unknown;
}
```

### `GOOGLE_WALLET_ADD_PASS`

#### Request

```js
{
  type: "GOOGLE_WALLET_ADD_PASS";
  payload: string; // JWT string or URL
}
```

#### Response

```js
{
  type: "GOOGLE_WALLET_ADD_PASS";
  payload: null;
  error?: unknown;
}
```

---

## APPLE_WALLET

### `APPLE_WALLET_CAN_ADD_PASSES`

#### Request

```json
{ "type": "APPLE_WALLET_CAN_ADD_PASSES" }
```

#### Response

```js
{
  type: "APPLE_WALLET_CAN_ADD_PASSES";
  payload: boolean;
  error?: unknown;
}
```

### `APPLE_WALLET_ADD_PASS`

#### Request

```js
{
  type: "APPLE_WALLET_ADD_PASS";
  payload: string; // likely URL/JWT (platform-specific)
}
```

#### Response

```js
{
  type: "APPLE_WALLET_ADD_PASS";
  payload: null;
  error?: unknown;
}
```

### `APPLE_WALLET_HAS_PASS`

#### Request

```js
{
  type: "APPLE_WALLET_HAS_PASS";
  payload: {
    cardIdentifier: string;
    serialNumber?: string;
  };
}
```

#### Response

```js
{
  type: "APPLE_WALLET_HAS_PASS";
  payload: boolean;
  error?: unknown;
}
```

### `APPLE_WALLET_REMOVE_PASS`

#### Request

```js
{
  type: "APPLE_WALLET_REMOVE_PASS";
  payload: {
    cardIdentifier: string;
    serialNumber?: string;
  };
}
```

#### Response

```js
{
  type: "APPLE_WALLET_REMOVE_PASS";
  payload: boolean;
  error?: unknown;
}
```

### `APPLE_WALLET_VIEW_PASS`

#### Request

```js
{
  type: "APPLE_WALLET_VIEW_PASS";
  payload: {
    cardIdentifier: string;
    serialNumber?: string;
  };
}
```

#### Response

```js
{
  type: "APPLE_WALLET_VIEW_PASS";
  payload: null;
  error?: unknown;
}
```

---

## Best practices & common pitfalls

### 1) Always validate messages

On receive, validate:
- `type` exists and is a string
- `payload` shape matches what your code expects

### 2) Treat the bridge like an RPC channel

- Don’t fire many concurrent requests without a strategy.
- The safest approach is sequential RPC (the provided bridge helper).

### 3) Keep storage small and non-sensitive

`COMMON_WRITE_ASYNC_STORAGE` is for small preferences/state.
Do not store secrets. Do not assume it is backed up.

### 4) Offline mode: make all asset paths relative

If your module must work offline, validate by loading the built `dist/index.html` from disk in a browser and ensuring assets resolve correctly.

### 5) Be a good mobile web citizen

#### Design principles (mobile-first)

- **Responsive layout by default**: avoid fixed widths/heights; use flexible layouts (Flex/Grid), `max-width`, and responsive spacing.
- **Safe areas / notches**: ensure content isn’t hidden behind rounded corners/notches. Consider using:

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
```

and CSS like:

```css
/* Example: keep content away from device cutouts */
.page {
  padding-top: env(safe-area-inset-top);
  padding-right: env(safe-area-inset-right);
  padding-bottom: env(safe-area-inset-bottom);
  padding-left: env(safe-area-inset-left);
}
```

- **Touch targets**: design for thumbs; keep interactive elements comfortably sized and spaced (avoid tiny icons with no padding).
- **Keyboard & forms**: expect the on-screen keyboard to cover parts of the page; ensure focused inputs scroll into view and primary actions remain reachable.
- **Dark mode & language**: respect app theme (`COMMON_APP_INFO.darkMode`) and language (`COMMON_APP_INFO.language`).
- **Accessibility**: good contrast, visible focus states, labels for inputs, and sensible heading structure. Support reduced motion where possible.
- **Performance on mid-range devices**: avoid heavy animations and huge bundles; lazy-load large features, compress images, and keep JS work per frame small.

#### Testing checklist (practical)

- **Screen sizes**: small phone, large phone, and tablet; portrait + landscape.
- **Text scaling**: increase system font size / display size and verify layout doesn’t clip or overlap.
- **Keyboard behavior**: test every form field; ensure it’s never obscured and the page doesn’t get “stuck” after closing the keyboard.
- **Theme**: dark + light mode; verify contrast and any images/icons.
- **Network**:
  - slow network (throttled)
  - offline
  - if using offline mode: first run (download) vs subsequent runs (cached)
- **Error handling**: test user cancellation / timeouts for PIN-gated and long-running operations and show clear recovery options.
