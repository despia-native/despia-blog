---
title: "Convert a Web App to a Native App: Wrapper or Rebuild?"
seoTitle: "Convert a Web App to a Native App: Wrapper or Rebuild?"
seoDescription: "Convert a web app to a native app. A bare URL wrapper gets rejected. A native runtime with 50+ device features does not."
datePublished: 2026-07-08T14:59:31.679Z
cuid: cmrc7eq4n00000aja0icw68r5
slug: convert-a-web-app-to-a-native-app-wrapper-or-rebuild
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/e196d9c3-0c36-4908-84da-ab9015636459.png
tags: react-native, webview, webapp, appstore, googleplay

---

You built a working web app and you want it in the App Store and Google Play as a real native app, not a browser tab in a shell that review bounces. Two pitches show up. Wrap the URL and submit it. Or rebuild the whole thing in React Native because that is what real apps use. One gets rejected at review. The other is a native toolchain you did not sign up to run. This holds whether your app is React, Vue, Svelte, or plain JavaScript.

There is a third path the argument keeps missing: keep the web app as the source of truth and run it through a native runtime that adds the native pieces the stores expect, with no rewrite. That is what Despia does, and it is why the choice is not really wrapper versus rebuild.

## Why a bare wrapper gets rejected

Sign-in through a web redirect breaks inside the WebView, because Google blocks embedded-WebView OAuth on purpose. Digital goods sold through a web checkout get rejected, since the stores require native billing. And a bare webview shell with no native behaviour is the textbook minimum-functionality rejection, and Google Play enforces the same bar, so Android is not a free pass either. A wrapper is cheap because it skips every native integration review looks for.

## The React Native rebuild tax

The other pitch is to rebuild the app in React Native, and an AI can scaffold that in an afternoon. The scaffold is not the cost. The cost is everything it hands you to run afterward.

A React Native app is a native project. You now own an Xcode build and a Gradle build, CocoaPods and native dependency resolution, and CI that compiles both. Releases go through EAS or Fastlane, not a dashboard button. Crashes arrive as native stack traces you symbolicate through Sentry, not console logs. Every few months brings a React Native upgrade, a New Architecture migration, or a dependency that breaks the build. And over-the-air updates only cover the JavaScript layer, so any native change still means a full store resubmission.

None of that is exotic. It is the normal operating cost of a native app. The point is that the rebuild pitch never mentions it, and it is permanent. You are not shipping an app, you are adopting a native toolchain, and you are throwing away the web codebase to do it.

## What the native runtime does, from your existing codebase

Despia runs your web app and exposes native features through one function, gated to the runtime. No rebuild, no second codebase.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// Native haptic feedback.
if (isDespia) despia('successhaptic://')

// Link the device to the signed-in user on every authenticated load.
if (isDespia) despia(`setonesignalplayerid://?user_id=${userId}`)
```

Native sign-in opens a real secure browser session instead of the WebView. Build the Google URL on your backend and hand it to the bridge:

```javascript
// Consent opens in the system browser, not the WebView
if (isDespia) despia(`oauth://?url=${encodeURIComponent(googleAuthUrl)}`)
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

All of it runs from your web code.

## Wrapper vs rebuild vs Despia

|  | Bare wrapper | React Native rebuild | Despia |
| --- | --- | --- | --- |
| Codebase | Web app in a shell, a dead end | A second native codebase to maintain | Web app stays the source of truth |
| Build and release | Wrapper dashboard | Xcode, Gradle, CocoaPods, EAS or Fastlane, CI | Despia builds the binary |
| Updates | Varies | JS over the air, native changes need resubmission | Web ships over the air |

## When each is the right call

A bare wrapper is fine for an internal tool, a prototype, or an app with no login and no digital goods. A React Native rebuild is the right call when you genuinely need heavy custom native code, a hand-tuned rendering path, or a native SDK with no bridge. Most web apps need neither. They need push, billing, sign-in, and a few device features on top of the app they already have, which is exactly what the runtime adds without the rewrite.

## FAQ

**Why was my web app rejected from the App Store?** Usually one of three things: web sign-in that breaks inside the WebView, digital goods sold through a web checkout instead of native billing, or a bare webview shell that fails minimum functionality on both stores. Each has a native fix that runs from your existing code.

**Do I have to rebuild my web app in React Native to ship it?** No. That is the false choice. A React Native rebuild means a new codebase and a native toolchain to run. Despia ships your existing web app as a native binary and adds the native pieces through the same code.

**Why does my Google login fail in the app?** Google blocks OAuth in embedded WebViews, so a web redirect fails with a "browser may not be secure" error. Native sign-in through the `oauth://` bridge fixes it, and your backend stays as is.

**Can I sell subscriptions through Stripe?** Not for digital goods, where a web checkout gets rejected, so on Despia you launch a native paywall with `revenuecat://launchPaywall`. Physical goods and real-world services can take Stripe, and the native PaymentSheet beats a web checkout inside the WebView.

**Do I need in-app account deletion?** Yes, if users can create an account. Apple requires it and Google Play expects the same, and it is a common rejection. A wrapper cannot add it; your own backend can, with a delete endpoint and a settings screen.

**Do updates still ship instantly?** Yes. Your web layer ships over the air with no resubmission. A React Native rebuild only gets that for its JavaScript, and any native change goes back through review.

## Ship one app, not two

Keep building in your web stack. Despia ships that app to the App Store and Google Play with the native integrations that pass review, updated over the air, from the one codebase you already have. No rewrite, no native toolchain to run.

[Docs at setup.despia.com](https://setup.despia.com) or [start at despia.com](https://despia.com). If you get rejected, the [store rejection playbook](https://setup.despia.com/store-rejections/introduction) has the fix.