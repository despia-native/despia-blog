---
title: "How to Add a RevenueCat Paywall to a WebView App"
seoTitle: "How to Add a RevenueCat Paywall to a WebView App"
seoDescription: "Add in-app purchases to a web app shipped with Despia. Build a RevenueCat paywall, launch it natively, and keep Stripe for web checkout."
datePublished: 2026-07-04T13:29:09.767Z
cuid: cmr6ef3vu00000akrd27s1th8
slug: how-to-add-a-revenuecat-paywall-to-a-webview-app
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/52a9f052-fb12-436e-a0d3-392b8df63a2f.jpg
tags: webview, iap, playstore, appstore, revenuecat

---

You built a web app, wired up Stripe, and now you want it on the App Store and Google Play with a working subscription. The moment you try to sell that subscription inside the native app, you hit a wall: Apple and Google will not let you charge for digital goods through Stripe. This post walks the real path, from store products to a live paywall, using RevenueCat inside a Despia app, while your Stripe checkout stays exactly where it is for the web.

## Why you cannot just use Stripe in the app

Apple and Google both require digital subscriptions and one-time unlocks bought inside a native app to go through their own billing, StoreKit on iOS and Play Billing on Android. Ship a Stripe checkout for a digital plan inside the app and it gets rejected at review. This is not a Despia constraint, it is a store policy that applies to every native app.

Stripe is fine everywhere the store rules do not reach. Purchases on your website stay on Stripe. What changes is only the in-app flow, and that is the part RevenueCat handles, because RevenueCat wraps StoreKit and Play Billing behind one API and one dashboard. So the end state is two payment surfaces that coexist: Stripe on the web, RevenueCat in the app, with your backend as the source of truth for who has access.

## Two ways to build the paywall

Despia gives you two paths, and you should pick before you start.

The first is the **native RevenueCat Paywall**. You design the paywall in the RevenueCat dashboard, and Despia launches it as a native screen with one call. No purchase UI to build, no price formatting, no store-specific edge cases. This is the path most builders want, and it is the one this post leads with.

The second is the **custom paywall**, where you design the paywall in your own web UI and call RevenueCat only to run the purchase. You get full design control and take on more of the wiring yourself. It is covered at the end.

## Step 1: Create your products in the stores

RevenueCat does not replace the stores, it sits on top of them, so the products have to exist there first.

On iOS, open App Store Connect and create your subscriptions or in-app purchases, each with a product ID (for example `pro_monthly`). On Android, do the same in Google Play Console under the subscriptions and in-app products sections. Product IDs do not have to match across stores, since RevenueCat maps them for you, but keeping them consistent saves confusion later.

One store difference worth knowing up front: Google Play needs a signed build uploaded to a testing track before its billing will return products, while iOS surfaces them once they are in the right state in App Store Connect. If your paywall shows up empty on Android first, this is almost always why.

## Step 2: Set up entitlements, products, and offerings in RevenueCat

RevenueCat has a small object model, and getting it right is most of the job. There are four pieces:

An **entitlement** is the thing a user unlocks, for example `premium`. Your app checks for the entitlement, never for a specific product. This is what lets you change prices and plans later without touching app logic.

A **product** is the store SKU you created in Step 1, imported into RevenueCat and attached to an entitlement. A monthly and an annual plan are two products that both grant the same `premium` entitlement.

An **offering** is the set of products you show on a paywall, grouped into **packages** (monthly, annual, and so on). Most apps start with a single offering called `default`. You can add more later for experiments or promotions and switch between them without an app update.

Create the entitlement, import and attach your products, then build an offering with the products as its packages. That offering is what the paywall reads from.

For RevenueCat to validate receipts, connect each store to it under Project settings, Apps. The iOS app needs an App Store Connect API key and an in-app purchase key, the Android app needs your Google Play service account JSON. Without these, purchases go through on the device but never verify.

## Step 3: Build the paywall in RevenueCat

RevenueCat's Paywalls let you configure the entire paywall view remotely, so you can change copy, layout, and pricing without shipping a new build. In the dashboard, pick a template, start from a blank canvas, or describe what you want in the AI editor and adjust from there.

The paywall pulls its prices and plans from the offering you attached, so the packages you set up in Step 2 are what render. Localized prices come straight from the stores, so you do not format currency yourself. When the design is done, attach the paywall to your offering and it is ready to launch.

## Step 4: Enable RevenueCat in Despia, then rebuild

In the Despia Editor, go to App, then Settings, then Integrations, then RevenueCat. Toggle it on and paste in both keys from RevenueCat's Project settings, API keys: the iOS Public SDK Key and the Android Public SDK Key. There are two, one per platform, and both have to match exactly.

