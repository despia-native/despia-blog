---
title: "WebView In-App Purchases Without Apple Rejection"
seoTitle: "WebView In-App Purchases Without Apple Rejection"
seoDescription: "How to wire RevenueCat in-app purchases through Despia so the native purchase sheet shows for Apple reviewers and clears App Review on the first try."
datePublished: 2026-06-30T14:16:58.520Z
cuid: cmr0qd6rs00000ahu31834klh
slug: webview-in-app-purchases-without-apple-rejection
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/0847e108-7e4d-4bdb-9527-f54bd2e9da95.jpg
tags: webview, pwa, iap, appstore

---

You added payments, submitted, and got the 3.1.1 rejection. Or worse: it works on your phone, works on TestFlight, and the reviewer says the purchase does nothing. Both of these come down to the same handful of mistakes, and none of them are hard to fix once you know what Apple is actually looking at. This is the path to a native StoreKit purchase, through RevenueCat, in a Despia app, that passes review the first time.

## Why Apple rejected you

Almost every IAP rejection is one of three things.

The first is selling digital goods through anything other than Apple's billing. A Stripe checkout, a JotForm payment, a link to your website. Apple does not care that it is embedded in your app. If the thing being unlocked is digital and consumed inside the app, it has to go through StoreKit. Guideline 3.1.1.

The second is the reviewer never seeing a purchase sheet. The app builds, the button is there, they tap it, and nothing happens. To a reviewer, a purchase flow that does not visibly launch is a broken or hidden purchase flow, and it gets rejected. This is the one that drives people up the wall, because it works everywhere except the review sandbox.

The third is no restore. Apple requires a way to restore previous non-consumable and subscription purchases. No restore button, automatic rejection.

Despia plus RevenueCat handles all three, but only if the setup and the code are both right. Either one wrong and you get a paywall that previews perfectly in the RevenueCat dashboard and fails on a real device.

## What Despia is doing here

Despia does not implement StoreKit itself. It integrates RevenueCat, which sits on top of StoreKit on iOS and Google Play Billing on Android. You keep your one web codebase, the same app you already built, and ship it as a real native binary. The web layer triggers a native RevenueCat paywall, RevenueCat owns the purchase, the receipt validation, the entitlements, and the renewals. You do not maintain a second native project and you do not write Swift.

That means the purchase sheet a reviewer sees is the real Apple sheet, sandbox-priced during review, going through Apple's billing. That is exactly what they want to see.

## The setup that has to exist before any code

This is the part people skip, and it is why a paywall previews fine and dies at runtime. Get all of it in place first.

*   Create the product in App Store Connect (subscription or non-consumable), with a real price and the metadata filled in.
    
*   Import the product into RevenueCat and attach it to an entitlement.
    
*   Build at least one offering in RevenueCat. The paywall launches an offering, not a raw product.
    
*   Upload your App Store Connect In-App Purchase Key (the `.p8`) into RevenueCat, with the Key ID and Issuer ID. This is separate from your iOS SDK key that starts with `appl_`. Without the `.p8`, paywalls render in the dashboard and fail the moment they hit a device, because RevenueCat cannot talk to Apple to fetch products.
    
*   Enable RevenueCat in Despia under App, Settings, Integrations, RevenueCat, and add your API key.
    

One more thing that catches people: RevenueCat needs a live store product. A sandbox-only or test setup with no real App Store Connect product and no offering will not bridge. The product has to exist for real, even though review runs it in sandbox.

## Launch the sheet, do not await it

Here is the single biggest code mistake. The RevenueCat scheme launches a native sheet. It does not return a StoreKit response, there is no promise to await, no callback baked into the call. If you wrap it in an async handler waiting for a result, or expect a return value before continuing, it silently does nothing in environments like the review sandbox while still looking fine on your own device. That is the "works on TestFlight, broken for the reviewer" bug, exactly.

Launch the paywall and let it go:

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

function openPaywall(userId) {
  if (isDespia) {
    despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)
  } else {
    // web users get the hosted RevenueCat paywall instead
    window.location.href = `https://pay.rev.cat/<your_token>/${encodeURIComponent(userId)}`
  }
}
```

If you are driving your own UI and want to buy a specific product instead of showing RevenueCat's hosted paywall, the direct form is:

```javascript
despia(`revenuecat://purchase?external_id=${userId}&product=monthly_premium`)
```

Same rule. Fire it, do not await it.

The result comes back through a global callback, assigned outside the Despia gate so it exists regardless of platform. Despia calls `onRevenueCatPurchase` when a purchase completes on the client:

```javascript
window.onRevenueCatPurchase = () => {
  // a purchase happened on the client. do NOT unlock here.
  // start polling your backend for the verified webhook event.
  pollEntitlements()
}
```

Note where each piece lives. The `despia()` calls sit inside the `isDespia` check. The `window.onRevenueCatPurchase` callback is assigned outside it. That split matters: the native calls only make sense in the native runtime, but the callback should be defined no matter where the code loads.

## Add restore, or you get rejected for it

Restore is not optional. Read the user's existing purchases through the purchase history scheme and unlock from there:

```javascript
async function pollEntitlements() {
  const data = await despia('getpurchasehistory://', ['restoredData'])
  const active = (data.restoredData ?? []).filter(p => p.isActive)
  if (active.some(p => p.entitlementId === 'premium')) {
    unlockPremium()
  }
}
```

Wire that same function to a visible "Restore purchases" button. That button is what a reviewer looks for, and its absence is a clean rejection on its own.

## Grant access from the webhook, not the client

The client callback tells you a purchase probably happened. It is not proof. Anyone can call a function. The source of truth is the RevenueCat webhook hitting your backend, validating the event, and updating the user's entitlement in your database. So the flow is: client purchase fires the callback, the callback starts polling your backend, the backend has been updated by the RevenueCat webhook, and only then do you unlock.

Unlocking premium directly inside `onRevenueCatPurchase` is how you ship a paywall that anyone can bypass from the console. Treat the callback as a hint to go check the real state, never as the grant itself.

## Why it works on TestFlight but the reviewer says it is broken

Two reasons, and they are worth saying plainly because they cost people days.

One: the await mistake above. The call silently fails in the review sandbox while passing on your device. Strip any code that awaits a response from the scheme and the sheet will launch for the reviewer.

Two: the IAP products are not attached to the build you submitted. In App Store Connect, the in-app purchase has to be included with the version you send to review, with its own metadata and screenshot complete. If the product is sitting there unattached, the reviewer's build genuinely has nothing to buy, no matter how correct your code is.

Fix both and the rejection clears.

## Get it on the stores

Take the app you already built, wire RevenueCat through one JavaScript call, and ship a real StoreKit purchase to iOS and Android without Xcode or a Mac. Code signing and submission run from the browser, from your single web codebase.

[See the RevenueCat setup in the docs at setup.despia.com](https://setup.despia.com)