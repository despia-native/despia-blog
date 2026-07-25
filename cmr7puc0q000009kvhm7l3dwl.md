---
title: "React In-App Purchases Without Webhooks (RevenueCat)"
seoTitle: "React In-App Purchases Without Webhooks (RevenueCat)"
seoDescription: "React in-app purchases without webhooks: restore entitlements on the frontend and verify server-side with a RevenueCat check when it matters."
datePublished: 2026-07-05T11:36:42.094Z
cuid: cmr7puc0q000009kvhm7l3dwl
slug: react-in-app-purchases-without-webhooks-revenuecat
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/e8812ef2-1f6a-4801-9181-4493d6f512b7.png
tags: reactjs, webview, mobile-app, webtoapp, react-web

---

Despia ships your React app to iOS and Android as a native binary and gives it RevenueCat's native billing, so you can add subscriptions without standing up a webhook endpoint, verifying signatures, and reconciling RevenueCat into your own tables. None of that is required to take money. A small hook restores entitlements to gate the UI, and a check on your API routes secures the server. RevenueCat stays the single source of truth, and you read it in the two places that need an answer.

## Skip the mirror, ask the source

The reason a webhook feels mandatory is that it keeps a local copy of subscription state. But a copy is precisely what creates the work: delivery failures, replays, ordering, and a table that drifts from RevenueCat the first time an event goes missing. If you never keep the copy, none of that exists. There are only two decisions in your app that depend on subscription state. What the interface renders, which the device can settle instantly, and whether a protected server route runs, which only the server can settle. Handle them independently and the webhook has nothing left to do.

## Configure the paywall in RevenueCat and turn it on in Despia

One-time setup: RevenueCat holds the products and the paywall, and Despia compiles the SDK in at build time.

In RevenueCat:

1.  Create a project at app.revenuecat.com (free under a monthly revenue threshold).
    
2.  Add an App Store app with your iOS bundle id, an App Store Connect API key that has App Manager access, and an in-app purchase key.
    
3.  Add a Play Store app with your package name and a Google Play service account JSON.
    
4.  Create entitlements like `premium`, import your products, and map each product to an entitlement.
    
5.  Collect the products into an offering (`default`, with your monthly and annual packages) and lay out the paywall against it in RevenueCat's paywall editor.
    
6.  Copy the iOS and Android public SDK keys.
    

In the Despia Editor, under App, Settings, Integrations, RevenueCat, paste both keys and rebuild. The SDK is baked into the binary, so skipping the rebuild leaves the keys doing nothing: silent purchase failures and empty entitlement arrays.

Install the SDK with `npm install despia-native`, then trigger the paywall from a component:

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

if (isDespia) {
  despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)
}
```

Reach for RevenueCat's native paywall before hand-building one in React. It localizes pricing into the user's currency, uses templates designed to convert, and is editable from the RevenueCat dashboard without shipping an update. The [reference](https://setup.despia.com/payments/revenuecat/reference) documents the alternatives if you need them.

## A restore hook for the UI

Wrap the restore in a hook so any component can gate on the entitlement, and let it re-check itself when a purchase completes.

```jsx
import { useState, useEffect, useCallback } from 'react'
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

export function useEntitlement(entitlementId) {
  const [active, setActive] = useState(false)

  const check = useCallback(async () => {
    if (!isDespia) return
    const { restoredData } = await despia('getpurchasehistory://', ['restoredData'])
    const owned = (restoredData ?? []).filter(p => p.isActive)
    setActive(owned.some(p => p.entitlementId === entitlementId))
  }, [entitlementId])

  useEffect(() => {
    check()
    window.onRevenueCatPurchase = check
    return () => { window.onRevenueCatPurchase = undefined }
  }, [check])

  return active
}
```

Then a component just reads a boolean: `const isPremium = useEntitlement('premium')`. Define one entitlement in RevenueCat with both platform products attached, so iOS and Android return the same id and your components never branch on platform. The `onRevenueCatPurchase` global is a trigger to re-run the check, not an unlock in itself, which keeps the UI tied to what the store reports rather than to the fact a button was tapped.

This hook trusts the client, which is exactly what you want for rendering and exactly what you cannot rely on for anything a user could fake.

## Guard your API routes on the server

For a route that costs money or serves protected data, verify at request time. The client sends the app user id, and your route asks RevenueCat before running.

```javascript
// server-side helper, secret key stays out of the bundle
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

// e.g. app/api/generate/route.js
export async function POST(req) {
  const { userId } = await req.json()
  if (!(await hasActiveEntitlement(userId, 'premium'))) {
    return new Response('subscription required', { status: 402 })
  }
  // entitled, do the protected work
}
```

The `userId` you check here has to be the same value you passed as `external_id` at paywall launch. Mismatched ids resolve to a different RevenueCat customer, and the check correctly returns no entitlement.

If your app proxies an AI model, that route already exists to hide the key, which makes it the obvious home for the gate. Keep the secret key server-side, url-encode the id, and read entitlements that carry an id and an expiry (null for lifetime) but no product info. Cache the result per user for a couple of minutes if you call it on every request, so latency and rate limits stay comfortable without letting access go noticeably stale. In sandbox the endpoint can read empty even when the entitlement is active, so cross-check the older subscriber lookup while testing. The API also differs across endpoints and versions, so verify the active-entitlements route against the current RevenueCat docs before you ship.

## The honest tradeoff

Without a webhook you give up instant reaction. A cancellation will not sever a live session on the same second or auto-fire a dunning email; you see it at the next check, which is the next request in practice. Reach for a webhook only when a feature genuinely needs real-time state, and layer it on top of this rather than in front of it.

## Ship your React app to the stores

Despia turns your React app into a real iOS and Android binary, native purchases and restores included, from the single codebase you already maintain.

[Read the full RevenueCat setup guide at setup.despia.com](https://setup.despia.com/payments/revenuecat/introduction)