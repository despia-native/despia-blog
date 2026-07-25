---
title: "Publish Base44 to AppStore: Despia vs Base44 Mobile Submit"
seoTitle: "Publish Base44 to AppStore: Despia vs Base44 Mobile Submit"
seoDescription: "Base44 Mobile Submit ships your app as a webview. Despia adds push, in-app purchases, offline, and 50+ native features from the same Base44 codebase."
datePublished: 2026-06-28T10:40:40.754Z
cuid: cmqxnrbqc00000akr568r6d6j
slug: publish-base44-to-appstore-despia-vs-base44-mobile-submit
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/c984c13c-3350-486d-9b24-068f3e47adfc.png
tags: webview, base, appstore

---

Base44 ships its own Mobile Submit flow now. You scan your app inside the editor, let the AI fix store issues, generate an IPA and AAB, and submit. So if it is built in and free to start, why reach for Despia at all? The answer is what happens after submission, when a user expects the app to push them a notification, take a subscription, or work on the subway.

## What Base44 Mobile Submit actually does

Mobile Submit is a convenience layer, and a good one for what it targets. It wraps your published Base44 app in a native shell that opens your app's URL, scans the result against the current Apple and Google guidelines, and uses the Base44 AI chat to walk you through fixes until your readiness score is high enough to submit. It generates the store bundles for you. Content and design changes you publish in Base44 show up in the app without a resubmission, because the shell is loading your live URL.

If your app is a content experience, a directory, a dashboard, or a form-driven tool, that is often enough to get on the stores. You never leave the editor, and you do not touch Xcode.

The ceiling is in Base44's own docs. The mobile app runs your web app inside a secure web view, and native-only features such as push notifications and full offline mode are not supported. HealthKit is not supported. There is no StoreKit or Google Play Billing integration yet, so you cannot sell digital goods, and routing them through Stripe gets the app rejected. Permissions are set by an AI scan and are not editable. The start URL is chosen for you. The bundle ID is fixed at `com.base[app-id].app` and cannot be changed, which causes signing-key mismatches if you ever uploaded the same brand from another tool.

None of that is a knock on Base44 as a builder. It is the difference between a webview that opens your URL and a native runtime that gives that URL real device access.

## Where Despia changes the shape of the app

Despia also runs your published Base44 app in the platform WebView, WKWebView on iOS and the Chromium-based WebView on Android. The difference is that it exposes 50+ native capabilities to that web layer through a single JavaScript function, ships web updates over the air via remote hydration, and exports a full Xcode and Android Studio project at any time if you ever outgrow it.

Critically, it is still one codebase. Your Base44 app stays the source of truth. You are not maintaining a second native project, you are adding native calls to the web app you already published. Base44 is a directly supported builder, and there is an MCP server at `https://setup.despia.com/mcp` so the Base44 AI can write correct Despia code without you pasting docs into chat.

The native features Despia gates behind a one-line user agent check are exactly the ones Mobile Submit lists as unsupported. Push, for example:

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// link the device to your Base44 user on every authenticated load
if (isDespia) {
    despia(`setonesignalplayerid://?user_id=${userId}`)
}
```

Then your backend sends to that user through OneSignal, even when the app is closed. The same pattern unlocks in-app purchases, which Base44 cannot do today:

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

if (isDespia) {
    // App Store / Play Billing via RevenueCat, store-compliant
    despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)
} else {
    // web users fall back to a RevenueCat web purchase link
    window.location.href = `https://pay.rev.cat/<your_token>/${encodeURIComponent(userId)}`
}
```

For physical goods, Despia gives you the native Stripe PaymentSheet instead of a webview checkout that risks rejection. Add Face ID and Touch ID through the Identity Vault, background GPS, haptics, NFC, HealthKit, and offline support via the local server, all from the same Base44 codebase.

## Side by side

| Capability | Base44 Mobile Submit | Despia |
| --- | --- | --- |
| Runs your published Base44 app | Yes, in a webview | Yes, in a native runtime |
| One codebase, no second project | Yes | Yes |
| Push notifications | Not supported | OneSignal, external user ID targeting |
| Digital goods (subscriptions, unlocks) | No StoreKit or Play Billing | RevenueCat, App Store and Play Billing |
| Physical goods payments | Stripe in webview risks rejection | Native Stripe PaymentSheet |
| Offline mode | Not supported | Local server, serves from localhost |
| Biometrics | Not available | Face ID, Touch ID, fingerprint |
| Other device features | Webview only | 50+ including GPS, haptics, NFC, HealthKit, widgets |
| Over-the-air web updates | Yes | Yes, via remote hydration |
| Bundle ID control | Fixed, not changeable | Configurable |
| Native export escape hatch | No | Full Xcode and Android Studio projects |
| AI builder integration | Built in | Base44 supported, plus MCP server |

## The rejection most webview apps hit

There is a second reason this matters, and it is about getting approved at all. Apple and Google reject apps that read as a repackaged website under their minimum functionality rules. A pure webview shell with no native behaviour is the textbook trigger. The standard fix is to add real native features, push, biometrics, offline, haptics, so the app does more than a browser tab. That is precisely the set Despia provides and Mobile Submit does not, which means Despia is not only adding capability, it is reducing the odds of a minimum-functionality rejection.

## When Base44 Mobile Submit is the right call

Use Mobile Submit when the app genuinely is a content or web experience and you want zero extra surface area. If you do not need push, do not sell anything in-app, do not need offline, and the app is read-mostly, the built-in flow gets you on the stores without learning anything new. There is real value in never leaving the editor, and for a v1 you want in front of users this week, that can be the correct trade. You can always move to Despia later when a feature demands native access, since the web app does not change.

The line is simple. If your app needs to behave like an app, notify, charge, authenticate with a fingerprint, work offline, Mobile Submit's webview cannot reach those, and Despia's runtime can, from the same Base44 codebase.

## Ship one app, not two

If you are going to keep building in Base44, keep building there. Despia ships that app to the App Store and Google Play, gives it native device access, and updates it over the air, from the one codebase you already have.

[Learn more about Despia in the docs](https://setup.despia.com)