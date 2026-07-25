---
title: "Convert Lovable to a Native App: Wrapper or Rebuild?"
seoTitle: "Convert Lovable to a Native App: Wrapper or Rebuild?"
seoDescription: "Convert Lovable to a native app. A bare URL wrapper gets rejected. A native runtime with 50+ device features does not."
datePublished: 2026-07-08T14:27:50.292Z
cuid: cmrc69z0400020akh53oxelnn
slug: convert-lovable-to-a-native-app-wrapper-or-rebuild
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/6da29845-bc8e-4a3a-9c4a-58a981d89ad9.png
tags: react-native, mobile-app, supabase, appstore, googleplay, web-to-app, lovable, web-to-native

---

You built a working app in Lovable, synced to GitHub, and you want it in the App Store and Google Play as a real native app, not a browser tab in a shell that review bounces. Two pitches show up. Wrap the URL and submit it. Or rebuild the whole thing in React Native because that is what real apps use. One gets rejected at review. The other is a native toolchain you did not sign up to run.

There is a third path the argument keeps missing: keep the Lovable app as the source of truth and run it through a native runtime that adds the native pieces the stores expect, with no rewrite. That is what Despia does, and it is why the choice is not really wrapper versus rebuild.

## Why a bare wrapper gets rejected

Sign-in through Supabase or a web redirect breaks inside the WebView, because Google blocks embedded-WebView OAuth on purpose. Digital goods sold through a web checkout get rejected, since the stores require native billing. And a bare webview shell with no native behaviour is the textbook minimum-functionality rejection, and Google Play enforces the same bar, so Android is not a free pass either. A wrapper is cheap because it skips every native integration review looks for.

## The React Native rebuild tax

The other pitch is to rebuild the app in React Native, and an AI can scaffold that in an afternoon. The scaffold is not the cost. The cost is everything it hands you to run afterward.

A React Native app is a native project. You now own an Xcode build and a Gradle build, CocoaPods and native dependency resolution, and CI that compiles both. Releases go through EAS or Fastlane, not a dashboard button. Crashes arrive as native stack traces you symbolicate through Sentry, not console logs. Every few months brings a React Native upgrade, a New Architecture migration, or a dependency that breaks the build. And over-the-air updates only cover the JavaScript layer, so any native change still means a full store resubmission.

None of that is exotic. It is the normal operating cost of a native app. The point is that the rebuild pitch never mentions it, and it is permanent. You are not shipping an app, you are adopting a native toolchain, and you are throwing away the Lovable codebase to do it.

## What the native runtime does, from your Lovable codebase

Despia runs your Lovable app and exposes native features through one function, gated to the runtime. No rebuild, no second codebase.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// Native haptic feedback.
if (isDespia) despia('successhaptic://')

// Link the device to the signed-in user on every authenticated load.
if (isDespia) despia(`setonesignalplayerid://?user_id=${userId}`)
```

Digital goods go through native billing, which the stores require. Launch the paywall, then gate features on the entitlement:

```javascript
// Digital goods: native paywall
despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)

// Gate premium on the active entitlement
const { restoredData = [] } = await despia('getpurchasehistory://', ['restoredData'])
if (restoredData.some(p => p.isActive && p.entitlementId === 'premium')) unlockPremium()
```

Physical goods and real-world services take Stripe, and the native PaymentSheet is a real sheet over your app, not a web checkout squeezed into the WebView:

```javascript
// Your backend creates the Payment Intent; the native sheet takes the card
window.stripeEvent = (e) => {
  if (e.method === 'paymentSheet' && e.status === 'completed') {
    fetch('/api/confirm-payment', { method: 'POST' }) // confirm via webhook before fulfilling
  }
}

const { publishable_key, client_secret } =
  await (await fetch('/api/create-payment-intent', { method: 'POST' })).json()
