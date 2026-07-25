---
title: "Base44 In-App Purchases Without Webhooks (RevenueCat)"
seoTitle: "Base44 In-App Purchases Without Webhooks (RevenueCat)"
seoDescription: "Base44 in-app purchases without webhooks: restore entitlements on the frontend and verify server-side with a RevenueCat check when it matters."
datePublished: 2026-07-05T11:00:18.512Z
cuid: cmr7ojj5c00000akq3sqghp91
slug: base44-in-app-purchases-without-webhooks-revenuecat
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/ecbf0d76-f6ad-4e99-a9e6-98e9e402ef63.png
tags: mobileapp, revenuecat, base44

---

You built the app in Base44, and the paywall is probably just an interface: a few buttons and a tier flag that were never wired to real billing. To sell subscriptions on the App Store and Google Play you need genuine in-app purchases, and most guides route you straight into webhook handlers, signature checks, and a subscription table you have to keep in sync. You can ship without any of it. Keep the Base44 screens you already built, let Despia add the native RevenueCat layer to the iOS and Android binaries, and answer the only two questions that matter.

## The two questions, and why neither needs a webhook

A webhook exists to copy subscription changes into your own database the moment they happen. That copy is a second source of truth, and keeping it honest means handling retries, replays, and out-of-order events forever. RevenueCat already tracks who is subscribed, so instead of mirroring it you ask it, in the two places that care:

*   What should the app show this user. A display decision the device can answer on its own.
    
*   Should the server run this protected action. A trust decision only the server can make safely.
    

Restore answers the first. A direct API call answers the second. No events, no mirror.

## Build the paywall in RevenueCat, then add the keys in Despia

Before any gating works, the products and the paywall live in RevenueCat, and Despia turns the integration on at build time. This is a one-time setup.

In RevenueCat at app.revenuecat.com:

1.  Create a project. It is free until you cross a monthly revenue threshold, so it never blocks a Base44 build in progress.
    
2.  Add an App Store app under Project settings, Apps, with your iOS bundle id and an App Store Connect API key that has App Manager access, plus an in-app purchase key.
    
3.  Add a Play Store app the same way, using the same package name as the bundle id and your Google Play service account JSON.
    
4.  Create your entitlements, one per unlock such as `premium`, then import your store products and attach each one to its entitlement.
    
5.  Group the products into an offering, say a `default` offering with a monthly and an annual package, and design the paywall against it in RevenueCat's paywall editor. Layout and pricing copy are set there, not in your app.
    
6.  Copy the iOS and Android public SDK keys from Project settings, API keys.
    

Then in the Despia Editor, open App, Settings, Integrations, RevenueCat, and paste both keys exactly. Trigger a fresh build afterwards. The RevenueCat SDK compiles into the binary, so it cannot arrive over the air, and until you rebuild the integration stays dormant even with the keys saved: purchases fail quietly and entitlement checks return nothing.

With the build done, install the SDK with `npm install despia-native` and open that paywall from your web layer:

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

if (isDespia) {
  despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)
}
```

Use RevenueCat's native paywall rather than building one in Base44. It renders each user's price in their own currency, ships as conversion-focused templates, and can be restyled from the RevenueCat dashboard without another build. If you have a reason to run a custom purchase flow instead, the [reference](https://setup.despia.com/payments/revenuecat/reference) covers the alternatives.

## Gate premium on the real entitlement

With the paywall handled by RevenueCat, your job in the app is the gate: deciding what a subscriber actually sees. Base it on a real entitlement, not a tier flag your UI set, which is a guess the store never confirmed. Read it with a restore.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// unlock on what the store reports, not on a UI flag we set
async function refreshAccess() {
  if (!isDespia) return
  const { restoredData } = await despia('getpurchasehistory://', ['restoredData'])
  const active = (restoredData ?? []).filter(p => p.isActive)
  applyPremium(active.some(p => p.entitlementId === 'premium'))
}

refreshAccess()                              // on load
window.onRevenueCatPurchase = refreshAccess  // and after a purchase completes
```

Create one entitlement in RevenueCat, attach your iOS and Android products to it, and both platforms return the same `entitlementId`. Run the restore on load and after `onRevenueCatPurchase` fires. Treat that callback as a signal to re-check, not a reason to grant access outright, so the gate always follows the store.

This restore is the right call for the interface. It reflects what the client claims, which is fine for showing and hiding screens and wrong for protecting anything valuable. That part moves to the server.

## Secure the real thing with a Base44 function

For anything a tampered client could take, a paid export, an action that costs you money, a premium-only endpoint, verify when the request arrives. Base44 can run server-side functions, and that is where the check belongs. The client sends the app user id it uses with RevenueCat, and the function asks RevenueCat whether that user is entitled before doing the work.

```javascript
// runs server-side in a Base44 function, secret key never ships to the client
async function hasActiveEntitlement(appUserId, entitlementId) {
  const id = encodeURIComponent(appUserId) // anon ids carry a $RCAnonymousID: prefix
  const res = await fetch(
    `https://api.revenuecat.com/v2/projects/${PROJECT_ID}/customers/${id}/active_entitlements`,
    { headers: { Authorization: `Bearer ${RC_SECRET_KEY}` } }
  )
  if (res.status === 404) return false        // customer RevenueCat has never seen
  if (!res.ok) throw new Error(`RevenueCat ${res.status}`)
  const { items = [] } = await res.json()
  return items.some(e => e.entitlement_id === entitlementId)
}
```

One thing to get right: the `userId` you pass here must be the same `external_id` you used when launching the paywall. If they differ, RevenueCat resolves a different customer and the check correctly returns no entitlement.

Gate the protected path on it and reject anyone without an active entitlement. A few practicals: the secret key stays on the server, url-encode the app user id, and expect a list of entitlements each carrying an id and an expiry timestamp (null for lifetime), not any product detail. Most users fit one page of results. In sandbox the check can read empty even when the entitlement is live, so confirm against RevenueCat's older subscriber lookup while testing. The API also differs across endpoints and versions, so verify the active-entitlements route against the current RevenueCat docs before you ship.

## What you trade away

Skipping webhooks costs you the instant push. A cancellation will not cut a live session on the same second or auto-send a lapse email; you learn about it at the user's next check, which is usually their next request anyway. If you later need true real-time behaviour, add a webhook then as an extra layer. It is never the thing to debug before your first paying user.

## Ship your Base44 app to the stores

Your Base44 app runs as a real iOS and Android binary through Despia, with native purchases one call away and no native project to maintain.

[Read the full RevenueCat setup guide at setup.despia.com](https://setup.despia.com/payments/revenuecat/introduction)