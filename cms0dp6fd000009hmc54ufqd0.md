---
title: "Base44 App Store Approval: The Rejection Checklist"
seoTitle: "Base44 App Store Approval: The Rejection Checklist"
seoDescription: "Every guideline that rejects a Base44 app, 4.2, 4.8, 3.1.1, 3.1.2 and 5.1.1, with the exact fix for each, so approval lands on the first submission."
datePublished: 2026-07-25T13:02:05.279Z
cuid: cms0dp6fd000009hmc54ufqd0
slug: base44-app-store-approval-the-rejection-checklist
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/da3f7699-a50e-47e4-a513-140a4aecd395.png
tags: android-app-development, ios, android, ios-app-development, wix, base44, appreview

---

App Review does not reject Base44 apps at random. It rejects them for a short, repeatable list of reasons, and almost every builder hits the same five. Nobody can promise Apple will approve your app, because your content and your metadata are yours. What can be made deterministic is the part that keeps failing: the structural rejections that come from shipping a Base44 app to a store it was not built for. Clear all five and there is nothing left on that list to reject you for.

This is the map of the whole surface, with the fix for each one.

## The five rejections, in one table

| Guideline | What triggers it | The fix |
| --- | --- | --- |
| 4.2 Minimum Functionality | The app is your website in a frame, with no native behaviour | Real native capability: push, biometrics, haptics, offline, native UI |
| 4.8 Login Services | Google sign-in with no privacy-preserving equivalent next to it | [Native Sign in with Apple](https://blog.despia.com/base44-apple-login-native-sign-in) on your own session system |
| 3.1.1 In-App Purchase | Stripe for digital goods, or IAP with no Restore Purchases button | StoreKit through RevenueCat, plus a visible restore control |
| 3.1.2 Subscriptions | Missing EULA link, or a paywall that does not disclose terms | Terms and privacy links on the paywall and account screen |
| 5.1.1 Data Collection | Forced registration before value, or no in-app account deletion | Guest accounts on first launch, deletion inside the app |

They look like five unrelated problems. Two of them share one root cause, and that is the part worth understanding before you start fixing anything.

## The root cause underneath the auth rejections

Base44's built-in auth cannot mint a session in code. There is no token-exchange endpoint, `User.create` returns 405 even from service-role functions, and sessions only fall out of Base44's own hosted browser flow. That single limitation is why the 4.8 loop feels unwinnable from inside the builder.

Native Sign in with Apple needs a session minted from a verified `id_token`, so 4.8 is blocked. Anonymous guest accounts need one minted from a device UUID, so the loginless answer to 5.1.1 is blocked with it. Both want the same capability and the platform does not offer it.

Be equally clear about what this does not block. Purchases and push only need a stable identifier to key against, and Base44's built-in user id is a perfectly good one. If you are staying on Base44's own login, pass that id as the RevenueCat `external_id` and as the OneSignal external user ID and you are done:

```javascript
const { id } = await base44.auth.me()

if (isDespia) {
    despia(`revenuecat://launchPaywall?external_id=${encodeURIComponent(id)}&offering=default`)
    despia(`setonesignalplayerid://?user_id=${encodeURIComponent(id)}`)
}
```

Custom auth is the price of native OAuth and guest accounts. It is not the price of monetising or notifying, and plenty of Base44 apps ship both without ever leaving the built-in login.

When you do need it, the shape is fixed: one `Account` entity with deny-all row-level security, one hand-rolled HS256 JWT, and every sign-in path, Apple, Google, password, and anonymous, minting the same token. Build it once and both rejections close together. The [Google login guide](https://blog.despia.com/base44-google-login-native-sign-in) walks that system from the data model up, including why Base44's built-in Google auth cannot be adapted to it.

## 4.2: the app has to be an app

Apple rejects apps that offer nothing beyond the website. [Base44's own mobile export](https://blog.despia.com/base44-to-ios-and-android-app-ship-it-native) is a wrapper around your published URL, and Base44's documentation states plainly that push notifications, in-app purchases, offline mode, biometrics, and haptics are not supported by it. Those absences are exactly what a reviewer scans for.

Native capability is what clears it, and in a Despia build each one is a JavaScript call from the Base44 code you already have:

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

if (isDespia) {
    despia('successhaptic://')
    despia(`setonesignalplayerid://?user_id=${accountId}`)
    await despia('setvault://?key=confirmAction&value=yes&locked=true')
}
```

Push alone changes the review conversation, because an app that reaches users between sessions is doing something a bookmark cannot. Native transitions, safe areas, sheets, and haptics do the rest of the work: the app has to read as an app when the reviewer opens it, not just pass a feature checklist. The [4.2 and 4.8 pillar](https://blog.despia.com/base44-app-store-rejection-guideline-4-2-and-4-8) covers what reviewers actually check.

## 4.8: hiding the Google button does not work

This is the rejection people burn the most submissions on. Guideline 4.8 says that if you use a third-party login to set up the user's primary account, you must offer an equivalent login that limits data collection to name and email, lets the user keep the email private, and does not collect app interactions for advertising without consent. In practice, Sign in with Apple is the only mainstream option that clears all three.

Builders read the rejection as being about the Google button, so they hide it or wire it elsewhere, resubmit, and get the identical rejection back. The rule is about account architecture, not UI. The OAuth client and the provider config are still wired in, and the reviewer tests the login rather than the pixel you moved.

The real fix is the [native Apple sign-in sheet](https://blog.despia.com/base44-apple-login-native-sign-in), not a web button styled to look like one. On iOS and web, Apple's JS SDK with `usePopup: true` presents the native sheet, Face ID or Touch ID, directly inside the WebView. Android has no equivalent, so it runs the same login through Despia's `oauth://` bridge in Chrome Custom Tabs and returns through a deeplink.

