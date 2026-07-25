---
title: "Base44 Restore Purchases: Fix Guideline 3.1.1"
seoTitle: "Base44 Restore Purchases: Fix Guideline 3.1.1"
seoDescription: "Apple rejected your Base44 app under Guideline 3.1.1 for no Restore Purchases button. Here is why, and the exact code to add the restore flow."
datePublished: 2026-07-24T10:27:51.551Z
cuid: cmrysqzjk000004l70wovdpi5
slug: base44-restore-purchases-fix-guideline-3-1-1
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/4acb2afd-3b11-419a-bc69-208a405fb52d.png
tags: android-app-development, ios, android, ios-app-development, revenuecat, base44, appreview

---

Apple rejected your Base44 app under Guideline 3.1.1, saying it offers In-App Purchases that can be restored but has no "Restore Purchases" feature. AI-built paywalls ship the buy button and skip the restore, so this rejection hits Base44 apps almost by default. It is not a bug, it is a missing required element, and it is quick to add once you know it is the cause.

## What the guideline demands

If a user can restore a purchase, for example after reinstalling or switching devices, you must give them a distinct button that does it. Apple states directly that restoring automatically on launch does not satisfy the rule. It has to be a control the user can tap.

## The Base44 billing problem underneath this

There is a catch that is specific to Base44. Base44 has no native billing yet, and Apple rejects Stripe for digital goods, so a Base44 app on its own cannot process In-App Purchases at all, let alone restore them. To get a real restore flow you need StoreKit, which Despia provides through its built-in RevenueCat integration. Once that is in place, restore is a single native call.

## The restore code

Query the native store for prior purchases and re-grant anything still active:

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

`getpurchasehistory://` runs against the device store, so it is instant and works offline. Wire that function to a visible button:

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// Render a "Restore Purchases" button when running in the native app
if (isDespia) {
    showRestoreButton(restorePurchases)
}
```

Put the button somewhere obvious, the paywall or the account screen, so the reviewer finds it without hunting.

## Protect the paid features on the server

The restore reflects what the client claims, which is right for toggling the UI and not enough to protect anything valuable. When a Base44 app calls a backend function for something that costs money or serves premium content, verify entitlement server-side at request time by asking RevenueCat directly. No webhook, no subscription table to keep in sync. The full pattern, using a Base44 server function for the check, is here: [Base44 in-app purchases without webhooks](https://blog.despia.com/base44-in-app-purchases-without-webhooks-revenuecat).

## Google Play, same button

Apple names this Guideline 3.1.1. Google Play has no equivalent number, but its Play Billing docs state that supporting restore is required for all developers, so ship the button on Android too. There is no separate Android code: the same `restorePurchases()` runs against whichever store the device uses, and one RevenueCat entitlement returns the same `entitlementId` on both.

## Where to place it so review passes

The reviewer looks for the button near the purchase options and in account settings. The paywall plus a profile or settings screen covers both. Do not bury it. A restore control the reviewer cannot locate is treated the same as one that does not exist.

## The checklist

*   Real StoreKit billing in place (RevenueCat via Despia)
    
*   A distinct, tappable Restore Purchases button
    
*   Button placed on the paywall or account screen
    
*   Restore re-grants active entitlements, not just a silent launch check
    

## Get it on the stores

Point Despia at your Base44 app, add native In-App Purchase and a Restore button from your existing code, and ship to iOS and Android without Xcode or a Mac.

[See the setup docs at setup.despia.com](https://setup.despia.com)