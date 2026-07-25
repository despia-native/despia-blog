---
title: "RevenueCat Purchase Not Confirming : Cause and Fix"
seoTitle: "RevenueCat Purchase Not Confirming: Cause and Fix"
seoDescription: "If your RevenueCat checkout keeps spinning after a successful purchase, the cause is backend confirmation. Fix it with a native store check."
datePublished: 2026-07-09T07:28:12.316Z
cuid: cmrd6q67y00000ajfghpw3bf6
slug: revenuecat-purchase-not-confirming-cause-and-fix
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/409006b9-f42a-4794-add9-b31b4ef86206.png
tags: android-app-development, ios, android, ios-app-development, revenuecat, in-app-purchase

---

You wired up RevenueCat, the App Store sandbox charges the test card, the purchase goes through, and then the paywall just sits there spinning. It never closes. The user paid and the app still acts locked. This is the most common in-app purchase bug in a WebView app, and it is almost never a purchase problem. It is a confirmation problem, and it is usually one line of logic in the wrong place.

## The paywall is waiting on the wrong signal

Here is the pattern that breaks, and it is the pattern an AI coding tool writes by default. After the purchase, the app calls a backend function to confirm the subscription, polls it a few times, and only closes the paywall once that backend says `subscribed: true`.

```javascript
// the pattern that spins forever in sandbox
const purchase = await startPurchase(plan)
if (purchase.status === 'completed') {
  for (let i = 0; i < 30; i++) {
    const { data } = await supabase.functions.invoke('check-subscription')
    if (data?.subscribed) { closePaywall(); break }   // rarely reached in sandbox
    await sleep(1500)
  }
  // 30 tries later: still not subscribed, paywall stays open
}
```

That backend function reads subscription state from somewhere you control, usually a row in your own database. The problem is that nothing put the purchase there. It happened on the device, client-side, and the server was never in the loop. Why the record is missing depends on the setup, and it is worth understanding the options rather than guessing at one:

*   No webhook is configured, so RevenueCat never told your backend a purchase happened. That database copy only exists if something writes it, and with no event handler nothing does.
    
*   The function queries a source a native store purchase never touches. If the table it reads is populated by a web checkout flow, or the check points at a different billing system entirely, a StoreKit or Play Billing purchase will not show up there no matter how long you poll.
    
*   Sandbox behaviour. Sandbox and TestFlight purchases do not always propagate to a backend the way production ones do, so even a correct webhook can read empty in test mode.
    
*   Timing. The client confirms the purchase the instant it clears, but the server write lands later, so a poll that starts immediately can race ahead of the record it is waiting for.
    

Any one of these leaves the loop running its full length and giving up, seconds after a payment that actually succeeded. The common thread is the same in every case: the paywall is waiting on a copy of the truth instead of the truth.

## The purchase already succeeded

The store confirmed the transaction on the device the moment it completed. RevenueCat calls `window.onRevenueCatPurchase()` right then, client-side. That callback is a signal that a transaction happened. It is not an instruction to go poll a server, and access should not wait on one.

The subscription state already lives on the device. The store caches it. RevenueCat can read it back instantly and offline through the native layer, with no round trip to anything you host. In a WebView app running on Despia, that read is a single call across to the native runtime. So the correct source of truth for closing the paywall is the store on the device, not a table in your database.

## The fix: confirm from the store, not the server

Read the active entitlements straight from the store and close the paywall off that result. Run it on load, after every purchase, and before you reveal anything gated.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

async function refreshEntitlements() {
  if (!isDespia) return
  const { restoredData } = await despia('getpurchasehistory://', ['restoredData'])
  const active = (restoredData ?? []).filter(p => p.isActive)
  setPremium(active.some(p => p.entitlementId === 'premium'))
}

refreshEntitlements()                               // on load
window.onRevenueCatPurchase = refreshEntitlements   // and after every purchase
```

The paywall now closes when `refreshEntitlements` finds an active entitlement, the instant the payment clears. `getpurchasehistory://` reads the store directly, so there is nothing to wait on and nothing to sync. The `entitlementId` you match on is shared across both stores, so this code does not care which platform it is running on. More on how that entitlement is wired below.

