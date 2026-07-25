---
title: "How to Add a Restore Purchases Button (iOS + Android)"
seoTitle: "How to Add a Restore Purchases Button (iOS + Android)"
seoDescription: "How to add a Restore Purchases button that passes App Store review and works on iOS and Android, with the exact code and where to place it."
datePublished: 2026-07-24T10:31:46.079Z
cuid: cmrysw0ij00010ai08a2i6rmf
slug: how-to-add-a-restore-purchases-button-ios-android
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/d3c5277d-26e4-4f54-a91a-ceeecdac6d44.png
tags: android-app-development, ios, android, webview, pwa, ios-app-development, revenuecat, appreview

---

If your app sells subscriptions or non-consumable purchases, Apple requires a Restore Purchases button under Guideline 3.1.1, and a missing one is a fast, automatic rejection. Here is how to add a Restore Purchases button that passes review, wired to the native store and working on both iOS and Android.

## What Guideline 3.1.1 requires

When an In-App Purchase can be restored, users need a distinct button that starts the restore. Apple says plainly that restoring automatically on launch does not satisfy the guideline. The reviewer wants a tappable control, and they will look for it near your purchase options and in account settings.

## Restore against the native store

In a Despia app, purchases run through StoreKit via RevenueCat, and restore is a single call that reads the device's purchase history:

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
}
```

`getpurchasehistory://` queries the store on-device, so it is instant and works offline. It returns every prior purchase with an `isActive` flag, and you re-grant access for the ones still valid.

## Wire it to a real button

The `despia()` calls only work inside the native app, so gate the button on the runtime check and keep your web build clean:

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

if (isDespia) {
    showRestoreButton(restorePurchases)
}
```

## The frontend restore is for the UI, not the value

The restore call reflects what the client claims, which is exactly what you want for showing and hiding parts of the interface. It is not enough to protect anything a manipulated client could take. For any request that costs money or serves protected content, verify server-side at request time by asking RevenueCat directly. That keeps the store as the source of truth on both sides with no webhook and no database copy to maintain. The full frontend-plus-server pattern is here: [WebView in-app purchases without webhooks](https://blog.despia.com/webview-in-app-purchases-without-webhooks-revenuecat).

## Google Play, same button

Apple names this Guideline 3.1.1. Google Play has no numbered equivalent, but its Play Billing docs state that supporting restore is required for all developers, so ship the button on Android too. There is no Android-specific code to write: the same `restorePurchases()` queries whichever store the device runs, and a single RevenueCat entitlement returns the same `entitlementId` on both platforms.

## Placement decides whether it passes

A restore button the reviewer cannot find is treated like one that does not exist. Put it on the paywall and on the account or settings screen. Both are places the reviewer checks, and covering both removes any ambiguity.

## The checklist

*   A distinct, tappable Restore Purchases button
    
*   Restore reads the store and re-grants active entitlements
    
*   Not a silent on-launch restore
    
*   Reachable from the paywall and account screen
    

## Get it on the stores

Ship your web app to iOS and Android with native In-App Purchase and a compliant restore flow, from the codebase you already have. Signing and submission run from the browser.

[See the setup docs at setup.despia.com](https://setup.despia.com)