---
title: "Base44 to App: Despia vs Median.co"
seoTitle: "Base44 to App: Despia vs Median.co"
seoDescription: "Despia vs Median: both ship a Base44 app to iOS and Android from your web code. The real split is ownership, pricing model, and AI-first tooling."
datePublished: 2026-07-25T08:59:21.772Z
cuid: cms05115k00000akqgivrgbdu
slug: base44-to-app-despia-vs-median-co
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/8740f420-f702-41a9-9dd6-bcaaed7bb769.png
tags: ios, android, webview, webapp, pwa, ios-app-development, base44

---

You built on Base44, it works, and now you want it in the App Store and Google Play. Two of the names you will hit are Despia and Median, and on the surface they do the same job: take the web app you already have and ship it as a native iOS and Android build. They are more alike than most comparison pages admit. The decision is not which one renders your app better, it is who owns the result and how you pay for it.

## What Median actually is

Median.co, formerly GoNative, has been building webview apps since 2014, which makes it one of the oldest and most proven platforms in this category. You configure your app in a browser-based App Studio, add capability through a Native Plugin Library, and wire native features into your web code with the Median JavaScript Bridge and an optional NPM package. The plugin set is real: push notifications through OneSignal and others, Face ID and Touch ID, social login, QR and barcode scanning, in-app purchases, and native navigation menus. Web content changes appear in the app immediately, the same over-the-air behaviour you would expect. Median also runs a fully managed service where their team builds, publishes, and keeps your app updated ahead of each OS release.

None of that is a thin wrapper, and it would be dishonest to call it one. Median is trusted by enterprise IT teams and product owners who want a mature platform and a support organisation behind it, and it explicitly supports apps built on Base44, Lovable, and similar tools. If you are comparing it to a bare URL-in-a-shell service, Median is far above that line.

## Where Despia and Median are the same

It is worth being clear about what does not separate these two, because a lot of comparison content pretends otherwise. Both run your web app in the platform webview, WKWebView on iOS and the Chromium-based WebView on Android. Both expose native device features through a JavaScript bridge. Both update web content over the air with no store resubmission. Both keep your web app as the single source of truth, so neither one saddles you with a second native codebase the way a rebuild-to-React-Native tool does. On rendering and architecture, they are the same class of product. The differences that matter are structural and commercial.

## The real difference: one-time and self-hosted, or subscription and managed

This is the split that should drive the decision.

Despia is a one-time per-app license, around $250, and it is self-hosted with no vendor lock-in. Despia never holds your hosting, the JavaScript SDK is MIT licensed, and its 50-plus native APIs are part of the runtime, not an upsell per feature. You can also export a complete Xcode and Android Studio project and keep working native source.

Median is a recurring subscription, and its plugins are tiered. Popular integrations sit in an Essential tier, more advanced ones in a Plus tier, and the fully managed, enterprise service is priced well into the thousands. Median is not strictly locked down either: it offers source access on its higher Professional licenses, so ownership is not purely either-or. But its default flow keeps your app pointed at a hosted URL and managed on the platform. Reviewers on G2 repeatedly flag the pricing as opaque and note being upcharged for plugins they thought were included, with features like biometrics costing more on top. That is the trade for the managed, enterprise-grade side of the product, and whether it is worth it depends on who you are.

So the honest framing is one-time and self-hosted versus subscription and managed. With Despia you pay once and self-host. With Median you subscribe to a managed platform and pay as you add features, with source available on its higher tiers.

## The AI-first difference, which matters most for Base44

Both platforms have a bridge, but they are built for different hands. Median's bridge is built for a developer configuring an app in App Studio. Despia's is built for an AI agent writing code.

Despia's native calls are flat command strings, and Despia publishes an MCP server you paste into Base44 under Settings, Account, MCP connections:

```text
https://setup.despia.com/mcp
```

Once connected, the Base44 agent writes the native calls itself, because a flat string like the one below is something an AI generates reliably, with no nested config objects or provider wrappers to get wrong.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// Base44's agent can emit this directly once the MCP is connected
if (isDespia) despia(`setonesignalplayerid://?user_id=${userId}`)
```

For a Base44 builder who is prompting rather than hand-coding, that is the difference between the AI shipping the native feature for you and you translating a plugin's setup into code by hand.

## A running start for Base44

If you would rather not begin from an empty project, Despia publishes an open [Base44 native boilerplate](https://github.com/despia-native/base44-native-boilerplate): a React, Vite, and Tailwind starter that already wires native Google Sign-In through the `oauth://` bridge, your own JWT sessions on a Base44 `Account` entity, and push notification setup, all on Base44's own backend. Making it yours is a three-spot checklist, config, secrets, and external accounts. That is a different kind of starting point than a managed App Studio: you clone real source, own it, and keep editing it in Base44.

## Side by side

|  | Median.co | Despia |
| --- | --- | --- |
| Architecture | Webview plus JS bridge | Webview plus JS bridge |
| Web updates over the air | Yes | Yes |
| Pricing model | Recurring subscription, plugins tiered | One-time per-app license, around $250 |
| Premium features | Essential and Plus plugin tiers, some upcharged | 50-plus capabilities included in the runtime |
| Hosting | Platform-managed | Self-hosted, Despia never holds hosting |
| Native source access | On the Professional license | Included, with full Xcode and Android Studio export, MIT SDK |
| Managed build and publish | Yes, full-service tier | Self-serve from the browser |
| AI-agent tooling | Configure in App Studio | MCP server so the Base44 agent writes the calls |
| Starter template | App Studio configuration | Open Base44 native boilerplate you clone and own |
| Track record | Since 2014 | Runtime since 2011, Despia since 2023 |

## When Median is the right call

If you want a vendor to own the entire app lifecycle, Median is a genuinely strong choice and Despia does not try to replicate it. Median's managed service builds your app, handles Apple and Google submission with guaranteed acceptance, and proactively updates ahead of OS releases so it never gets delisted. For a non-technical organisation with budget that wants zero involvement in the build and publish process, that white-glove model is worth paying for, and Median has run it at enterprise scale since 2014 with the support and compliance posture to match. Buying an outcome instead of a tool is a legitimate decision, and that is what Median sells at its higher tiers.

Despia is the opposite bet: you keep the code, you keep the hosting, you pay once, and you point your AI at an MCP so the app writes itself. For a Base44 builder shipping their own product, that is usually the better fit. For an enterprise that wants a managed vendor relationship, Median may be.

## Ship one app, not two

Both platforms wrap your Base44 app well, so this is not a rendering contest. Median is a managed platform you subscribe to, strongest when you want a vendor to run the whole lifecycle. Despia is a one-time, self-hosted license you can point your AI agent at. Pick based on whether you want to own the setup or hand off the lifecycle.

[Learn more about Despia in the docs](https://setup.despia.com) or [start building at despia.com](https://despia.com).