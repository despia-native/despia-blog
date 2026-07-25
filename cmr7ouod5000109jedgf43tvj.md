---
title: "Lovable In-App Purchases Without Webhooks (RevenueCat)"
seoTitle: "Lovable In-App Purchases Without Webhooks (RevenueCat)"
seoDescription: "Lovable in-app purchases without webhooks: restore entitlements on the frontend and verify server-side with a RevenueCat check when it matters."
datePublished: 2026-07-05T11:08:58.461Z
cuid: cmr7ouod5000109jedgf43tvj
slug: lovable-in-app-purchases-without-webhooks-revenuecat
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/0b250ba3-b3e0-44b5-b6a1-d801f955ec9f.png
tags: mobile-app, revenuecat, lovable

---

Ask Lovable for subscriptions and it will scaffold a RevenueCat file and a webhook handler into your project, because its native path runs on Capacitor. If you are shipping the same app with Despia, most of that boilerplate is dead weight. The `revenuecat.ts` and webhook files it generates describe Capacitor's setup, not what you actually need. Keep the React and Supabase your Lovable app already produced, let Despia ship it as native iOS and Android binaries with RevenueCat's native billing built in, and wire real in-app purchases with a frontend restore and a single server-side check.

## Why the webhook handler is optional

A webhook mirrors RevenueCat's subscription state into your own database so you always hold a local copy. That copy is the maintenance: retries, replays, signature verification, and drift when a delivery fails. RevenueCat is already the source of truth, so rather than keeping a second one in sync you query it directly, in exactly two spots. What the app shows a user is a display decision the phone can make instantly. Whether a protected server action runs is a trust decision the server has to make. The first is a restore, the second is one API call, and neither is an event you have to catch.

## Set up the paywall in RevenueCat and enable it in Despia

The products and the paywall are configured once in RevenueCat, and Despia activates the integration on a build. None of it needs the RevenueCat file Lovable may have scaffolded for Capacitor.

Inside RevenueCat at app.revenuecat.com:

1.  Start a project. The free tier runs up to a monthly revenue threshold, so setup never gates your first release.
    
2.  Register an App Store app in Project settings, Apps, with your iOS bundle id, an App Store Connect API key that has App Manager rights, and an in-app purchase key, so RevenueCat can validate receipts.
    
3.  Register a Play Store app alongside it, using your package name (identical to the bundle id) and your Google Play service account JSON.
    
4.  Define entitlements such as `premium` under Entitlements, import your App Store and Play products, and attach each to the entitlement it grants.
    
5.  Bundle those products into an offering, for instance a `default` offering with monthly and annual packages, and build the paywall on top of it in RevenueCat's paywall editor.
    
6.  Grab the iOS and Android public SDK keys from Project settings, API keys.
    

Back in the Despia Editor, go to App, Settings, Integrations, RevenueCat, and paste both keys. Run a new build after saving. Because the SDK is compiled in, an over-the-air update will not activate it, and saved keys without a rebuild leave purchases failing silently and entitlement lookups empty.

After the build, add the SDK with `npm install despia-native` and open the paywall from your React code:

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

if (isDespia) {
  despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)
}
```

Stick with RevenueCat's native paywall instead of building one in Lovable. The native paywalls handle localized pricing and currency automatically, come as templates built to convert, and update from the RevenueCat dashboard without a rebuild. For other setups, see the [reference](https://setup.despia.com/payments/revenuecat/reference).

## Frontend: restore to gate the interface

Lovable generates React, so the restore lives in a small effect. Gate on the live entitlement, never on a local flag.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

useEffect(() => {
  // reflect the store's entitlement in state, then re-check after purchases
  async function check() {
    if (!isDespia) return
    const { restoredData } = await despia('getpurchasehistory://', ['restoredData'])
    const active = (restoredData ?? []).filter(p => p.isActive)
    setPremium(active.some(p => p.entitlementId === 'premium'))
  }
  check()
  window.onRevenueCatPurchase = check
  return () => { window.onRevenueCatPurchase = undefined }
}, [])
```

Set up one entitlement in RevenueCat with both platform products attached, so iOS and Android hand back the same `entitlementId`. The `onRevenueCatPurchase` callback is a cue to re-run the check, not a green light to unlock on its own. The restore reflects what the client reports, which is right for toggling UI and not enough for guarding anything a modified client could steal.

## Backend: one check in a Supabase edge function

Lovable ships Supabase (Lovable Cloud), so the server-side gate slots straight into an edge function. The client passes the app user id it uses with RevenueCat, and the function asks RevenueCat whether that user is entitled before it does anything costly.

```javascript
// Supabase edge function, the secret key stays on the server
import { serve } from 'https://deno.land/std/http/server.ts'

serve(async (req) => {
  const { userId } = await req.json()
  const id = encodeURIComponent(userId) // anon ids carry a $RCAnonymousID: prefix
  const rc = await fetch(
    `https://api.revenuecat.com/v2/projects/${PROJECT_ID}/customers/${id}/active_entitlements`,
    { headers: { Authorization: `Bearer ${RC_SECRET_KEY}` } }
  )
  // treat an unknown customer as not subscribed
  if (rc.status === 404) return new Response('subscription required', { status: 402 })
  if (!rc.ok) return new Response('upstream error', { status: 502 })
  const { items = [] } = await rc.json()
  if (!items.some(e => e.entitlement_id === 'premium')) {
    return new Response('subscription required', { status: 402 })
  }
  // entitled, run the protected work
  return new Response('ok')
})
```

Make sure the `userId` you send here is the same `external_id` you launched the paywall with. If they do not match, RevenueCat looks up a different customer and the check correctly comes back with no entitlement.

This fits Lovable apps especially well, because most of them already proxy an AI call through a Supabase function to keep the key off the client. That function is where the subscription gate belongs, answered with one RevenueCat request. Keep the secret key server-side, url-encode the id, and expect entitlements with an id and an expiry (null for lifetime) rather than product data. In sandbox the check can read empty even when the entitlement is active, so verify against the older subscriber endpoint while testing. The API also differs across endpoints and versions, so verify the active-entitlements route against the current RevenueCat docs before you ship.

## What you give up

Without a webhook you lose the real-time push: a cancellation will not proactively end a live session or fire a lapse email on the same second. You catch it on the next check, which is normally the user's next request. Add a webhook later if a feature truly needs instant reaction. It belongs on top of this setup, not before it.

## Ship your Lovable app natively

Despia runs your Lovable app as a real iOS and Android binary, with native purchases and restores through one JavaScript call, and no second native project to maintain.

[Read the full RevenueCat setup guide at setup.despia.com](https://setup.despia.com/payments/revenuecat/introduction)