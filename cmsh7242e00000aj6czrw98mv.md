---
title: "Lovable Mobile App Cost: The Real 10-Year Math"
seoTitle: "Lovable Mobile App Cost: The Real 10-Year Math"
seoDescription: "What it actually costs to ship a Lovable app to the App Store and Google Play, comparing Capacitor and Despia across 1, 5 and 10 years."
datePublished: 2026-08-06T07:28:16.435Z
cuid: cmsh7242e00000aj6czrw98mv
slug: lovable-mobile-app-cost-the-real-10-year-math
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/6561088f-6641-46f3-86de-0607a506346a.png
tags: android-app-development, ios, android, webview, webapp, pwa, ios-app-development, capacitor, lovable

---

Lovable gives you two-way GitHub sync, so you already hold the repo. That makes the mobile step feel close to free: clone it, run `npx cap add ios`, done. The repo is the cheap part. The expensive part is the ten years afterwards, when someone has to keep a native project compiling against Apple's and Google's moving targets.

Here is the full arithmetic, with every number sourced and every assumption stated.

## The scenario

Fixing the scenario matters more than the pricing pages, because both routes get cheap or expensive depending on who you are.

*   You built the app in Lovable. It works, it is deployed, and the repo is synced to GitHub.
    
*   You ship it to both the App Store and Google Play.
    
*   It has 1,000 monthly active users.
    
*   You keep prompting Lovable to change the web app, maybe six meaningful releases a year.
    
*   You are not an iOS or Android developer, and you do not want to become one.
    

That last line is the one that moves the total by thousands of dollars, and it is the line most cost comparisons leave out.

## There is no free built-in route

Worth stating plainly, because Lovable users often assume there is one. Lovable has no built-in App Store or Google Play publishing. Your free fallback is a PWA, which means no store listing, no push on iOS unless the user manually adds it to their Home Screen, and no App Store presence at all.

If a store listing is the point, you are choosing between two paid routes, and the cost question becomes real.

One reassurance first, because it is a common fear: your Supabase backend does not need rebuilding for either route. It is an HTTPS API, and it behaves identically from a native client.

## Costs both routes pay

These are identical either way, so they are not the argument. Listing them anyway, because a total that hides them is not a total.

| Line item | Cost |
| --- | --- |
| Apple Developer Program | $99 per year |
| Google Play Console | $25 one time |
| Lovable plan, hosting, Supabase | Whatever you already pay |

Both routes ship under your own developer accounts. Neither tool holds your listing.

## What the Capacitor route actually costs

Capacitor itself is MIT licensed and free forever. That is real and worth saying plainly. What is not free is everything Capacitor deliberately leaves to you.

Open your repo and look. There is no `ios/` folder, no `android/` folder, and no `capacitor.config.ts`. Lovable builds with Vite, so the path is real and short: point `webDir` at `dist` and add the platforms. Ten minutes of work. Then the project is yours forever.

**Capacitor forks your Lovable workflow.** This is the Lovable-specific cost and it is not on any pricing page. You keep prompting, Lovable keeps changing the web code, and the native shell becomes a second thing you own and have to keep in step. Every plugin you add, every config change, every permission string now lives outside the tool you actually build in.

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

Your workflow does not fork. You keep prompting Lovable, you deploy, and the deployed app is the app in the stores. Web changes ship over the air the moment you deploy.

Native changes still need a rebuild, exactly like Capacitor, because that is a store rule and not a tool limitation. The difference is what the two re-hire scenarios cost you here. A native plugin change is a toggle in the dashboard and one build, which spends 5 of the 200 credits your license came with, against $50 to $200. The annual OS breaker is not your line item at all, because there is no `ios/` folder in your repo to migrate to Xcode 26 and no `targetSdk` in a Gradle file you own.

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

**The hired column is where most Lovable builders actually land**, because the whole reason you chose Lovable is that you do not write Swift. At $6,900 over ten years, the free library is the most expensive option on the table, and it stays expensive because the maintenance never ends.

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

## When Capacitor is the right call

Plainly, because a comparison that never concedes anything is an ad.

*   You have an in-house mobile developer, or you are one. The native project is an asset in that case, not a liability, and the cash cost is low.
    
*   You need custom native code or a specific plugin that is not in Despia's feature set. Capacitor's plugin ecosystem is large and open.
    
*   You have stopped prompting. If the Lovable phase is over and the repo is now a normal codebase you maintain by hand, the forked-workflow argument mostly evaporates.
    
*   You want the entire stack auditable and self-hostable. Capacitor is MIT, and Capgo's plugin and backend are open source under MIT and MPL-2.0.
    

## Common questions

**Does my Supabase backend need to change?** No. It is an HTTPS API and a native client talks to it exactly as the web app does. Nothing about the database, auth, or edge functions moves.

**Can I just ship a PWA instead?** You can, and it costs nothing, but there is no App Store listing at the end of it and iOS push requires the user to manually add the app to their Home Screen first.

**Does Despia charge for each store update?** No. Web changes ship over the air with no store review and no per-update fee. Native changes need a rebuild on any route, and on Despia it spends 5 credits from the 200 each license includes, which is 40 builds per app per OS. Beyond that, builds are $25 per pack of 10, flat rate with no per-minute metering.

**Is Capgo required with Capacitor?** No, but something is. Capacitor has no update mechanism of its own, so without an OTA service every web change becomes a store release with a review wait.

**Can I leave Despia later?** Yes. The Xcode project and Android Studio project export at any time, and the app is registered under your own Apple and Google accounts throughout.

## Before you budget, read these

*   [How to add a RevenueCat paywall to a web view app](https://blog.despia.com/how-to-add-a-revenuecat-paywall-to-a-webview-app)
    
*   [Lovable App Store approval: the rejection checklist](https://blog.despia.com/lovable-app-store-approval-the-rejection-checklist)
    
*   [Lovable app permissions: fix the 5.1.1 rejection](https://blog.despia.com/lovable-app-permissions-fix-the-5-1-1-rejection)
    

## Ship one app, not two

If you are going to keep building in Lovable, keep building there. Despia ships that app to the App Store and Google Play, gives it native device access, and updates it over the air, from the one codebase you already have.

[Learn more about Despia in the docs](https://setup.despia.com) or [start building at despia.com](https://despia.com).