---
title: "Vue.js In-App Purchases Without Webhooks (RevenueCat)"
seoTitle: "Vue.js In-App Purchases Without Webhooks (RevenueCat)"
seoDescription: "Vue.js in-app purchases without webhooks: restore entitlements on the frontend and verify server-side with a RevenueCat check when it matters."
datePublished: 2026-07-05T11:41:15.397Z
cuid: cmr7q06ws00000akkfelvgssi
slug: vue-js-in-app-purchases-without-webhooks-revenuecat
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/79425f4a-e3ae-40e0-84ac-4e01b0325d52.png
tags: vuejs, webview, vue, revenuecat

---

Despia runs your Vue app on iOS and Android as a native binary with RevenueCat's native billing built in, so you can add subscriptions without wiring up webhook delivery, signature verification, and a mirrored subscription table. You can skip the whole event pipeline. A composable restores entitlements to gate the interface, and a server-side check secures your endpoints. RevenueCat holds the truth, and you read it in the only two places that depend on it.

## Two decisions, no event pipeline

A webhook is there to keep a copy of subscription state in your own database. The copy is the cost: retries, out-of-order events, verification, and drift when a delivery is missed. Drop the copy and you drop the cost. Your app makes exactly two decisions off subscription state. What the UI shows, which the device answers on its own, and whether a protected request runs, which the server has to answer. A restore covers the first, a direct call covers the second, and there is no event left to handle.

## Create the paywall in RevenueCat and connect Despia

Configure this once. The products and paywall live in RevenueCat, and Despia activates them by compiling the SDK into a build.

RevenueCat side:

1.  Open a project at app.revenuecat.com. The free tier covers you up to a monthly revenue threshold.
    
2.  Add an App Store app carrying your iOS bundle id, an App Store Connect API key with App Manager access, and an in-app purchase key.
    
3.  Add a Play Store app with the matching package name and your Google Play service account JSON.
    
4.  Create your entitlements (`premium`, `no_ads`, whatever you unlock), import the store products, and attach each to its entitlement.
    
5.  Assemble the products into an offering such as `default`, then design the paywall for that offering in RevenueCat's paywall editor.
    
6.  Copy out the iOS and Android public SDK keys.
    

Despia side: in the Editor go to App, Settings, Integrations, RevenueCat, paste both keys, and start a fresh build. The SDK cannot activate over the air, so a rebuild is required, otherwise purchases fail without a sound and entitlement checks return empty.

Add the SDK with `npm install despia-native`, then open the paywall from Vue with a single call:

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

if (isDespia) {
  despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)
}
```

Prefer RevenueCat's native paywall over building your own in Vue. The native paywalls show localized prices in each user's currency, ship as templates designed for conversion, and can be changed from the RevenueCat dashboard without a rebuild. See the [reference](https://setup.despia.com/payments/revenuecat/reference) for alternative setups.

## A restore composable for the UI

Vue's composition API makes this a clean composable that returns a reactive flag and re-checks after any purchase.

```javascript
import { ref, onMounted, onUnmounted } from 'vue'
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

export function useEntitlement(entitlementId) {
  const active = ref(false)

  async function check() {
    if (!isDespia) return
    const { restoredData } = await despia('getpurchasehistory://', ['restoredData'])
    const owned = (restoredData ?? []).filter(p => p.isActive)
    active.value = owned.some(p => p.entitlementId === entitlementId)
  }

  onMounted(() => {
    check()
    window.onRevenueCatPurchase = check
  })
  onUnmounted(() => { window.onRevenueCatPurchase = undefined })

  return active
}
```

A component reads it as `const isPremium = useEntitlement('premium')` and templates react to it. Configure one entitlement in RevenueCat with both platform products attached, so iOS and Android return the same id and your templates never branch on platform. Let `onRevenueCatPurchase` trigger a re-check rather than grant access directly, so the flag always tracks the store.

Reactive gating is the right tool for the interface. It follows what the client claims, which is fine for rendering and not enough for protecting anything a tampered client could take.

## A server-side check for your endpoints

For an endpoint that spends money or returns protected data, verify when the request lands. The client sends its RevenueCat app user id, and your handler asks RevenueCat before doing the work.

```javascript
// server-side only, secret key never reaches the client bundle
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

// in your API handler
if (!(await hasActiveEntitlement(userId, 'premium'))) {
  return new Response('subscription required', { status: 402 })
}
// entitled, run the protected work
```

Use the same stable `userId` here that you passed as `external_id` when launching the paywall. If those ids differ, RevenueCat looks up a different customer and the check correctly returns no entitlement.

If your app has an AI feature, its backend already exists to keep the model key private, and that is where the gate goes. Keep the secret key on the server, url-encode the id, and expect entitlements with an id and an expiry (null for lifetime) but no product detail. A short per-user cache is worth it if you check on every request. In sandbox the endpoint can read empty even when the entitlement is active, so cross-check the older subscriber lookup while you test. The API also differs across endpoints and versions, so verify the active-entitlements route against the current RevenueCat docs before you ship.

## What you trade off

Skipping webhooks means losing the instant push. A cancellation will not cut a live session or send a lapse email on the same second; you learn about it at the next check, which is the next request anyway. When a feature truly needs real-time state, add a webhook then, on top of this rather than ahead of it.

## Ship your Vue app to the stores

Despia ships your Vue app as a real iOS and Android binary, native purchases and restores included, from the codebase you already have.

[Read the full RevenueCat setup guide at setup.despia.com](https://setup.despia.com/payments/revenuecat/introduction)