```javascript
window.AppleID.auth.init({
  clientId: appConfig.appleServicesId,
  scope: 'name email',
  redirectURI: window.location.origin + '/',
  usePopup: true, // redirect mode blanks the screen in a WebView and gets rejected
})
```

That last line is load-bearing rather than a preference. The full setup, Services ID, Return URLs, backend token verification, and why login needs no `.p8` key, is in the [Base44 Apple login guide](https://blog.despia.com/base44-apple-login-native-sign-in).

One exemption worth knowing: if your app uses only your own email and password accounts, with no social login anywhere, Sign in with Apple is not required. Keep Google and it is.

## 3.1.1: Stripe is not an option, and restore is not optional

Two separate failures live under this number.

The first is payment routing. Base44's documentation is explicit that Stripe for digital content inside a mobile app gets rejected, because Apple and Google require their own billing. Base44 has no native billing yet, so a Base44 app on its own cannot process in-app purchases at all. Despia includes RevenueCat in every build, which puts real StoreKit and Play Billing behind the same web code, with a web checkout fallback keyed to the same user ID.

The second is the button almost nobody ships. If a purchase can be restored, Apple requires a distinct, tappable Restore Purchases control, and states directly that restoring automatically on launch does not satisfy it.

```javascript
async function restorePurchases() {
    const data = await despia('getpurchasehistory://', ['restoredData'])
    const active = (data.restoredData ?? []).filter(p => p.isActive)

    if (active.length === 0) return showMessage('No active purchases found.')
    if (active.some(p => p.entitlementId === 'premium')) unlockPremium()
}
```

Put it on the paywall and on the account screen. A restore control the reviewer cannot find is treated the same as one that does not exist. Google Play has no matching guideline number, but its billing documentation requires restore support too, and the same function covers both stores. The [restore purchases guide](https://blog.despia.com/base44-restore-purchases-fix-guideline-3-1-1) has the server-side entitlement check that goes with it.

## 3.1.2: the fastest rejection to clear

If you sell an auto-renewing subscription, the reviewer looks inside the app for the subscription title, its length, the price, a working privacy policy link, and a working Terms of Use link. In App Store Connect they check for a privacy policy and a EULA. The EULA is the one everyone forgets, and Apple's standard one is accepted if you do not have your own.

AI-generated paywalls tend to render a price on a button and nothing else, which Apple treats as insufficient disclosure. State the renewal terms as readable text near the buy button, and put the terms and privacy links on both the paywall and the account screen. Details and the exact wording are in the [EULA guide](https://blog.despia.com/base44-eula-rejection-fix-guideline-3-1-2).

## 5.1.1: do not make them sign up to see anything

Guideline 5.1.1 pushes back on apps that force registration before delivering value, and 5.1.1(v) requires in-app account deletion for any app that creates accounts. Both are common Base44 rejections. Deletion you can ship on either auth stack. The no-signup part is the one that needs your own, because an anonymous account minted at first launch is a session Base44 cannot issue in code.

A UUID in Despia's Storage Vault maps to an anonymous `Account` record in Base44 and mints your JWT. The user is inside the app before anyone asks them for anything, and that same account ID keys RevenueCat and push. The vault matters here because it survives uninstall and reinstall, and on iOS it is backed by iCloud Key Value Store, which is the same scope Apple uses for purchases.

Deletion has a trap worth naming. If you delete the record and mint a new account ID, RevenueCat entitlements keyed to the old ID orphan the purchase, and a paying user watches their subscription disappear. Anonymize the record instead: keep the `id` and `device_id`, clear everything personal, and access never breaks. The [loginless guide](https://blog.despia.com/base44-loginless-mobile-app-anonymous-login) covers the whole model, including account linking and switching.

## Start from something that already passes

None of this is worth building from scratch. The starter at [github.com/despia-native/base44-native-boilerplate](https://github.com/despia-native/base44-native-boilerplate) is a React, Vite, and Tailwind project that runs on Base44 and ships native through Despia, with the rejection surface already covered: native Sign in with Apple next to Google on a custom JWT system, automatic guest accounts, in-app account deletion with a two-step confirmation, deny-all RLS on every entity, push, RevenueCat, and a native-first iOS UI.

Clone it, run `base44 dev`, and pushed changes sync back into the Base44 Builder. One gap to close before you open sign-up to the public: the auth endpoints ship without per-IP or per-email rate limiting, so put an attempt counter with a cooldown in front of login, register, and password reset.

## The submission checklist

Structural work above, plus the metadata items that get apps bounced under 2.1 before a reviewer ever opens them:

*   Native features present and reachable: push, haptics, biometrics, offline
    
*   Sign in with Apple next to Google, both minting the same session
    
*   Guest access on first launch, no forced registration
    
*   In-app account deletion, with the anonymize path if purchases exist
    
*   Restore Purchases button on the paywall and the account screen
    
*   Terms and privacy links on the paywall and the account screen
    
*   Subscription title, length, and price shown as text
    
*   EULA link in App Store Connect
    
*   A working demo account in App Review notes, including anything gated
    
*   Screenshots that show the app, not the marketing site
    
*   Privacy nutrition labels matching what the app actually collects
    

## What happens when they still ask for something

They sometimes will, and the useful question is how expensive the answer is. On a Base44 app running through Despia, almost every fix a reviewer asks for is a web-layer change: a link, a button, copy on the paywall, a screen the reviewer could not find. Those ship over the air the moment you publish in Base44, so the reviewer sees the fix without a new binary and without rejoining the build queue. A resubmission is only needed when the native layer itself changes, like an icon or a new permission.

That is the honest version of a guarantee. The rejection list is finite and every item on it has a known fix. Clear them before you submit, and keep the ability to answer anything else in minutes instead of days.

## Get it on the stores

Take the Base44 app you already built and ship it to iOS and Android with the review surface handled: native auth, purchases, push, and guest accounts from one codebase. Code signing and submission run from the browser, no Mac and no CLI.

[See the setup docs at setup.despia.com](https://setup.despia.com)