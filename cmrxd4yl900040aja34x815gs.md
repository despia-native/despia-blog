---
title: "Turn a Lovable Web App into a Mobile App"
seoTitle: "Turn a Lovable Web App into a Mobile App"
seoDescription: "Convert your Lovable web app into a native mobile app and publish it to the App Store and Google Play, on TanStack Start or the older Vite stack."
datePublished: 2026-07-23T10:23:03.455Z
cuid: cmrxd4yl900040aja34x815gs
slug: turn-a-lovable-web-app-into-a-mobile-app
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/0ecfd534-e720-4837-86c0-6ce888be6ee4.png
tags: webview, webapp, pwa, lovable

---

New Lovable projects ship on TanStack Start, older ones on React and Vite, and Despia has full support for both. It converts either into a native mobile app on the App Store and Google Play, no second project to maintain, with native billing, push, and 50+ device features. Here is the path: add the package, detect the runtime, call a native feature, and publish.

## Add the package

Lovable runs on a real npm project, so you have two ways in. The fast one is the chat, tell Lovable to add it:

```plaintext
Install the npm package despia-native and import the default export where I call native features.
```

If you have connected the GitHub repo, you can also add it the normal way from your own machine:

```bash
npm install despia-native
```

Either path lands the same dependency. Import the default export where you need it:

```javascript
import despia from 'despia-native'
```

Installing `despia-native` adds the JavaScript bridge. Features that need native SDKs, such as OneSignal, RevenueCat, or Stripe, still need the matching Despia integration enabled and a fresh native build.

## Detect the Despia runtime

The `despia()` calls only resolve inside the Despia app. On `yourapp.lovable.app` in a browser they do nothing, so gate every call and the same build runs on web and native without branching your logic.

For Despia apps, use the user agent check:

```javascript
const ua = typeof navigator !== 'undefined' ? navigator.userAgent.toLowerCase() : ''
const isDespia = ua.includes('despia')
const isDespiaIOS = isDespia && (ua.includes('iphone') || ua.includes('ipad'))
const isDespiaAndroid = isDespia && ua.includes('android')
```

New Lovable projects run on TanStack Start with SSR, where the server has no `navigator`, so the `typeof navigator` guard keeps detection client-only. On an older Vite project it is simply harmless. Keep this in a single util. Lovable rewrites component code as you prompt, and you do not want your detection logic regenerated out from under you.

## Call your first native feature

Three call shapes cover everything. Lead with the features you will actually ship: push, purchases, biometric storage.

Fire-and-forget. Register the signed-in user for push on every authenticated load:

```javascript
if (isDespia) despia(`setonesignalplayerid://?user_id=${userId}`)
```

Promise. Check entitlements straight from the native store, no backend:

```javascript
async function checkEntitlements() {
    const data   = await despia('getpurchasehistory://', ['restoredData'])
    const active = (data.restoredData ?? []).filter(p => p.isActive)
    if (active.some(p => p.entitlementId === 'premium')) unlockPremium()
}
```

Callback. Assigned outside the gate so the runtime can invoke it, re-checking on every purchase:

```javascript
window.onRevenueCatPurchase = checkEntitlements
checkEntitlements()
```

Storage Vault rounds it out. It survives reinstall and can gate a value behind Face ID:

```javascript
await despia('setvault://?key=sessionToken&value=abc123&locked=true')

// reading a locked key triggers the biometric prompt
const { sessionToken } = await despia('readvault://?key=sessionToken', ['sessionToken'])
```

## What Lovable's own export leaves out

Lovable's Capacitor path runs your app from `file://`, which breaks Supabase auth, OAuth, and Sign In with Apple, and it does not include native billing. Despia serves your app from a real web origin instead, your hosting URL over https by default (remote hydration), or `http://localhost` in offline mode, so cookies and every SDK behave like they do on the web, and RevenueCat is built in:

```javascript
if (isDespia) {
    despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)
}
```

That is the rail for digital goods. Physical goods and real-world services are the reverse: Apple and Google take no cut and require Stripe, so anything that ships or gets performed goes through the native Stripe sheet instead of the paywall:

```javascript
async function checkoutCart() {
    const res = await fetch('/api/create-payment-intent', { method: 'POST' })
    const { publishable_key, client_secret } = await res.json()
    if (isDespia) {
        despia(`stripe://payment?publishable_key=${publishable_key}&payment_intent_client_secret=${client_secret}`)
    }
}
```

Register push on every authenticated load so notifications map to the right user:

```javascript
if (isDespia) despia(`setonesignalplayerid://?user_id=${userId}`)
```

## Let Lovable write correct Despia code

The best way to keep the generated code accurate is to connect the Despia MCP, so the agent reads the real API surface instead of hallucinating calls you have to fix later. Add it as a chat connector once, under Connectors, Chat connectors, New MCP server. Name it Despia, set the URL to `https://setup.despia.com/mcp`, choose no authentication, then add the server.

If MCP is not an option, the second-best path is to paste the relevant feature pages from `setup.despia.com` into the chat before you prompt. Same source, more manual.

## Ship one app, not two

Keep building in Lovable. Despia ships that app to both stores, gives it native device access, and pushes web updates over the air, from the single codebase you already have.

[Learn more about Despia in the docs](https://setup.despia.com)