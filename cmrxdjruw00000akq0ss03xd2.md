---
title: "Turn a React Web App into a Mobile App"
seoTitle: "Turn a React Web App into a Mobile App"
seoDescription: "Convert a React web app into a native mobile app and publish it to the App Store and Google Play, with native device features and no ejecting."
datePublished: 2026-07-23T10:34:34.590Z
cuid: cmrxdjruw00000akq0ss03xd2
slug: turn-a-react-web-app-into-a-mobile-app
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/8695d9f3-2fc1-4485-8188-34fd1d5141ee.png
tags: react-js, reactjs, webview, webapp, pwa

---

* * *

Your React app is the source of truth. Despia runs it as a native mobile app on the App Store and Google Play and hands it 50+ device features through one function. This covers the path, plus the two React specifics that keep it clean: gating in effects and assigning callbacks correctly.

## Install the package

```bash
npm install despia-native
```

The package ships TypeScript definitions, so the import is typed with no extra setup:

```javascript
import despia from 'despia-native'
```

Installing `despia-native` adds the JavaScript bridge. Features that need native SDKs, such as OneSignal, RevenueCat, or Stripe, still need the matching Despia integration enabled and a fresh native build.

## Detect the Despia runtime

`despia()` only resolves inside the Despia app. In a normal browser it is inert, so guard every call and one build serves web and native.

```javascript
const ua = typeof navigator !== 'undefined' ? navigator.userAgent.toLowerCase() : ''
export const isDespia = ua.includes('despia')
export const isDespiaIOS = isDespia && (ua.includes('iphone') || ua.includes('ipad'))
export const isDespiaAndroid = isDespia && ua.includes('android')
```

Export these from one module. The `typeof navigator` guard keeps it safe on Next.js or any SSR build, where the server has no `navigator`. On a client-only app the guard is simply harmless.

## Call features from an effect

Do native work in effects, not in render. Render runs on the server and on every re-render, effects run once on the client where the runtime lives. Here it reads live entitlements from the native store, no backend call:

```jsx
import { useEffect, useState } from 'react'
import despia from 'despia-native'
import { isDespia } from './despia'

function Gate() {
  const [premium, setPremium] = useState(false)

  useEffect(() => {
    if (!isDespia) return
    despia('getpurchasehistory://', ['restoredData']).then(({ restoredData }) => {
      const active = (restoredData ?? []).filter((p) => p.isActive)
      setPremium(active.some((p) => p.entitlementId === 'premium'))
    })
  }, [])

  return premium ? <Premium /> : <Upsell />
}
```

Fire-and-forget calls drop straight into a handler. Register the user for push, or launch the native paywall on a tap:

```jsx
<button onClick={() => isDespia && despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)}>
  Go Premium
</button>
```

That paywall is for digital goods. Physical goods and real-world services take the opposite rail: Apple and Google take no cut and require Stripe, so a shipped product or a booking goes through the native Stripe sheet, not the paywall:

```jsx
async function checkoutCart() {
  const res = await fetch('/api/create-payment-intent', { method: 'POST' })
  const { publishable_key, client_secret } = await res.json()
  if (isDespia) {
    despia(`stripe://payment?publishable_key=${publishable_key}&payment_intent_client_secret=${client_secret}`)
  }
}
```

## Assign callbacks once, outside render

Native events arrive on `window.onXxx`. Set them in an effect with an empty dependency array so you register one handler for the app's lifetime, not one per render:

```jsx
useEffect(() => {
  window.onRevenueCatPurchase = () => refreshEntitlements()
  return () => { window.onRevenueCatPurchase = undefined }
}, [])
```

Assigning these in the render body would rebind on every pass and leak stale closures. The effect is the right home.

## Point your AI editor at the Despia docs

If you scaffold React with Cursor, VS Code, Windsurf, or Claude, give it the Despia MCP so it writes against the real API instead of hallucinating scheme names. The remote server is the same everywhere:

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

Every native capability here is one function call away, inside the React codebase you already have. No Xcode, no native project to keep in sync, and web updates ship over the air.

[Read the full reference at setup.despia.com](https://setup.despia.com)