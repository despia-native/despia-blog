---
title: "RevenueCat on Base44: Why It Fails and the Fix"
seoTitle: "RevenueCat on Base44: Why It Fails and the Fix"
seoDescription: "RevenueCat cannot reach StoreKit inside a Base44 WebView, so the paywall fails. Here is why, and how Despia bridges Base44 to native billing."
datePublished: 2026-07-05T14:14:56.575Z
cuid: cmr7vhu09000009krevtlaz2c
slug: revenuecat-on-base44-why-it-fails-and-the-fix
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/744501d4-ea46-48d7-b6e5-8bc47df71974.png
tags: webview, appstore, googleplay, revenuecat, base44, storekit, android-billing

---

A builder on the RevenueCat community forum hit a wall that a lot of Base44 users hit: they wanted App Store subscriptions, Base44 only integrates Stripe, Apple does not allow Stripe for digital goods, and every attempt to wire in RevenueCat threw errors. The [full thread is here](https://community.revenuecat.com/sdks-51/i-cannot-add-revenuecat-on-base44-as-a-subscription-paywall-please-help-7701). RevenueCat's staff answer was correct about the cause, and it also pointed straight at the fix without naming it. This post fills in the fix.

## Why RevenueCat cannot see the App Store from a Base44 app

The diagnosis in that thread is right. When Base44 exports for the App Store, it produces a native iOS wrapper, but your actual app runs inside a WebView. RevenueCat's iOS SDK is native code that talks directly to Apple's StoreKit framework. StoreKit is what shows the purchase sheet, validates the receipt, and manages the subscription. None of that exists in a WebView on its own.

So the missing piece is not RevenueCat. It is the native billing connection inside the Base44 app. Base44 does not ship a native bridge to StoreKit or to the RevenueCat SDK, which means there is nothing for the JavaScript in your web app to call. Errors like the one the original poster saw are the symptom. The web layer is trying to reach a native API that was never compiled into the binary.

This is not a Base44 flaw so much as a property of any web-in-a-WebView build. The App Store purchase flow is native by design, and Apple will not accept Stripe or any web checkout for digital purchases under Guideline 3.1.1.

## RevenueCat's own answer already described the solution

Read the staff reply closely and it says the thing that works: use a web-to-native wrapper that supports RevenueCat through a native bridge, and your Base44 app URL can still be used. In other words, keep the app you built, but run it inside a native runtime that has the RevenueCat SDK compiled in and exposed to your web code.

The other suggestion in the thread was to rebuild the app in a different no-code builder that RevenueCat supports out of the box. That works too, but it means starting over. You throw away the Base44 project and rebuild the whole thing somewhere else just to get a paywall. If you like what you built in Base44, that is a heavy price for one feature.

Despia is the first option. Your Base44 app stays the source of truth, and the native billing layer comes from the runtime.

## How Despia closes the gap

Despia takes the web app you already have, the Base44 URL included, and ships it as a real native binary for iOS and Android. The web code runs in the platform WebView, but the binary also contains the native RevenueCat SDK, and Despia exposes it to your web layer through a single JavaScript function. You do not write Swift, you do not manage an Xcode project, and you do not maintain a second app. One codebase, still Base44, now with native StoreKit and Google Play billing underneath it.

Concretely, once RevenueCat is enabled on your Despia build, these calls work from your existing Base44 JavaScript:

```javascript
import despia from 'despia-native'

// Only call native schemes when running inside the Despia runtime.
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

function openPaywall(userId) {
  if (isDespia) {
    despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)
  } else {
    // Web visitors get a RevenueCat Web Purchase Link instead.
    window.location.href = `https://pay.rev.cat/<your_token>/${encodeURIComponent(userId)}`
  }
}
```

Checking what a user owns is a direct query to the native store, no backend required:

```javascript
async function checkEntitlements() {
  const data = await despia('getpurchasehistory://', ['restoredData'])
  const active = (data.restoredData ?? []).filter(p => p.isActive)

  if (active.some(p => p.entitlementId === 'premium')) unlockPremium()
}

// Run on load, and again whenever a purchase confirms.
checkEntitlements()
window.onRevenueCatPurchase = checkEntitlements
```

`launchPaywall` opens the native RevenueCat paywall you configured in the dashboard. `getpurchasehistory://` reads active entitlements straight from StoreKit or Play Billing. The `onRevenueCatPurchase` callback fires when the store confirms a transaction. That is the native bridge the RevenueCat forum answer said you needed, provided by the runtime instead of by hand.

## What the setup actually looks like

The RevenueCat side is the standard setup, nothing Base44-specific. The Despia side is two fields and a rebuild.

1.  Create your app in RevenueCat, add the iOS and Android apps, and upload the App Store Connect and Google Play credentials so RevenueCat can validate receipts.
    
2.  Create your entitlements (for example `premium`), import your products, and group them into offerings that drive the paywall.
    
3.  Copy the iOS and Android Public SDK Keys from RevenueCat.
    
4.  In the Despia Editor, open App > Settings > Integrations > RevenueCat and paste both keys.
    
5.  Rebuild the app.
    

The [full RevenueCat setup for Despia is in the docs](https://setup.despia.com/payments/revenuecat/introduction), including entitlement checks, the Customer Center, webhooks, and the web fallback.

## The one step that silently breaks

The RevenueCat SDK is compiled into the binary, so saving the keys is not enough on its own. If you paste the keys and skip the rebuild, the app still runs the old binary with no RevenueCat inside it. Purchases fail quietly and entitlement checks come back empty, which looks exactly like a broken integration. It is not. Trigger a fresh build after adding the keys and the schemes start working in production. This is the single most common reason a correctly configured Despia paywall appears dead.

## Get your Base44 app on the stores with working payments

Keep the Base44 app you already built. Despia ships it to the App Store and Google Play as a native binary, gives it the native RevenueCat SDK and 50+ other device features through one JavaScript call, and updates the web layer over the air without a resubmission.

[Read the RevenueCat setup in the docs](https://setup.despia.com/payments/revenuecat/introduction) or [start building at despia.com](https://despia.com).