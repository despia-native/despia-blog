---
title: "Restore Purchases Button Missing? Fix Guideline 3.1.1"
seoTitle: "Restore Purchases Button Missing? Fix Guideline 3.1.1"
seoDescription: "Apple rejects apps with no Restore Purchases button under Guideline 3.1.1. Here is the fix, the code for iOS and Android, and the mistake to avoid."
datePublished: 2026-07-24T10:26:33.337Z
cuid: cmryspb6r00000ahx7ilrdzgj
slug: restore-purchases-button-missing-fix-guideline-3-1-1
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/dbb538bd-1036-4460-b03f-77bcbafe0650.png
tags: android-app-development, ios, android, ios-app-development, revenuecat, appreview

---

Guideline 3.1.1 catches a lot of apps on their second submission. Billing works, the app ships, and Apple sends it back saying purchases can be restored but there is no way to restore them. Whatever you built the app in, the requirement is identical: a distinct button that restores purchases, and it cannot be an automatic restore on launch. Google Play expects the same entitlement recovery, and the same code covers it. This is not a bug, it is a missing required element, which is exactly why it is quick to fix once you know it is the cause. Here is what the reviewers want and the two clean ways to give it to them.

## Read the rejection literally

Apple's wording matters here. Restoring previously purchased products should happen when a user taps a "Restore" button, and automatically restoring on launch will not resolve the issue. So a silent check at startup, even a correct one, fails review. The reviewer needs a visible control they can tap and watch complete.

## Option one: your own Restore button

In a Despia app, restore reads the device's purchase history and re-grants any active entitlement:

```javascript
import despia from 'despia-native'

async function restorePurchases() {
    const data   = await despia('getpurchasehistory://', ['restoredData'])
    const active = (data.restoredData ?? []).filter(p => p.isActive)

    if (active.length === 0) {
        showMessage('No active purchases found.')
        return
    }
    if (active.some(p => p.entitlementId === 'premium')) unlockPremium()
    if (active.some(p => p.entitlementId === 'no_ads'))  removeAds()
}
```

`getpurchasehistory://` runs on-device, so it is instant and offline-capable. Gate the button on the runtime check so it only appears in the native app:

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

if (isDespia) {
    showRestoreButton(restorePurchases)
}
```

## Option two: the Customer Center

If you would rather not build and maintain restore UI, Despia surfaces RevenueCat's Customer Center, the native sheet where users restore, manage, and cancel:

```javascript
if (isDespia) {
    despia(`revenuecat://center?external_id=${userId}`)
}

window.onRevenueCatCenter = (event) => {
    if (event.event === 'restoreCompleted') restorePurchases()
}
```

The `despia()` call stays inside the `isDespia` gate, the callback is assigned outside it. This satisfies the distinct-restore requirement and gives users subscription management in the same screen.

## The one mistake to avoid

The Customer Center event payload includes an entitlements array, and it is tempting to trust it. Do not treat it as final. The event tells you a restore finished, but the device store is the source of truth. Re-run your own `getpurchasehistory://` check on `restoreCompleted`, as above, so your app reflects what the store actually holds rather than what a client-side event reported. This is the difference between a restore that looks right and one that is right.

## Restore shows the UI, the server protects the value

The restore above reflects what the client claims. That is correct for showing and hiding parts of the interface, and it is not enough to protect anything a manipulated client could walk away with. For any request that costs money or serves protected content, verify server-side at request time by asking RevenueCat directly whether the user is entitled. No webhook, no subscription table to keep in sync. The full pattern, frontend restore plus one server check, is here: [WebView in-app purchases without webhooks](https://blog.despia.com/webview-in-app-purchases-without-webhooks-revenuecat).

## Google Play, and why there is no Android code

Apple names this rejection Guideline 3.1.1. Google Play does not use a numbered guideline, but the requirement is just as explicit: its Play Billing documentation states that supporting restore is required for all developers, and digital purchases fall under the Google Play Payments policy. Play ties purchases to the user's Google account, so on a reinstall the entitlement comes back when the app queries the store, and users can also restore from the Play subscriptions center. You still implement the restore path.

The point that matters: there is no Android-specific code. The same `restorePurchases()` above covers both stores. `getpurchasehistory://` queries whichever store the device is running, and if you attached your iOS and Android products to one RevenueCat entitlement, both return the same `entitlementId`. The response also carries a `store` field (`app_store` or `play_store`) if you ever need to branch. Show the one button on both platforms and the restore flow is identical everywhere.

## Where to place it

Reviewers look near the purchase options and in account settings. Put the button, or the Customer Center entry, on both the paywall and the account screen. A restore path the reviewer cannot find gets treated as absent, and the rejection repeats.

## The checklist

*   A distinct, tappable Restore control, never an on-launch restore
    
*   Restore re-grants active entitlements from the store
    
*   On Customer Center, re-check entitlements on `restoreCompleted`
    
*   Reachable from the paywall and the account screen
    

## Try Despia

Ship your web app to iOS and Android with native In-App Purchase, restore, and the Customer Center built in, from a single codebase. No Xcode, no native project to maintain.

[Learn more at setup.despia.com](https://setup.despia.com)