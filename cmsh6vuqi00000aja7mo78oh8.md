---
title: "Base44 App Store Cost: The Real 10-Year Math"
seoTitle: "Base44 App Store Cost: The Real 10-Year Math"
seoDescription: "What it really costs to keep a Base44 app on the App Store and Google Play, across built-in publishing, Capacitor and Despia, over 10 years."
datePublished: 2026-08-06T07:23:24.414Z
cuid: cmsh6vuqi00000aja7mo78oh8
slug: base44-app-store-cost-the-real-10-year-math
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/00307de9-9fae-4487-8b9f-cea7968bd61f.png
tags: android-app-development, ios, android, webapp, pwa, ios-app-development, capacitor, base44

---

Every Base44 user pricing out a mobile launch hits the same three numbers: Base44's built-in publishing is included, Capacitor is free, and Despia is $249 per app per OS. Two of those three are true and misleading, because none of them is the cost of keeping an app in the stores for ten years.

Here is the full arithmetic, with every number sourced and every assumption stated.

## The scenario

Fixing the scenario matters more than the pricing pages, because every route gets cheap or expensive depending on who you are.

*   You built the app in Base44. It works, it is hosted, and you are done prompting for now.
    
*   You ship it to both the App Store and Google Play.
    
*   It has 1,000 monthly active users.
    
*   You release web changes maybe six times a year.
    
*   You are not an iOS or Android developer, and you do not want to become one.
    

That last line is the one that moves the total by thousands of dollars, and it is the line most cost comparisons leave out.

## Start with the route that costs nothing

Base44 has built-in store publishing, and for some apps it is the correct answer. Do not skip past it because it is free.

