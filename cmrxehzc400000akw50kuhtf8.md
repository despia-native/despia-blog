---
title: "Turn a Web App into a Mobile App"
seoTitle: "Turn a Web App into a Mobile App"
seoDescription: "Convert a web app into a native mobile app and publish it to the App Store and Google Play, whether you build with npm or ship a plain HTML page."
datePublished: 2026-07-23T11:01:10.584Z
cuid: cmrxehzc400000akw50kuhtf8
slug: turn-a-web-app-into-a-mobile-app
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/4c8489eb-e69e-49b3-93dc-a63cfb3bd509.png
tags: cdn, npm, webview, webapp, pwa

---

Any web app can become a native mobile app on the App Store and Google Play, whether you build with a bundler or ship a single HTML page. Despia gives it 50+ device features behind one function. This covers both install paths, runtime detection, and the three call shapes: fire-and-forget, promise, and callback.

## Install: with a bundler or from a CDN

Two ways in, depending on whether you have a build step.

With a bundler (Vite, webpack, Rollup, esbuild, Parcel), install from npm and import the default export:

```bash
npm install despia-native
```

```javascript
import despia from 'despia-native'
```

No build step, just an HTML page? Load it from the CDN instead. The ESM build is the clean option:

```html
<script type="module">
  import despia from 'https://cdn.jsdelivr.net/npm/despia-native/+esm'
</script>
```

If you cannot use module scripts, the UMD build exposes `despia` globally, so load it before any script that calls it:

```html
<script src="https://cdn.jsdelivr.net/npm/despia-native/index.min.js"></script>
```

Either way, `despia-native` is just the JavaScript bridge. Features that need native SDKs, such as OneSignal, RevenueCat, or Stripe, still need the matching Despia integration enabled and a fresh native build.

## Detect the Despia runtime

`despia()` only resolves inside the Despia app, so gate every call and the same code runs untouched in a normal browser:

```javascript
const ua = navigator.userAgent.toLowerCase()
const isDespia = ua.includes('despia')
const isDespiaIOS = isDespia && (ua.includes('iphone') || ua.includes('ipad'))
const isDespiaAndroid = isDespia && ua.includes('android')
```

## The three call shapes

Each shape maps to a feature you will actually ship.

Fire-and-forget, no return value. Register the signed-in user for push on every authenticated load:

```javascript
if (isDespia) despia(`setonesignalplayerid://?user_id=${userId}`)
```

Promise, for calls that read device state. Check entitlements straight from the native store, no backend:

```javascript
async function checkEntitlements() {
  const data   = await despia('getpurchasehistory://', ['restoredData'])
  const active = (data.restoredData ?? []).filter((p) => p.isActive)
  if (active.some((p) => p.entitlementId === 'premium')) unlockPremium()
}
```

Callback, for native events. Assign the `window.onXxx` handler once at load, outside the gate, so the runtime can call it later:

```javascript
window.onRevenueCatPurchase = checkEntitlements
```

## Wire it to real elements

Run your setup after the DOM is ready and attach handlers directly. This works the same whether the SDK came from npm or a script tag:

```javascript
document.addEventListener('DOMContentLoaded', () => {
  const buy = document.querySelector('#buy')
  buy.addEventListener('click', () => {
    if (isDespia) {
      despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)
    }
  })
})
```

That paywall is the rail for digital goods. Physical goods and real-world services are the reverse: Apple and Google take no cut and require Stripe, so a shipped product or a booking goes through the native Stripe sheet instead:

```javascript
async function checkoutCart() {
  const res = await fetch('/api/create-payment-intent', { method: 'POST' })
  const { publishable_key, client_secret } = await res.json()
  if (isDespia) {
    despia(`stripe://payment?publishable_key=${publishable_key}&payment_intent_client_secret=${client_secret}`)
  }
}
```

## A full page with no build

For the no-bundler case, everything above fits in one HTML file with nothing to install:

```html
<!doctype html>
<meta charset="utf-8">
<button id="buy">Go Premium</button>
<script type="module">
  import despia from 'https://cdn.jsdelivr.net/npm/despia-native/+esm'

  const isDespia = navigator.userAgent.toLowerCase().includes('despia')

  document.querySelector('#buy').addEventListener('click', () => {
    if (isDespia) {
      despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)
    }
  })
</script>
```

## Point your AI editor at the Despia docs

If you write your code with Cursor, VS Code, Windsurf, or Claude, give it the Despia MCP so it writes against the real API instead of guessing at scheme names. The remote server is the same everywhere:

```json
{
  "mcpServers": {
    "despia": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://setup.despia.com/mcp"]
    }
  }
}
```

Cursor reads `~/.cursor/mcp.json`, VS Code reads `.vscode/mcp.json` under a `servers` key, and any MCP-capable AI builder takes the URL `https://setup.despia.com/mcp` directly. Local editors need Node 18 or newer.

MCP is the best way to keep generation accurate. If your setup has no MCP support, the second-best option is to paste the relevant feature pages from `setup.despia.com` into the chat before you prompt.

## Add this to your app

Every native capability here is one function call from your existing web app, bundler or no bundler. Ship the same code to iOS and Android and update it over the air.

[Read the full reference at setup.despia.com](https://setup.despia.com)