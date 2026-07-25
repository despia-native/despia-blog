---
title: "Lovable to App: Despia vs Median.co"
seoTitle: "Lovable to App: Despia vs Median.co"
seoDescription: "v"
datePublished: 2026-07-25T09:01:35.161Z
cuid: cms053w2k00000akxh4mmg56j
slug: lovable-to-app-despia-vs-median-co
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/4c7fb7f9-ac4e-40bc-9c77-7ca558d8dfa8.png
tags: android-app-development, ios, android, webview, webapp, pwa, ios-app-development, lovable

---

You built on Lovable with Supabase behind it, and now you want the App Store and Google Play. Lovable users have three ways to get there, not two: Lovable's own Capacitor export, Median, and Despia. The Capacitor route hands you a native project to maintain yourself. The other two keep Lovable as the live product and wrap it in a native shell, which is why the real comparison is Despia against Median. They are more alike than most pages admit, and the decision comes down to who owns the result and how you pay.

## Your three options, quickly

Lovable's Capacitor export is a real native project, and if you have a Mac, Xcode, and the appetite to maintain it, it is a legitimate path. The costs show up fast for everyone else. It ships without a billing or push pipeline, and Capacitor serves your app offline from the `file://` protocol, which breaks Supabase auth, OAuth redirects, and Sign in with Apple, because those expect an http origin. So going native that way tends to break the auth you already built.

Median and Despia both avoid that, because both load your live Lovable URL rather than bundling assets behind `file://`. Your Supabase auth, cookies, and redirects keep working exactly as they do on the web. That shared architecture is also why the choice between them is not about `file://` or about rendering. It is structural and commercial.

## What Median actually is

Median.co, formerly GoNative, has been building webview apps since 2014, which makes it one of the most proven platforms in this category. You configure your app in a browser-based App Studio, add capability through a Native Plugin Library, and wire native features into your web code with the Median JavaScript Bridge and an optional NPM package. The plugin set is real: push through OneSignal and others, Face ID and Touch ID, social login, QR scanning, in-app purchases, native navigation. Web content updates appear over the air. Median also runs a fully managed service where their team builds, publishes, and keeps your app updated ahead of each OS release.

That is not a thin wrapper, and calling it one would be dishonest. Median is trusted by enterprise IT teams and product owners, and it explicitly supports apps built on Lovable and similar tools. Compared to a bare URL-in-a-shell service, it is well above that line.

## Where Despia and Median are the same

For a Lovable app, both do the same core job. Both run your web app in the platform webview. Both expose native features through a JavaScript bridge. Both load your live Lovable URL and update web content over the air with no store resubmission, so Supabase stays the single source of truth on either one. Neither saddles you with a second codebase the way the Capacitor export does. On architecture, they are the same class of product. The differences that matter are ownership, pricing, and tooling.

## The real difference: one-time and self-hosted, or subscription and managed

Despia is a one-time per-app license, around $250, self-hosted with no vendor lock-in. Despia never holds your hosting, the JavaScript SDK is MIT licensed, and its 50-plus native APIs are part of the runtime, not an upsell per feature. You can also export a complete Xcode and Android Studio project and keep working native source.

Median is a recurring subscription, and its plugins are tiered. Popular integrations sit in an Essential tier, more advanced ones in a Plus tier, and the fully managed, enterprise service is priced well into the thousands. Median is not strictly locked down either: it offers source access on its higher Professional licenses, so ownership is not purely either-or. But its default flow keeps your app managed on the platform. Reviewers on G2 repeatedly flag the pricing as opaque and note being upcharged for plugins they thought were included, with features like biometrics costing more on top.

So the honest framing is one-time and self-hosted versus subscription and managed. With Despia you pay once and keep the code and the hosting. With Median you subscribe to a managed platform and pay as you add features, with source available on its higher tiers.

## The AI-first difference for a Lovable builder

Both platforms have a bridge, but they are built for different hands. Median's bridge is built for a developer configuring an app in App Studio. Despia's is built for an AI agent writing code, which is how you built the app in the first place.

Despia's native calls are flat command strings, and Despia publishes an MCP server you add to Lovable as a custom chat connector through the New MCP server entry in Lovable's chat surface:

```text
https://setup.despia.com/mcp
```

Once connected, the Lovable agent writes the native calls itself, because a flat string is something an AI generates reliably, with no nested config objects to get wrong.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// Lovable's agent can emit this directly once the MCP is connected
if (isDespia) despia(`setonesignalplayerid://?user_id=${userId}`)
```

For someone shipping a Lovable app by prompting rather than hand-coding, that is the difference between the AI adding the native feature and you translating a plugin's setup into code by hand.

## Side by side

|  | Median.co | Despia |
| --- | --- | --- |
| Architecture | Webview plus JS bridge | Webview plus JS bridge |
| Loads your live Lovable URL | Yes | Yes |
| Supabase auth works | Yes | Yes |
| Web updates over the air | Yes | Yes |
| Pricing model | Recurring subscription, plugins tiered | One-time per-app license, around $250 |
| Premium features | Essential and Plus plugin tiers, some upcharged | 50-plus capabilities included in the runtime |
| Hosting | Platform-managed | Self-hosted, Despia never holds hosting |
| Native source access | On the Professional license | Included, with full Xcode and Android Studio export, MIT SDK |
| Managed build and publish | Yes, full-service tier | Self-serve from the browser |
| AI-agent tooling | Configure in App Studio | MCP added as a Lovable chat connector |
| Track record | Since 2014 | Runtime since 2011, Despia since 2023 |

## When Median is the right call

If you want a vendor to own the entire app lifecycle, Median is a genuinely strong choice and Despia does not try to replicate it. Median's managed service builds your app, handles Apple and Google submission with guaranteed acceptance, and proactively updates ahead of OS releases so it never gets delisted. For a team with budget that wants zero involvement in build and publish, that is worth paying for, and Median has run it at enterprise scale since 2014. Buying an outcome instead of a tool is a legitimate decision.

Despia is the opposite bet: you keep the code and the hosting, you pay once, and you point Lovable's agent at an MCP so the app writes itself. For a Lovable builder shipping their own product, that usually fits better. For an enterprise that wants a managed vendor relationship, Median may.

## Ship one app, not two

All three routes get a Lovable app into the stores, but they are not the same deal. The Capacitor export hands you a native project and the `file://` problems that come with it. Median is a managed platform you subscribe to, strongest when you want a vendor to run the lifecycle. Despia is a one-time, self-hosted license that keeps Supabase working and lets your AI agent do the wiring. Pick based on whether you want to own the setup or hand off the lifecycle.

[Learn more about Despia in the docs](https://setup.despia.com) or [start building at despia.com](https://despia.com).