Then rebuild the app. This is the step people miss. The RevenueCat SDK is compiled into the binary, so it cannot ship over the air like your web content. Until you trigger a fresh build, the toggle can read enabled while every purchase fails silently and every entitlement check comes back empty. If billing suddenly stops working right after you change these settings, rebuild before anything else.

If you are building in Lovable or a similar stack, install the SDK in your project:

```bash
npm install despia-native
```

```javascript
import despia from 'despia-native'
```

This is the same `despia()` function referenced throughout the docs, just imported as a module.

## Step 5: Launch the paywall

Now the app side. Detect whether you are running inside the Despia runtime, and branch: native paywall in the app, your existing web checkout on the web. The canonical check is the user agent, never a global object.

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

function openPaywall(userId) {
  if (isDespia) {
    // native RevenueCat paywall, reads from the "default" offering
    despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)
  } else {
    // web stays on Stripe (or RevenueCat web billing)
    startStripeCheckout(userId)
  }
}
```

Pass your own user ID as `external_id`. RevenueCat ties the purchase to that ID, which is what lets the same account stay in sync across the app and the web.

## Step 6: Confirm the purchase on your backend, not the client

When a purchase completes, the Despia runtime calls a global function, `onRevenueCatPurchase()`. Assign it outside the `isDespia` gate so it exists regardless of environment.

The important part: do not unlock access here. A client-side callback is a signal, not proof. The source of truth is the RevenueCat webhook hitting your backend. Treat the callback as your cue to start checking whether that webhook has landed.

```javascript
window.onRevenueCatPurchase = async function () {
  // purchase reported on the client, now confirm it server-side
  await pollForEntitlement()
}
```

Set up a webhook in RevenueCat pointing at your backend to receive purchase, renewal, and cancellation events. On each event, validate it and update the user's status in your database. That database, not the app, decides who has `premium`. This is what keeps entitlements correct across renewals, refunds, and multiple devices.

## Step 7: Check entitlements and restore purchases

On load, and when a user reinstalls or switches devices, read their active entitlements from the native layer. Despia returns restore data you can filter for what is active.

```javascript
async function checkEntitlements() {
  const data = await despia('getpurchasehistory://', ['restoredData'])
  const active = (data.restoredData ?? []).filter(p => p.isActive)
  if (active.some(p => p.entitlementId === 'premium')) unlockPremium()
}

checkEntitlements()
```

This doubles as your Restore Purchases handler, which Apple requires any app with non-consumable purchases to provide. Wire it to a visible button so a returning user can recover access without paying again. If you would rather not build the manage-and-restore UI yourself, `despia("revenuecat://center?external_id=" + userId)` opens RevenueCat's native Customer Center, where users can restore, manage, or cancel from one sheet.

One caveat that matters for this exact setup: `getpurchasehistory://` only sees purchases made through the native stores. A subscription the same user bought on your website through Stripe will not appear here. So when you have both surfaces, check your backend for entitlement first, and treat the native store query as the fallback for app-side purchases, not the primary source of truth.

## The custom paywall alternative

If you need the paywall to match your app's design exactly, build the UI yourself and call RevenueCat only for the purchase. The product ID format differs by platform here: iOS takes the plain product ID, Android needs the subscription group as a prefix, `group:productId`.

```javascript
if (isDespiaIOS) {
  despia(`revenuecat://purchase?external_id=${userId}&product=monthly_premium_ios`)
} else if (isDespiaAndroid) {
  despia(`revenuecat://purchase?external_id=${userId}&product=premium:monthly_premium_android`)
}
```

Derive the platform from the same user agent string:

```javascript
const ua = navigator.userAgent.toLowerCase()
const isDespiaIOS = ua.includes('despia') && (ua.includes('iphone') || ua.includes('ipad'))
const isDespiaAndroid = ua.includes('despia') && ua.includes('android')
```

The success callback is the same `onRevenueCatPurchase()` used by the paywall flow, so the rest holds unchanged: do not grant access on the callback, confirm through the webhook on your backend, and use the same entitlement check for restores. You trade the ready-made native paywall for full design control, and nothing else about the wiring changes.

## Where Stripe fits at the end

You do not remove Stripe. Web visitors keep buying through it, app users buy through RevenueCat, and both write to the same backend against the same `premium` entitlement. If you would rather run a single source of truth for purchases across both surfaces, RevenueCat can track web purchases too and you can route web users through its web billing instead. Either way, the app side is done: one call to launch, one webhook to trust, one entitlement to check.

## Add this to your app

Every call in this post, the paywall launch, the purchase, the entitlement check, is one JavaScript function inside your existing web codebase. No Xcode, no native purchase code, no second project to maintain.

[Read the RevenueCat reference at setup.despia.com](https://setup.despia.com/payments/revenuecat/introduction)