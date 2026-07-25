---
title: "WebView Apps: What They Are and How to Ship One"
seoTitle: "WebView Apps: What They Are and How to Ship One"
seoDescription: "A WebView app runs your web code in a native shell. Here is what that means, whether Apple allows it, and how to ship one that passes review in 2026."
datePublished: 2026-07-25T11:11:24.106Z
cuid: cms09qu2700000aktcrue30i4
slug: webview-apps-what-they-are-and-how-to-ship-one
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/d7ccc559-9a79-4d65-bf1b-bfd98bbab03c.png
tags: webview, webapp, pwa

---

A WebView app is a native mobile app that runs your existing web app inside the system browser engine, so your HTML, CSS, and JavaScript ship to the App Store and Google Play as a real installable binary. It is the architecture behind a large share of the apps people use every day, and it is also the one builders are most anxious about, because the first thing anyone asks is whether Apple will reject it. The honest answer sets up everything else on this page: a bare URL wrapper usually is rejected, and a WebView app with real native behavior usually is not. Roughly one in ten apps on the App Store and Google Play already ship a WebView container, per Appfigures, which scans stores for the SDKs apps use. It is a mainstream, allowed architecture, and the difference between the version that passes and the version that fails is specific and learnable.

## What a WebView app actually is

A WebView is a browser engine embedded inside a native app container. On iOS that engine is WKWebView, and on Android it is the Chromium-based System WebView. A WebView app loads your web content into that engine and wraps it in a native binary that the stores accept and users install like any other app.

It is not a browser tab, because it ships as its own app with its own icon, native container, and access to device features. It is not a progressive web app either. A PWA is still web distribution: it cannot be listed in the App Store, and on iOS it only receives push if the user adds it to the home screen by hand. A WebView app is native distribution with a web-rendered interface, which is a different thing from both.

The practical appeal is that you keep one codebase. The web app you already built stays the source of truth, and the native shell gives it store distribution and device capability without a second project written in Swift or Kotlin.

## WebView app vs native app vs PWA

|  | WebView app | Native app | PWA |
| --- | --- | --- | --- |
| Codebase | Your existing web app | Separate Swift and Kotlin | Your existing web app |
| Store distribution | App Store and Google Play | App Store and Google Play | Not on the App Store |
| Device features | Through a native bridge | Full, direct | Limited, more on Android than iOS |
| Push on iOS | Yes, native APNs | Yes | Only for a home-screen PWA |
| Updates | Web layer over the air | Store review each release | Instant |
| Build cost | Low, one codebase | High, two codebases | Low |

## Will Apple reject a WebView app?

This is the question behind most WebView searches, so here is the direct version. Apple's Guideline 4.2, minimum functionality, is the rule that matters. It rejects apps that offer nothing beyond a repackaged website. If your app loads a URL and does nothing else, it is very likely to be rejected, and no tool can promise otherwise for a bare wrapper.

The fix is not a trick, it is giving the app genuine native behavior so it is an app, not a bookmark. In practice that means things a website cannot do: native navigation, push notifications, biometric login, offline handling, haptics, native sharing. Apple explicitly allows apps that use the system WebView framework properly and deliver an app-like experience beyond a mirrored website. Reviewers are looking for utility that a browser tab does not provide, and a handful of real native features clears that bar for most apps.

One point of confusion worth clearing up: people read that "WebView is deprecated" and panic. What Apple deprecated is the old UIWebView, years ago, and it no longer accepts apps that use it. WKWebView, the modern engine every current WebView app uses, is fully supported and is the sanctioned way to render web content in an app. A WebView app built on WKWebView is not using anything deprecated.

## Can a WebView app do push, payments, and offline?

These are the follow-up questions, and the answers decide whether the architecture fits your product.

Push notifications: yes, real ones. A WebView app receives native push over APNs on iOS and FCM on Android, which reaches the lock screen even when the app is closed. This is different from web push and is one of the native features that helps clear Guideline 4.2.

Payments: it depends on what you sell. If you sell digital goods or subscriptions used inside the app, Apple's Guideline 3.1.1 generally requires its In-App Purchase system. If you sell physical goods or services, you use your existing processor like Stripe, and Apple's IAP does not apply. The rules around external purchase links have been shifting since 2024, so confirm the current guideline for your category before you build around it.

Offline: yes, with a real native runtime that can cache content and assets, rather than showing the browser's dinosaur when the connection drops. Offline handling is itself one of the behaviors reviewers look for.

## Do WebView apps update without App Store review?

Mostly, yes, and this is one of the architecture's biggest advantages. Because your interface is web content, changes to it can load without shipping a new binary, the same way a website updates. This does not violate Apple's Guideline 2.5.2, which forbids downloading and executing native code, because you are updating web content, not installing new native executables. You still submit a new build when you add a genuinely new native capability, which for most apps is once or twice a year rather than every release.

## How to build a WebView app

Two paths. You can hand-roll it with WKWebView and an Android WebView, or use Capacitor, and take on the native project yourself: a Mac, Xcode, signing certificates, and a second codebase to maintain, with push and billing as things you wire up by hand. For a team with native developers that wants that control, it is a legitimate route.

The other path is a runtime that gives you the native layer through your existing web code. Despia takes the web app you already built, ships it as a real iOS and Android binary, and exposes 50-plus native features through a single JavaScript function, so the native behavior that clears Guideline 4.2 is a call rather than a native rewrite.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// Native behavior that turns a wrapped site into an app, from your web code
if (isDespia) {
  despia('successhaptic://')                       // haptics on key actions
  despia(`setonesignalplayerid://?user_id=${userId}`) // register for native push
}
```

Web changes ship over the air, the build and store submission run from the browser without a Mac, and you can export a full Xcode and Android Studio project at any time, so the escape hatch is real. That combination, one codebase plus native capability, is what separates a WebView app that passes review from a wrapper that does not.

## WebView app FAQ

**Are WebView apps allowed on the App Store?** Yes. Apple allows WebView apps that use WKWebView and provide an app-like experience with native functionality. A bare website wrapper with no native features is what gets rejected under Guideline 4.2.

**Why do WebView apps get rejected?** Almost always Guideline 4.2, minimum functionality: the app offers nothing beyond the mobile website. Adding native navigation, push, offline support, and similar features resolves it.

**Is WebView deprecated?** UIWebView is deprecated and no longer accepted. WKWebView, which modern WebView apps use, is current and fully supported.

**What is the difference between a WebView app and a native app?** A native app is written in Swift or Kotlin. A WebView app runs your web code inside a native shell. With a solid native layer, users generally cannot tell the difference.

**Can a WebView app send push notifications?** Yes, native push over APNs and FCM, delivered even when the app is closed.

**Is a WebView app the same as a PWA?** No. A PWA is web distribution and cannot be listed in the App Store. A WebView app is a native binary distributed through the stores.

## Ship your web app as a real app

If you built your product on the web, keep building there. Despia ships that app to the App Store and Google Play, gives it the native features that pass review, and updates it over the air, from the single codebase you already have.

[Learn more in the docs](https://setup.despia.com) or [start building at despia.com](https://despia.com).