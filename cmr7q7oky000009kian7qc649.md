---
title: "RevenueCat Without Webhooks: Restore and a Server Check"
seoTitle: "RevenueCat Without Webhooks: Restore and a Server Check"
seoDescription: "You can ship RevenueCat without webhooks: restore entitlements, and call the RevenueCat API server-side to gate your protected endpoints."
datePublished: 2026-07-05T11:47:04.901Z
cuid: cmr7q7oky000009kian7qc649
slug: revenuecat-without-webhooks-restore-and-a-server-check
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/20520df5-facb-413f-a457-485fd2893df0.png
tags: webview, revenuecat, hybridapp, webnative

---

Webhook setup is the part of RevenueCat that eats a day. An endpoint to stand up, retries and out-of-order events to handle, a signature to verify, and a copy of the subscription state in your own database that drifts the moment a delivery fails. For most apps you can skip all of it. Restore on the frontend to decide what the UI shows (in a Despia app that restore is a single native call), and ask RevenueCat directly on the backend to decide whether a protected call runs. RevenueCat already holds the truth, so you read it instead of mirroring it.

## Why webhooks are the part you get stuck on

A webhook exists to push subscription changes into your own store the instant they happen, so your database always has a local copy of who is subscribed. That copy is the problem. It is a second source of truth, and keeping it correct means handling failed deliveries, replays, events that land out of order, and a signing secret you have to verify on every call. None of that ships your first paying user. It is an optimization for the day you need state changes in real time, not a requirement for taking money.

Drop the mirror and the question gets simpler. There are only two things you actually need to answer, and neither one needs an event pipeline.

## The two questions you actually need to answer

One, what should the interface show this user right now. That is a display decision, and the device can answer it instantly.

Two, should this server actually do the expensive or protected thing being asked of it. That is a trust decision, and only the server can answer it safely.

Answer the first with a restore on the frontend. Answer the second with a direct API call on the backend. That is the whole setup.

## Frontend: restore to gate the UI

Restore reads the entitlements the store already has cached on the device and tells you which ones are active. Gate the interface on that, never on a local flag you set yourself. In a Despia app the restore is one call, and the result comes back as active entitlements you match by id. Install the SDK with `npm install despia-native` and import `despia` where you call it.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// show or hide features based on what the store reports
async function refreshUI() {
  if (!isDespia) return
  const { restoredData } = await despia('getpurchasehistory://', ['restoredData'])
  const active = (restoredData ?? []).filter(p => p.isActive)
  unlockPremium(active.some(p => p.entitlementId === 'premium'))
}

refreshUI()                              // on load
window.onRevenueCatPurchase = refreshUI  // and again after any purchase
```

Set the entitlement up once in RevenueCat and attach both your iOS and Android products to it, so both platforms return the same `entitlementId` and your gating code stops caring which store the user came from.

Two habits keep this reliable. Run the check on load, on navigation, and before you reveal a gated feature. The calls are instant and work offline, because Apple and Google cache entitlement state on the device. And treat `onRevenueCatPurchase` as a signal, not a grant: it fires when a purchase succeeds, but you re-run the same restore rather than unlocking straight from the event, so access is always tied to what the store actually reports.

This gate is honest about one thing: it reflects what the client says. That is exactly right for showing and hiding UI, and exactly wrong for protecting anything a faked client could walk off with. For that, the check moves to the server.

## Backend: ask RevenueCat when the call comes in

For anything a manipulated client could abuse, a server-delivered file, tokens billed to you, a premium API route, verify at the moment the request arrives. The client sends the app user id it uses with RevenueCat, and your backend asks RevenueCat whether that user has an active entitlement before it does the work.

```javascript
// server-side only, the secret key never reaches the client
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

Then gate the route:

```javascript
// check before spending money or handing over protected data
if (!(await hasActiveEntitlement(userId, 'premium'))) {
  return new Response('subscription required', { status: 402 })
}
// entitled, safe to run the real work
```

Whatever id you check here must be the same `external_id` you launched the paywall with, or RevenueCat resolves a different customer and the check correctly returns no entitlement.

This lands cleanly on the app most people are building now. If your product has an AI feature, you already run a backend to proxy the model call and keep the key off the client. That proxy is the natural place for the gate, and it answers the subscription question with one request to RevenueCat instead of a webhook pipeline and a table you have to reconcile.

A few practicals so the server check behaves the first time:

*   The secret key stays on the server. It is not the same as your public SDK key, and it must never ship in the app.
    
*   The lookup is keyed by the app user id. URL-encode it before you put it in the path, since anonymous ids carry reserved characters that will otherwise break the request.
    
*   The response lists active entitlements, each with an id and an expiry as an epoch-millisecond timestamp, or null for lifetime access. It does not tell you which product or plan granted it, so cross-reference elsewhere if you need the plan.
    
*   Results paginate. Most users fit a single page, but follow the next-page link until it is empty if you support stacked entitlements.
    
*   If you hit this endpoint on every request, cache the result per user for a couple of minutes. It cuts latency and stays well clear of rate limits without letting access go stale in any way a user would notice.
    

One testing gotcha worth knowing: sandbox and simulated StoreKit purchases sometimes read as empty on the newer endpoint even when the entitlement is clearly active. If a sandbox grant looks missing, confirm against RevenueCat's older subscriber lookup before assuming your code is wrong. The API also differs across endpoints and versions, so verify the active-entitlements route against the current RevenueCat docs before you ship.

## What you give up without webhooks

Being honest about the tradeoff is the point. Without a webhook you lose the instant push. A cancellation will not proactively cut a live session on the same second, and it will not fire a dunning or lifecycle email the moment the plan lapses. You find out at the next check instead, which for most apps is the next request the user makes anyway, so it rarely matters.

When it does matter, when you genuinely need real-time revocation, subscription-driven emails, or live analytics, add a webhook then. It slots in as an addition on top of this setup, not a prerequisite you debug before your first sale.

## Putting it together

Enable RevenueCat in your Despia settings and add the API key. Create one entitlement and attach both platform products to it. Wire the frontend restore into app load and the purchase callback to gate the interface. Add the server-side check to any route that spends money or serves protected data. That is a complete, shippable subscription setup with no webhook anywhere in it, and it holds up until a feature specifically demands real-time state.

## Ship it from your existing codebase

The native paywall, purchase, and restore calls are one `despia()` away, inside the web app you already have. No Xcode, no native project to maintain, no billing SDK on the web layer.

[Read the full RevenueCat setup guide at setup.despia.com](https://setup.despia.com/payments/revenuecat/introduction)