The limits are stated in [Base44's own documentation](https://docs.base44.com/documentation/building-your-app/uploading-to-app-stores): the mobile output is a web view, with no push notifications and no full offline support. If your app has no payments, no push, and no device access, that is a genuinely fine place to stop, and everything below this line is money you do not need to spend.

If your app needs push, in-app purchases, biometrics, background location, or anything else Apple looks for under [Guideline 4.2](https://developer.apple.com/app-store/review/guidelines/#minimum-functionality), you are choosing between the other two routes, and the cost question becomes real.

## Costs every route pays

These are identical whichever way you go, so they are not the argument. Listing them anyway, because a total that hides them is not a total.

| Line item | Cost |
| --- | --- |
| Apple Developer Program | $99 per year |
| Google Play Console | $25 one time |
| Base44 hosting and plan | Whatever you already pay |

Both paid routes ship under your own developer accounts. Neither tool holds your listing.

## What the Capacitor route actually costs

Capacitor itself is MIT licensed and free forever. That is real and worth saying plainly. What is not free is everything around it.

**The export is not free and not complete.** Both ZIP and GitHub export require a paid Base44 plan, so the free library starts behind a paywall. What you get is a Vite React project, which means `npm run build` to `dist` is the right entry point for Capacitor. What you do not get is the backend: entity schemas, live records, and backend logic stay on Base44, and your local `.env` still needs `VITE_BASE44_APP_ID` and `VITE_BASE44_APP_BASE_URL` pointing back at Base44's servers. Base44's own CLI has `login`, `create`, entity and function sync, deploy to Base44 hosting, and an `eject` command. None of those has a mobile build target.

**Over-the-air updates.** Capacitor has no update service. The standard answer is [Capgo](https://capgo.app/pricing/), and it is a good product. At 1,000 MAU you need the Solo plan, which is $12 a month billed annually at $146, and includes 2,000 MAU, 100 GiB of bandwidth, unlimited live updates, and one build hour a month. There is also a credits-only pay-as-you-go option at $0.003 per MAU plus $0.06 per GiB, which for a small app lands closer to $40 to $60 a year if you are willing to manage prepaid credits instead of a plan. Either way it is a subscription, and it is metered on how many people use your app.

**A build machine.** Historically this meant buying a Mac. It no longer has to. Capgo Native Build runs macOS M4 machines and includes App Store publishing on every plan, so the Mac requirement is gone if you are already paying for Capgo. Solo includes one build hour a month, with overage at $0.08 per build minute.

**The person who maintains the native project.** This is the line item that does not go away, and it is where the money is.

Capacitor generates `ios/` and `android/` folders that are yours to keep. That is the whole design. It means you own the Podfile, the Gradle config, the Info.plist, the Android manifest, the plugin versions, and the target SDK. Capgo's own documentation is explicit about the boundary: changes to `capacitor.config.ts`, native plugin configuration, native package installs or upgrades, and anything requiring `npx cap sync` all need a native app release, not a live update.

Two of those releases are mandatory every year whether your product changes or not:

*   Since [April 28, 2026](https://developer.apple.com/news/upcoming-requirements/), every upload to App Store Connect must be built with Xcode 26 and the iOS 26 SDK. Apple resets this baseline annually.
    
*   From [August 31, 2026](https://developer.android.com/google/play/requirements/target-sdk), new Google Play submissions and updates must target Android 16 (API 36), and apps that fall below API 35 stop being discoverable to new users on newer devices. Google runs this on an annual clock too.
    

If you cannot do that work, you hire it. Rates for Capacitor finalization and store submission run roughly like this:

| Who you hire | Typical range | What it covers |
| --- | --- | --- |
| Offshore freelancer | $150 to $350 | Native folders generated, icons and splash screens, base store config, binary upload |
| Experienced Capacitor freelancer | $400 to $800 | Native plugin setup, provisioning profiles, certificates, keystores, review guidance |
| Boutique agency or senior US/EU dev | $900 to $1,500+ | CI/CD pipeline, OTA service wiring, dedicated support through rejections |

Mobile developers generally bill $30 to $100 an hour, and a first store deployment is 4 to 12 hours of work. That is the setup fee. The part nobody budgets for is that you come back to the same developer for two specific reasons, indefinitely.

**Native plugin changes.** Adding Bluetooth, swapping push providers, wiring in a new SDK, or even changing the app icon all modify the native project and require a rebuilt binary. Expect $50 to $200 per update.

**Annual OS breakers.** Every year Apple and Google raise the submission floor and change privacy rules, and every year some plugin in your dependency tree stops compiling against the new SDK. Keeping a Capacitor app compiling and compliant runs $150 to $500 a year in general maintenance, before you have shipped a single new feature.

## What Despia actually costs

$249 per app per OS, one time. One app on both stores is $498, paid once, for the life of the app.

Each license includes 200 cloud credits, and a build costs 5 credits, so that is 40 native builds per app per OS. A two-platform app starts with 80. At the release cadence in this scenario, two mandatory compliance rebuilds a year across both platforms plus the occasional icon or permission change, that is roughly a decade of releases before you buy a single build. If you do run out, builds come in packs of 10 for $25.

Some Despia users do burn through the whole allowance inside a year. That is almost always a workflow problem rather than a pricing one, because a build is for native changes and a web change that gets rebuilt instead of deployed over the air is a build spent for nothing. The table below still assumes a $25 pack every two years, which is deliberately pessimistic against an allowance that should cover the decade.

Builds are also flat rate. A build is a build: it is not metered by the minute, so a slow one costs the same as a fast one and there is no per-minute overage to model. Over-the-air updates are unlimited with no MAU metering. There is no plan, no renewal, and no line that scales with how many people open your app.

Your Base44 backend does not change. It is an HTTPS API, and it behaves identically from a native client.

Web changes ship over the air the moment you deploy. Native changes still need a rebuild, exactly like Capacitor, because that is a store rule and not a tool limitation. The difference is what the two re-hire scenarios cost you here. A native plugin change is a toggle in the dashboard and one build, which spends 5 of the 200 credits your license came with, against $50 to $200. The annual OS breaker is not your line item at all, because there is no `ios/` folder in your repo to migrate to Xcode 26 and no `targetSdk` in a Gradle file you own.

Adding a native capability does not touch a native project either:

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

if (isDespia) despia('successhaptic://')
```

No `npm install`, no `npx cap sync`, no native release to add a feature that was already compiled into the runtime.

## The ten-year totals

Assumptions: one app on both stores, 1,000 MAU, six web releases a year, two mandatory native compliance rebuilds a year, Capgo Solo at the annual rate. Freelancer pricing taken at the middle of each range above: $600 for initial setup, $300 a year for OS compliance maintenance, and one native plugin change a year at $125. The Despia column assumes a $25 build pack every two years, which is conservative given two licenses already cover 80 builds.

|  | Despia | Capacitor, you own the native project | Capacitor, you hire it out |
| --- | --- | --- | --- |
| Year 1 | $622 | $270 | $870 |
| 5 years | $1,068 | $1,250 | $3,550 |
| 10 years | $1,638 | $2,475 | $6,900 |
| Recurring per year after setup | $99 store fee, plus about $13 a year in builds | $245 | $670 |
| Your hours, year one | Under 2 | 20 to 30 | 0 |
| Your hours, per year after | Under 2 | 8 to 12 | 0 |

Three things fall out of that table.

**The Capacitor DIY column is genuinely cheap in cash and expensive in hours.** If you already write code and do not mind a yearly Xcode migration, $245 a year is a defensible number. Price your own 8 to 12 hours a year at anything above $30 an hour and the cash advantage disappears, but that is your call to make.

**The hired column is where most Base44 builders actually land**, because the whole reason you chose Base44 is that you do not write Swift. At $6,900 over ten years, the free library is the most expensive option on the table, and it stays expensive because the maintenance never ends.

**Despia's cost is front-loaded and then nearly flat.** After the license, the recurring lines are the $99 Apple fee that every route pays and about $13 a year in builds, and that build figure is a pessimistic placeholder rather than a bill you should expect.

## The line that actually diverges: growth

The 1,000 MAU comparison is the friendly one for a metered service. Watch what happens when the app works.

| Monthly active users | Capgo annual | Despia annual |
| --- | --- | --- |
| 1,000 | $146 (Solo) | $0 |
| 10,000 | $396 (Maker) | $0 |
| 100,000 | $998 (Team) | $0 |
| 1,000,000 | $2,490+ (Enterprise) | $0 |

Capgo counts MAU as distinct devices contacting Capgo in a rolling 30-day window, and the same device counts separately per native app ID. Despia does not meter update delivery at all. Over ten years, that is the difference between a cost that grows with your success and a cost that was fixed the day you bought it.

## Why Despia costs more on day one

Two reasons, and neither is margin.

The first is that you are buying a runtime, not a library. Despia ships a compiled native layer with 50-plus device capabilities already in it, so the price includes the work that Capacitor pushes to plugin selection, native project maintenance, and your yearly SDK migration.

The second is that it is a license, not a subscription. The $249 is the whole software cost, decided once, at a price that does not move when your user count does. That is a worse deal for someone who ships an app, gets 40 users, and abandons it. It is a much better deal for anyone whose app survives.

The obvious objection stands: a Despia app runs your web code in WKWebView and the Android WebView, and that is not SwiftUI. True. The answer is the 50-plus native capabilities exposed through one JavaScript call, and the fact that the full Xcode and Android Studio projects export whenever you want them.

## When each route is the right call

Plainly, because a comparison that never concedes anything is an ad.

*   **Base44 built-in publishing** if your app has no payments, no push, and no device access. It is included and it works within the limits Base44 states.
    
*   **Export plus Capacitor** if you have a developer, or you are one. The native project is an asset in that case, not a liability.
    
*   **Despia** if you need the native features that get an app past review and you do not want to own a native project to get them.
    

## Common questions

**Does the Base44 CLI build a mobile app?** No. The CLI handles login, project creation, entity and function sync, deployment to Base44 hosting, and `eject`. There is no mobile build target in it.

**Does my Base44 backend need rebuilding for a native app?** No. Entity schemas, records, and backend logic stay on Base44, and a native client talks to the same HTTPS API the web app does.

**Does Despia charge for each store update?** No. Web changes ship over the air with no store review and no per-update fee. Native changes need a rebuild on any route, and on Despia it spends 5 credits from the 200 each license includes, which is 40 builds per app per OS. Beyond that, builds are $25 per pack of 10, flat rate with no per-minute metering.

**Is Capgo required with Capacitor?** No, but something is. Capacitor has no update mechanism of its own, so without an OTA service every web change becomes a store release with a review wait.

**Can I leave Despia later?** Yes. The Xcode project and Android Studio project export at any time, and the app is registered under your own Apple and Google accounts throughout.

## Before you budget, read these

*   [In-app purchases on Base44 without webhooks, using RevenueCat](https://blog.despia.com/base44-in-app-purchases-without-webhooks-revenuecat)
    
*   [Base44 App Store approval: the rejection checklist](https://blog.despia.com/base44-app-store-approval-the-rejection-checklist)
    
*   [Base44 app permissions: fix the 5.1.1 rejection](https://blog.despia.com/base44-app-permissions-fix-the-5-1-1-rejection)
    

## Ship one app, not two

If you are going to keep building in Base44, keep building there. Despia ships that app to the App Store and Google Play, gives it native device access, and updates it over the air, from the one project you already have.

[Learn more about Despia in the docs](https://setup.despia.com) or [start building at despia.com](https://despia.com).