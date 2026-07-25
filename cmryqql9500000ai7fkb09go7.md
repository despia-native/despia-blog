---
title: "Capacitor vs Despia: The True Cost for No-Coders"
seoTitle: "Capacitor vs Despia: The True Cost for No-Coders"
seoDescription: "Capacitor vs Despia on true cost: Capacitor is free, but a no-coder pays in developer hours and a per-user OTA bill. Here is the real math."
datePublished: 2026-07-24T09:31:33.811Z
cuid: cmryqql9500000ai7fkb09go7
slug: capacitor-vs-despia-the-true-cost-for-no-coders
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/dd2736c9-4275-466c-9d48-daae97f4d36f.png
tags: android-app-development, ios, android, webview, ios-app-development, capacitor, hybridapp, despia

---

Capacitor is free. That part is true, and it is the first thing anyone tells you. The question that matters if you built your app in Base44, Lovable, or plain HTML and cannot write Swift is different: what does it cost to get that app onto the App Store and Google Play, keep it updated, and keep it updated as you grow. The $0 headline answers none of that.

Here is the real math, with current numbers.

## What Capacitor actually is

Capacitor is an open-source native runtime from the Ionic team. It wraps your web build in a real iOS and Android project that you own outright. You get a full Xcode project, a full Android Studio project, a large plugin ecosystem, and the ability to drop into native code whenever you want. If you have a developer, or you are one, it is a genuinely good, well-supported foundation. Nothing in this post argues that Capacitor is bad software. It is not.

The catch is that Capacitor hands you a native project, and a native project has to be set up and maintained by someone who knows how.

## Where the $0 goes: the developer hours

A no-coder cannot ship a Capacitor app alone. The initial setup means installing Node and the Capacitor CLI, installing Xcode and Android Studio, generating signing certificates and provisioning profiles, running `npx cap sync`, wiring up each native plugin, and producing the first signed build. On a Mac, because iOS builds need one.

Realistically that is at least a couple of hours of a friendly developer's time to stand up, and it is not one-and-done. Every time you add or change a native plugin, or touch the Capacitor config, you need a native rebuild and another pass through that toolchain. So the "free" tool carries a recurring line item: someone technical, on call, whenever the native layer moves.

If you are paying for that help rather than borrowing it, developer time is the largest cost in this whole comparison, and it is the one the $0 headline hides.

## Where the $0 goes again: over-the-air updates

Capacitor has no built-in way to push a web fix without going back through App Store review. To update your JavaScript, CSS, and copy without a resubmission, you add a live-update service. The standard one is [Capgo](https://capgo.app/).

Capgo is a solid product, open-source plugin and all. But its hosted delivery is priced per Monthly Active User, and that is where the bill starts scaling with your success:

*   Solo: 2,000 MAU, $12 per month billed annually
    
*   Maker: 10,000 MAU, $33 per month billed annually
    
*   Team: 100,000 MAU, $83 per month billed annually
    

MAU is counted as distinct active devices in a rolling 30-day window. Cross your plan's ceiling and update delivery is blocked until you upgrade or buy credits. So the moment your app does well, the OTA bill goes up, and the thing you are paying for is the right to keep shipping fixes to the users you already earned.

None of this is a knock on Capgo. It is a real business with real infrastructure. It is just a second, recurring, usage-metered cost sitting on top of the "free" framework, and it exists because Capacitor does not host or update your web layer for you.

## The side-by-side

|  | Capacitor (+ Capgo for OTA) | Despia |
| --- | --- | --- |
| Framework license | Free, open source | One-time per-app/os license, around $250 |
| Setup for a no-coder | Needs a developer: Node, CLI, Xcode, Android Studio, signing | Browser-based, no Mac, no CLI |
| Native rebuild for config changes | Yes, back through the toolchain | Yes, but from the browser |
| Over-the-air web updates | Add a service (Capgo) | Built in |
| OTA pricing | Per MAU, recurring, blocks at the ceiling | Included, served from your own hosting |
| MAU cap on updates | Yes, tied to plan tier | None |
| Where updates are hosted | Capgo's cloud | Your existing web host |
| Codebase | Web project plus a native project to maintain | One web codebase, source of truth |
| Native escape hatch | Full native project ownership | Full Xcode and Android Studio export |

## How Despia changes the math

Despia takes the web app you already built and ships it as a real native binary for iOS and Android, with 50+ native device features available through a single JavaScript call. Two things about that binary matter for cost.

First, updates are built in and served from your own hosting. Despia's default deployment is remote hydration: the binary ships without embedded web assets and fetches the current build from your configured hosting URL on every launch. That is over-the-air updates, and it runs off the web host you already deploy to. There is no separate OTA vendor and no per-user delivery meter, because you are serving your own bundle from your own infrastructure. Ship a fix to your host, users get it on next launch. MAU does not enter into it.

Second, publishing does not need a developer's toolchain. App Store and Google Play submission runs from a browser. Code signing and provisioning are handled for you. No Mac, no CLI, no `npx cap sync`. The hours a no-coder would otherwise rent from a developer to stand up a Capacitor project mostly disappear, because there is no native project to stand up.

The pricing shape is the real difference. Capacitor plus Capgo is a free framework with a recurring, MAU-metered OTA bill and developer time on top. Despia is a one-time per-app license with OTA already included and hosted on infrastructure you control. One scales its cost with your growth. The other does not.

You can read exactly how the hydration model works in the [Despia introduction docs](https://setup.despia.com/introduction).

## When Capacitor is the right call

Concede the honest cases. If you already have native developers, or you want to own and shape the native project directly down to individual plugins and Swift or Kotlin code, Capacitor plus Capgo is a strong, mature stack and you should use it. If you are standardizing a team on the Ionic ecosystem, that is a good reason too. And if your app needs deep native customization beyond what a runtime's built-in feature set covers, the full control of a native project you maintain is worth the maintenance.

Despia answers the obvious objection here directly: a runtime-hosted web app is not a hand-built native project. True. The answers are the 50+ native features exposed through one JavaScript call, and the full native export, so if you outgrow the runtime you leave with complete Xcode and Android Studio projects and nothing is locked away.

The choice is not free versus paid. It is a metered bill that grows with your users versus a one-time license that does not.

## Ship one app, not two

If you are going to keep building in your web stack, keep building there. Despia ships that app to the App Store and Google Play, gives it native device access, and updates it over the air from your own hosting, out of the one codebase you already have.

[Learn more about Despia in the docs](https://setup.despia.com) or [start building at despia.com](https://despia.com).