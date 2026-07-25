---
title: "Lovable App Store Approval: The Rejection Checklist"
seoTitle: "Lovable App Store Approval: The Rejection Checklist"
seoDescription: "Every guideline that rejects a Lovable app, 2.1, 4.2, 4.8, 3.1.1, 3.1.2 and 5.1.1, with the exact fixes, so approval lands on the first submission."
datePublished: 2026-07-25T18:18:34.693Z
cuid: cms0p06r400000ahwdtleex5p
slug: lovable-app-store-approval-the-rejection-checklist
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/980bebf4-5902-48ea-b65a-03bea1f6db72.png
tags: android-app-development, ios, android, webapp, pwa, ios-app-development, appstore, lovable, appreview

---

App Review does not reject Lovable apps at random. It rejects them for a short, repeatable list of reasons, and almost every builder hits the same six. Nobody can promise Apple will approve your app, because your content and your metadata are yours. What can be made deterministic is the part that keeps failing: the structural rejections that come from shipping a Lovable app to a store it was not built for. Clear all six and there is nothing left on that list to reject you for.

This is the map of the whole surface, with the fix for each one.

## The six rejections, in one table

| Guideline | What triggers it | The fix |
| --- | --- | --- |
| 2.1 App Completeness | Login is broken in the build, so the reviewer never gets in | Serve the app from a real origin, not `file://` |
| 4.2 Minimum Functionality | The app is your website in a shell, with no native behaviour | Real native capability: push, biometrics, haptics, offline |
| 4.8 Login Services | Google sign-in with no privacy-preserving equivalent next to it | [Native Sign in with Apple](https://blog.despia.com/lovable-apple-login-native-sign-in) on the same Supabase session |
| 3.1.1 In-App Purchase | Stripe for digital goods, or IAP with no Restore Purchases button | StoreKit through RevenueCat, plus a visible restore control |
| 3.1.2 Subscriptions | Missing EULA link, or a paywall that does not disclose terms | Terms and privacy links on the paywall and account screen |
| 5.1.1 Data Collection | Forced registration before value, or no in-app account deletion | Anonymous accounts on first launch, deletion inside the app |

They look like six unrelated problems. They come from two root causes, and knowing which is which saves you from fixing the wrong layer.

## Root cause one: the delivery layer

Lovable's Capacitor export is a real native project you fully own, and if you have iOS and Android developers it is a legitimate route. The costs land fast for everyone else. You need a Mac, Xcode, provisioning profiles, and signing certificates. Billing is not included. Push requires a plugin. Over-the-air updates lost their default answer when Ionic discontinued Appflow in February 2025.

The quietest failure is the worst one. Capacitor serves your app offline from the `file://` protocol, and Supabase auth, OAuth redirects, and most SDKs expect an `http` origin. Sign in with Apple will not load on `file://` at all. That is a guideline 2.1 rejection waiting to happen, because a reviewer who cannot log in files the app as incomplete and never reaches the guidelines below.

Despia never touches `file://`. The default is remote hydration: the binary ships without embedded web assets and loads the current build from your URL on every launch, so cookies, Supabase auth, and every SDK behave exactly as they do on `yourapp.lovable.app`. For offline-first apps, the Local Server option caches the asset graph on-device and serves it from `http://localhost`, a real secure origin in both WebViews, so auth and SDKs keep working with no network at all. The [Lovable to iOS and Android guide](https://blog.despia.com/lovable-to-ios-and-android-app-ship-it-native) has the full comparison.

One caveat worth knowing before you pick Local Server: it serves over HTTP, and Apple's JS SDK requires HTTPS, so Apple sign-in cannot initialize there. If you need both, switch Offline Support from Native to PWA in the Despia Editor and cache assets with a service worker instead.

## Root cause two: the context boundary

Google blocks its consent screen inside an embedded WebView, the `disallowed_useragent` error, because an embedded view could read the user's password. Apple's guidelines agree that OAuth belongs in a browser the app cannot see into. So the plain web Google button either errors or gets rejected, and no amount of styling changes that.

The fix is to leave the WebView for the sign-in. Despia's `oauth://` bridge opens the system browser, ASWebAuthenticationSession on iOS and Chrome Custom Tabs on Android, the user authenticates, and Despia closes it and returns the tokens.

Then comes the part that catches people. The secure browser and the WebView are separate browsers with separate storage, by design. Anything crossing between them has to travel in the URL, which is why the flow needs a static `native-callback.html` in `public/` and a deeplink home:

```javascript
const { url } = await fetch('/api/auth/google-url', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ deeplink_scheme: 'myapp' }), // Despia > Publish > Deeplink
}).then(r => r.json())

despia(`oauth://?url=${encodeURIComponent(url)}`)
```

Two details are load-bearing. The `oauth/` prefix in the returning deeplink is what tells Despia to close the browser: `myapp://oauth/auth` works, `myapp://auth` strands the user in the browser. And `native-callback.html` has to be a real static file, not a React route, because the router strips the hash before you can read it. The [Google login guide](https://blog.despia.com/lovable-google-login-native-sign-in) has the whole flow.

Note what this root cause is not. Lovable Cloud is Supabase underneath, so your backend can mint sessions in code perfectly well. `setSession` and `signInWithIdToken` both work, and Supabase ships anonymous sign-in. You are not fighting a missing capability, only a browser boundary.

## 4.2: the app has to be an app

Apple rejects apps that offer nothing beyond the website. Native capability clears it, and with Despia each one is a JavaScript call from the Lovable code you already have:

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

if (isDespia) {
    despia('successhaptic://')
    despia(`setonesignalplayerid://?user_id=${accountId}`)

    // locked=true gates the read, not the write
    await despia(`setvault://?key=sessionToken&value=${token}&locked=true`)
}

// next launch: Face ID or Touch ID prompts before the value comes back
try {
    const { sessionToken } = await despia('readvault://?key=sessionToken', ['sessionToken'])
    resumeSession(sessionToken)
} catch {
    // key absent, first launch
}
```

The vault is worth understanding rather than pasting. It is encrypted key-value storage backed by iCloud Key Value Storage on iOS and App Backup on Android, so it survives uninstall and reinstall, and the biometric prompt fires when you read a locked key, never when you write it. A write with no matching read shows the reviewer nothing.

Push alone changes the review conversation, because an app that reaches users between sessions is doing something a bookmark cannot. Native transitions, safe areas, sheets, and haptics do the rest. The app has to read as an app when the reviewer opens it, not just pass a feature checklist.

## 4.8: hiding the Google button does not work

Guideline 4.8 says that if you use a third-party login to set up the user's primary account, you must offer an equivalent login that limits data collection to name and email, lets the user keep the email private, and does not collect app interactions for advertising without consent. Sign in with Apple is usually the simplest compliant choice, unless one of Apple's exceptions applies, which include proprietary account systems, enterprise and education accounts, and government identity systems.

Builders read the rejection as being about the Google button, so they hide it and resubmit, and get the identical rejection back. The rule is about account architecture, not UI. The reviewer tests the login, not the pixel you moved.

Apple is the easier of the two to wire, which surprises people. Apple's JS SDK with `usePopup: true` presents the native Apple sign-in sheet directly from the WebView on iOS, so iPhone needs no browser, no bridge, and no callback file:

```javascript
const nonce = crypto.randomUUID()

window.AppleID.auth.init({
  clientId: 'com.yourcompany.yourapp.webauth', // Services ID
  scope: 'name email',
  redirectURI: window.location.origin + '/',   // exact match, trailing slash included
  usePopup: true,                              // redirect mode blanks the WebView and fails review
  nonce,
})

const response = await window.AppleID.auth.signIn()
await supabase.auth.signInWithIdToken({
  provider: 'apple',
  token: response.authorization.id_token,
  nonce,
})
```

Android has no equivalent sheet, so it runs the Apple screen through the same `oauth://` bridge and ends on the same `signInWithIdToken` call. One credential, one exchange, no second flow to reason about. Login needs no `.p8` key, since this verifies Apple's `id_token` directly.

Two Lovable-specific footguns live here. On the current TanStack Start stack, `/auth` has to be a top-level public route at `src/routes/auth.tsx`, not under `_authenticated/`, or the route guard redirects the token-carrying URL away before sign-in runs. And Apple only returns the user's name on the very first authorization, so persist it from that first response or it is gone. The [Apple login guide](https://blog.despia.com/lovable-apple-login-native-sign-in) covers both stacks, current and legacy Vite.

## 3.1.1: Stripe is not an option, and restore is not optional

Two separate failures live under this number.

The first is payment routing. Apple requires In-App Purchases and Google requires Play Billing for digital content, and both reject apps that route around them. Capacitor includes no native billing, so a plain Lovable build has no compliant way to sell at all. Despia ships RevenueCat in every build, which puts real StoreKit and Play Billing behind the same web code, with a web checkout fallback keyed to the same user ID.

The second is the button almost nobody ships. If a purchase can be restored, Apple requires a distinct, tappable Restore Purchases control, and states directly that restoring automatically on launch does not satisfy it.

```javascript
async function restorePurchases() {
    const data = await despia('getpurchasehistory://', ['restoredData'])
    const active = (data.restoredData ?? []).filter(p => p.isActive)

    if (active.length === 0) return showMessage('No active purchases found.')
    if (active.some(p => p.entitlementId === 'premium')) unlockPremium()
}
```

If you would rather not build restore UI at all, `revenuecat://center?external_id=${userId}` opens RevenueCat's Customer Center, a native sheet where users restore, manage, or cancel. Either way, put it on the paywall and on the account screen. A restore control the reviewer cannot find is treated the same as one that does not exist. Google Play has no matching guideline number, but its billing docs require restore support too, and the same function covers both stores. The [restore purchases guide](https://blog.despia.com/lovable-restore-purchases-fix-guideline-3-1-1) has the server-side entitlement check that goes with it.

## 3.1.2: the fastest rejection to clear, unless you are on Capacitor

If you sell an auto-renewing subscription, the reviewer looks inside the app for the subscription title, its length, the price, a working privacy policy link, and a working Terms of Use link. In App Store Connect they check for a privacy policy and a EULA. A privacy policy alone does not satisfy 3.1.2, which is where most of the confusion comes from. Apple's standard EULA is accepted if you do not have your own.

AI-generated paywalls tend to render a price on a button and nothing else, which Apple treats as insufficient disclosure. State the renewal terms as readable text near the buy button.

The reason this one is worth taking seriously despite being trivial: on a plain Capacitor build, adding a one-line link to your paywall means a full native rebuild, a new binary, and another trip through the review queue. That is how a missing EULA link eats a second review cycle. On Despia the web layer updates over the air, so the link reaches the reviewer without a new build. Details in the [EULA guide](https://blog.despia.com/lovable-eula-rejection-fix-guideline-3-1-2).

## 5.1.1: do not make them sign up to see anything

Guideline 5.1.1 pushes back on apps that force registration before delivering value, and 5.1.1(v) requires in-app account deletion for any app that creates accounts. Both are common Lovable rejections.

The answer to the first is an anonymous account minted at first launch. A UUID in Despia's Storage Vault maps to an anonymous account row in Supabase, and the user is inside the app before anyone asks them for anything. The vault matters here because it survives uninstall and reinstall, and on iOS it is backed by iCloud Key Value Store, the same scope Apple uses for purchases.

Deletion has a trap worth naming. If you delete the row and mint a new account ID, RevenueCat entitlements keyed to the old ID orphan the purchase, and a paying user watches their subscription disappear. Anonymize instead: keep the `id` and `device_id`, clear every personal column, and access never breaks. A random device UUID with no identifying fields attached is not personal data, so this satisfies the rule.

One Lovable-specific warning. When an anonymous user later creates credentials, attach them to the existing row rather than creating a new user. Lovable-generated auth flows default to Supabase's standard signup, which mints a brand new user and silently strands the subscription on the old ID. Prompt for upgrade-in-place explicitly. The [loginless guide](https://blog.despia.com/lovable-loginless-mobile-app-anonymous-login-iap-push) covers the whole model including account switching.

## What purchases and push actually need

Worth stating plainly, because it is the thing people over-build. Purchases and push need one stable identifier, and whatever account ID you already have is a fine one. Anonymous row, Supabase user, does not matter. Pass the same value as the RevenueCat `external_id` and the OneSignal external user ID:

```javascript
if (isDespia) {
    despia(`revenuecat://launchPaywall?external_id=${encodeURIComponent(accountId)}&offering=default`)
    despia(`setonesignalplayerid://?user_id=${encodeURIComponent(accountId)}`)
}
```

One ID, three systems. Native OAuth is the thing that needs real work in a Lovable app. Monetising and notifying are not.

## The submission checklist

Structural work above, plus the metadata items that bounce apps under 2.1 before a reviewer opens them:

*   Login verified inside the actual build, not the browser preview
    
*   Native features present and reachable: push, haptics, biometrics, offline
    
*   Sign in with Apple next to Google, both landing on the same session
    
*   Guest access on first launch, no forced registration
    
*   In-app account deletion, with the anonymize path if purchases exist
    
*   Restore Purchases button or Customer Center on the paywall and account screen
    
*   Terms and privacy links on the paywall and the account screen
    
*   Subscription title, length, and price shown as text
    
*   EULA link in App Store Connect
    
*   A working demo account in App Review notes, including anything gated
    
*   Screenshots that show the app, not the marketing site
    
*   Privacy nutrition labels matching what the app actually collects
    

## What happens when they still ask for something

They sometimes will, and the useful question is how expensive the answer is. On a Lovable app running through Despia, almost every fix a reviewer asks for is a web-layer change: a link, a button, copy on the paywall, a screen the reviewer could not find. Those ship over the air the moment you redeploy in Lovable, so the reviewer sees the fix without a new binary and without rejoining the build queue. A resubmission is only needed when the native layer itself changes, like an icon or a new permission.

That is the honest version of a guarantee. The rejection list is finite and every item on it has a known fix. Clear them before you submit, and keep the ability to answer anything else in minutes instead of days.

## Get it on the stores

Take the Lovable app you already built and ship it to iOS and Android with the review surface handled: native auth, purchases, push, and anonymous accounts from one codebase. Code signing and submission run from the browser, no Mac and no CLI.

[See the setup docs at setup.despia.com](https://setup.despia.com)