Note the placement. The `despia()` call sits inside the `isDespia` gate. The `window.onRevenueCatPurchase` assignment sits outside it, because it is a callback the runtime invokes, not a call you make.

## You do not need webhooks to protect the server

The frontend restore is right for showing and hiding UI. It reflects what the client claims, which is fine for the interface and not enough for anything a manipulated client could walk away with. For a request that costs money or serves paid content, verify server-side.

The usual way to do that is webhooks: RevenueCat posts subscription events, you catch them, and you keep a copy of the state in your database. For an AI-built app that is a lot of moving parts to stand up and keep healthy, failed deliveries, replays, ordering, signature checks, and you do not need any of it. Ask RevenueCat directly at the moment the request is made.

```javascript
// server-side only, the secret key never ships inside the app
async function hasActiveEntitlement(appUserId, entitlementId) {
  const id = encodeURIComponent(appUserId)   // anon ids carry a $RCAnonymousID: prefix
  const res = await fetch(
    `https://api.revenuecat.com/v2/projects/${PROJECT_ID}/customers/${id}/active_entitlements`,
    { headers: { Authorization: `Bearer ${RC_SECRET_KEY}` } }
  )
  if (res.status === 404) return false   // a customer RevenueCat has never seen
  if (!res.ok) throw new Error(`RevenueCat ${res.status}`)
  const { items = [] } = await res.json()
  return items.some(e => e.entitlement_id === entitlementId)
}
```

Two things decide whether this works. The `appUserId` you check has to match the `external_id` you launched the paywall with, or RevenueCat resolves a different customer and correctly returns nothing. And in sandbox this endpoint sometimes reads empty even when the entitlement is active, so during testing confirm against the older subscriber lookup before you conclude anything is broken.

What you give up without webhooks is real-time reaction. A cancellation will not cut a live session on the same second, you find out at the next check, which is the user's next request in practice. If a feature truly needs instant state, add a webhook then, as a layer on top of this, not a prerequisite for it.

## While you are here, check the detection code

There is a second tell that this code was generated without reading the docs. The environment check often looks for globals like this:

```javascript
// wrong: these are not Despia
const inApp = !!(window.ReactNativeWebView || window.webkit?.messageHandlers)
```

Those are React Native and Capacitor globals. A Despia runtime does not use them. It usually still works by accident, because the real gate falls through to the user agent, but it is a sign the whole integration was guessed at rather than read. The only supported detection is the user agent, and you want the full suite, not just the top-level check, because iOS and Android are not interchangeable here:

```javascript
const ua = navigator.userAgent.toLowerCase()
const isDespia        = ua.includes('despia')
const isDespiaIOS     = isDespia && (ua.includes('iphone') || ua.includes('ipad'))
const isDespiaAndroid = isDespia && ua.includes('android')
```

In a server-rendered framework like Next.js or Remix, `navigator` is undefined during the render pass, so guard the check with `typeof navigator !== 'undefined'` or run it in an effect, or the app throws on boot.

You need the platform split because the two stores do not share products. The same subscription is registered separately in App Store Connect and Google Play, with a different product id on each side, so a direct purchase has to pick the id per platform:

```javascript
const productId = isDespiaIOS ? 'pro_monthly_ios' : 'pro_monthly_android'
if (isDespia) {
  despia(`revenuecat://purchase?external_id=${userId}&product=${productId}`)
}
```

The entitlement you read back on restore is the opposite. Attach both platform products to a single entitlement in RevenueCat and the same `entitlementId` comes back on iOS and Android, which is why the restore fix above stays platform agnostic while the purchase call does not.

Gate every native call on `isDespia`. If the purchase code was written against globals that do not exist, the confirmation logic was probably written the same way, which is how you ended up with a paywall that waits on a server that never answers.

## Add this to your app

Native in-app purchases, restores, and a server-side entitlement check are one JavaScript call and one API request away, inside the web codebase you already ship. No webhook pipeline, no second native project, no Xcode.

[Read the full RevenueCat reference at setup.despia.com](https://setup.despia.com/payments/revenuecat/reference)