despia(`stripe://payment?publishable_key=${publishable_key}&payment_intent_client_secret=${client_secret}`)
```

The Storage Vault keeps tokens outside WebView storage, iCloud key-value backed on iOS and Android app-backup backed on Android. That lets you restore session, trial, or ban state across reinstalls when the user keeps the same Apple ID or Google account, and optionally lock reads behind Face ID or biometrics:

```javascript
// Persists across reinstall on the same Apple ID / Google account
await despia(`setvault://?key=refreshToken&value=${token}&locked=false`)
const { refreshToken } = await despia('readvault://?key=refreshToken', ['refreshToken'])

// locked=true requires Face ID / Touch ID to read the value back
await despia('setvault://?key=unlock&value=yes&locked=true')
const { unlock } = await despia('readvault://?key=unlock', ['unlock'])
```

All of it runs from your existing code, and Supabase stays your backend.

## Native Google sign-in: the one place the rebuild pitch has a point

Sign-in is the one spot where "just rebuild it native" has a real argument, because web sign-in through Supabase or a redirect breaks inside the WebView. So there is genuine native work here, no pretending it is one line. But it is a small piece, not a full rebuild. Build the Google URL, Supabase can hand you one, and open it through the native bridge:

```javascript
// Consent opens in the system browser, not the WebView
if (isDespia) despia(`oauth://?url=${encodeURIComponent(googleAuthUrl)}`)
```

Consent opens in the system browser, comes back through a deep link, and Supabase stays your backend. That is one focused piece against a whole native toolchain.

## Wrapper vs rebuild vs Despia

|  | Bare wrapper | React Native rebuild | Despia |
| --- | --- | --- | --- |
| Codebase | Lovable app in a shell, a dead end | A second native codebase to maintain | Lovable stays the source of truth |
| Build and release | Wrapper dashboard | Xcode, Gradle, CocoaPods, EAS or Fastlane, CI | Despia builds the binary |
| Updates | Varies | JS over the air, native changes need resubmission | Web ships over the air |

## When each is the right call

A bare wrapper is fine for a read-mostly content app with no login and no digital goods. A React Native rebuild is the right call when you genuinely need heavy custom native code, a hand-tuned rendering path, or a native SDK with no bridge. Most Lovable apps need neither. They need push, billing, sign-in, and a few device features on top of the app they already have, which is exactly what the runtime adds without the rewrite.

## FAQ

**Why was my Lovable app rejected from the App Store?** Usually one of three things: web sign-in through Supabase or a redirect that breaks inside the WebView, digital goods sold through a web checkout instead of native billing, or a bare webview shell that fails minimum functionality on both stores. Each has a native fix that runs from your existing code.

**Do I have to rebuild my Lovable app in React Native to ship it?** No. That is the false choice. A React Native rebuild means a new codebase and a native toolchain to run. Despia ships your existing Lovable app as a native binary and adds the native pieces through the same code.

**Why does my Supabase or Google login break in the app?** Google blocks OAuth in embedded WebViews, so a web redirect fails with a "browser may not be secure" error. Native sign-in through the `oauth://` bridge fixes it. Supabase stays your backend.

**Can I sell subscriptions?** Only through native billing. For digital goods, a web checkout gets rejected, so on Despia you launch the native paywall with `revenuecat://launchPaywall` and pass the user's `external_id`. Physical goods and real-world services can take Stripe, and the native PaymentSheet beats a web checkout inside the WebView.

**Do I keep my Lovable GitHub repo?** Yes. Your Lovable codebase stays the source of truth. Despia runs it as a native binary and updates the web layer over the air with no resubmission.

**Do updates still ship instantly?** Yes. Your web layer ships over the air. A React Native rebuild only gets that for its JavaScript, and any native change goes back through review.

## Get it on the stores from the app you built

Keep your Lovable codebase. Despia ships it to the App Store and Google Play with the native sign-in, billing, and push that pass review, from the one repo you already have. No rewrite, no native toolchain to run.

[Docs at setup.despia.com](https://setup.despia.com) or [start at despia.com](https://despia.com). If you get rejected, the [store rejection playbook](https://setup.despia.com/store-rejections/introduction) has the fix.