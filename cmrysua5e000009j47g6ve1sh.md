---
title: "Lovable Restore Purchases: Fix Guideline 3.1.1"
seoTitle: "Lovable Restore Purchases: Fix Guideline 3.1.1"
seoDescription: "Apple rejected your Lovable app under Guideline 3.1.1 for no Restore Purchases button. Here is why it happens and the code to add the restore flow."
datePublished: 2026-07-24T10:30:25.265Z
cuid: cmrysua5e000009j47g6ve1sh
slug: lovable-restore-purchases-fix-guideline-3-1-1
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/460f17d4-e98c-4b3b-84cc-0bed4f292532.png
tags: android-app-development, ios, android, ios-app-development, revenuecat, lovable, appreview

---

Your Lovable app was rejected under Guideline 3.1.1 because it offers In-App Purchases that can be restored but has no "Restore Purchases" button. AI-built paywalls generate the purchase flow and leave out the restore, so this is one of the most common rejections for Lovable apps. It is a missing required element, not a bug, and it is one of the fastest to fix once billing is working.

## What Apple wants

If purchases can be restored, users need a distinct button that triggers the restore. Apple is explicit that automatically restoring on app launch does not count. The reviewer wants a control they can tap and watch work.

## The Lovable billing gap first

Lovable exports through Capacitor, which does not include native billing. Since Apple rejects Stripe for digital goods, a plain Lovable build has no compliant way to sell or restore In-App Purchases. Restoring a purchase requires StoreKit, and Despia ships that through a built-in RevenueCat integration, no plugin and no native project. With that in place, restore is one call.

## The restore code

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

`getpurchasehistory://` reads the device store directly, so it returns instantly and works offline. Attach it to a button that only renders in the native app:

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

if (isDespia) {
    showRestoreButton(restorePurchases)
}
```

## The zero-effort option: Customer Center

If you would rather not build restore UI at all, Despia exposes RevenueCat's Customer Center, a native sheet where users can restore, manage, or cancel their subscription:

```javascript
if (isDespia) {
    despia(`revenuecat://center?external_id=${userId}`)
}

// Re-check entitlements after the user restores inside the sheet
window.onRevenueCatCenter = (event) => {
    if (event.event === 'restoreCompleted') restorePurchases()
}
```

The callback is assigned outside the `isDespia` gate, the `despia()` call sits inside it. This gives Apple the distinct restore path it wants and covers subscription management in the same screen.

## Verify entitlement in your Supabase function

The restore reflects what the client claims, which is fine for showing and hiding UI and not enough to protect a paid action. For any request that costs money or serves premium content, check server-side at request time. Lovable apps run on Supabase / Lovable Cloud, so this slots into an edge function, one RevenueCat request, no webhook and no subscription table to keep in sync. It fits especially well since most Lovable apps already proxy an AI call through a function to keep the key off the client, and that function is exactly where the gate belongs. The full edge function pattern is here: [Lovable in-app purchases without webhooks](https://blog.despia.com/lovable-in-app-purchases-without-webhooks-revenuecat).

## Google Play, same button

Apple names this Guideline 3.1.1. Google Play uses no number, but its Play Billing docs are explicit that supporting restore is required for all developers, so the button belongs on Android as well. You write no Android-specific code: the same `restorePurchases()` covers both stores, and if both platform products map to one RevenueCat entitlement, each returns the same `entitlementId`.

## Place it where the reviewer looks

Put the button or the Customer Center entry on the paywall and the account screen. The reviewer checks near the purchase options and in settings. If they cannot find it, the rejection stands.

## The checklist

*   Real StoreKit billing in place (RevenueCat via Despia)
    
*   A distinct Restore button, or the Customer Center sheet
    
*   Reachable from the paywall or account screen
    
*   Entitlements re-checked after a restore completes
    

## Get it on the stores

Connect Despia to your Lovable app and ship to iOS and Android with native billing and restore from one codebase, no Xcode and no Capacitor plugin to maintain.

[See the setup docs at setup.despia.com](https://setup.despia.com)