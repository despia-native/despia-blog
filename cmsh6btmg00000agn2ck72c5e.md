---
title: "Vibe Coding Mobile App Cost: 1, 5 and 10 Years"
seoTitle: "Vibe Coding Mobile App Cost: 1, 5 and 10 Years"
seoDescription: "You shipped a web app with Cursor or Claude Code. Here is what getting it into the App Store really costs on Capacitor versus Despia, over ten years."
datePublished: 2026-08-06T07:07:49.841Z
cuid: cmsh6btmg00000agn2ck72c5e
slug: vibe-coding-mobile-app-cost-1-5-and-10-years
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/e4f76b0f-000c-4a11-b89e-34626e140752.png
tags: apps, react-native, flutter, capacitor, ai-tools

---

You have a terminal, a repo, and an agent that will happily run `npx cap add ios` and tell you it worked. That is the part that is genuinely cheap now. The part that has not gotten cheaper is the ten years afterwards, when someone has to keep a native project compiling against Apple's and Google's moving targets, and your agent's training data is a year behind the deadline that just broke your build.

Here is the full arithmetic, with every number sourced and every assumption stated.

## The scenario

Fixing the scenario matters more than the pricing pages, because both routes get cheap or expensive depending on who you are.

*   You built a web app with Cursor, Claude Code, or something similar. React, Vue, Svelte, whatever the agent reached for. It is built and hosted.
    
*   You ship it to both the App Store and Google Play.
    
*   It has 1,000 monthly active users.
    
*   You release web changes maybe six times a year.
    
*   You can read code and run commands, but you have never shipped a native app and you do not want the native project to become your job.
    

That last line is where the money is. You are not the no-code builder who cannot open a terminal, and you are also not an iOS developer. The costs land differently for you than for either.

## Costs both routes pay

These are identical either way, so they are not the argument. Listing them anyway, because a total that hides them is not a total.

| Line item | Cost |
| --- | --- |
| Apple Developer Program | $99 per year |
| Google Play Console | $25 one time |
| Web hosting | Whatever you already pay |

Both routes ship under your own developer accounts. Neither tool holds your listing.

## What Capacitor actually costs

Capacitor itself is MIT licensed and free forever. That is real and worth saying plainly, and for someone with a working terminal it is a genuine option in a way it is not for a no-code builder. What is not free is everything Capacitor deliberately leaves to you.

**The thing your agent is good at is the cheap part.** Generating `capacitor.config.ts`, pointing `webDir` at your build output, running `npx cap add ios android`. Twenty minutes, and the agent will do it well. That work was never the expensive part.

**The thing your agent is bad at is the recurring part.** Native project maintenance is a moving target against SDK versions and store policies that changed after training. The failure mode is specific and it costs you real cycles: an agent will emit a plugin method that is real, documented, and idiomatic, just for a different package or a different major version than the one in your `package.json`. It compiles or it does not, you find out in a build, and every one of those round trips is a native rebuild rather than a hot reload. A web-only project absorbs that cheaply. A native project does not.

**Over-the-air updates.** Capacitor has no update service. The standard answer is [Capgo](https://capgo.app/pricing/), and it is a good product. At 1,000 MAU you need the Solo plan, which is $12 a month billed annually at $146, and includes 2,000 MAU, 100 GiB of bandwidth, unlimited live updates, and one build hour a month. There is also a credits-only pay-as-you-go option at $0.003 per MAU plus $0.06 per GiB, which for a small app lands closer to $40 to $60 a year if you are willing to manage prepaid credits. Either way it is a subscription, and it is metered on how many people use your app.

**A build machine.** Historically this meant buying a Mac. It no longer has to. Capgo Native Build runs macOS M4 machines and includes App Store publishing on every plan, so the Mac requirement is gone if you are already paying for Capgo. Solo includes one build hour a month, with overage at $0.08 per build minute.

**The annual compliance work.** Capacitor generates `ios/` and `android/` folders that are yours to keep. That is the whole design. You own the Podfile, the Gradle config, the Info.plist, the Android manifest, the plugin versions, and the target SDK. Capgo's own documentation is explicit about the boundary: changes to `capacitor.config.ts`, native plugin configuration, native package installs or upgrades, and anything requiring `npx cap sync` all need a native app release, not a live update.

Two of those releases are mandatory every year whether your product changes or not:

*   Since [April 28, 2026](https://developer.apple.com/news/upcoming-requirements/), every upload to App Store Connect must be built with Xcode 26 and the iOS 26 SDK. Apple resets this baseline annually.
    
*   From [August 31, 2026](https://developer.android.com/google/play/requirements/target-sdk), new Google Play submissions and updates must target Android 16 (API 36), and apps that fall below API 35 stop being discoverable to new users on newer devices. Google runs this on an annual clock too.
    

If you do not do that work yourself, you hire it. Rates run roughly like this:

| Who you hire | Typical range | What it covers |
| --- | --- | --- |
| Offshore freelancer | $150 to $350 | Native folders generated, icons and splash screens, base store config, binary upload |
| Experienced Capacitor freelancer | $400 to $800 | Native plugin setup, provisioning profiles, certificates, keystores, review guidance |
| Boutique agency or senior US/EU dev | $900 to $1,500+ | CI/CD pipeline, OTA service wiring, dedicated support through rejections |

Mobile developers generally bill $30 to $100 an hour, and a first store deployment is 4 to 12 hours of work. That is the setup fee. After that you come back for two specific reasons, indefinitely.

**Native plugin changes.** Adding Bluetooth, swapping push providers, wiring in a new SDK, or even changing the app icon all modify the native project and require a rebuilt binary. Expect $50 to $200 per update, or your own afternoon.

**Annual OS breakers.** Every year Apple and Google raise the submission floor, and every year some plugin in your dependency tree stops compiling against the new SDK. Keeping a Capacitor app compiling and compliant runs $150 to $500 a year in general maintenance, before you have shipped a single new feature.

## What Despia actually costs

$249 per app per OS, one time. One app on both stores is $498, paid once, for the life of the app.

Each license includes 200 cloud credits, and a build costs 5 credits, so that is 40 native builds per app per OS. A two-platform app starts with 80. At the release cadence in this scenario, two mandatory compliance rebuilds a year across both platforms plus the occasional icon or permission change, that is roughly a decade of releases before you buy a single build. If you do run out, builds come in packs of 10 for $25.

Some Despia users do burn through the whole allowance inside a year. That is almost always a workflow problem rather than a pricing one, because a build is for native changes and a web change that gets rebuilt instead of deployed over the air is a build spent for nothing. The table below still assumes a $25 pack every two years, which is deliberately pessimistic against an allowance that should cover the decade.

Builds are also flat rate. A build is a build: it is not metered by the minute, so a slow one costs the same as a fast one and there is no per-minute overage to model. Over-the-air updates are unlimited with no MAU metering. There is no plan, no renewal, and no line that scales with how many people open your app.

Web changes ship over the air the moment you deploy. Native changes still need a rebuild, exactly like Capacitor, because that is a store rule and not a tool limitation. The difference is what the two re-hire scenarios cost you here. A native plugin change is a toggle in the dashboard and one build, which spends 5 of the 200 credits your license came with, or one afternoon you get back. The annual OS breaker is not your line item at all, because there is no `ios/` folder in your repo to migrate to Xcode 26 and no `targetSdk` in a Gradle file you own.

There is also less surface for your agent to get wrong. Native capabilities are scheme calls in your web code, not a plugin graph with versions:

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

The middle column is the one that applies to you, and it is the honest competition. $245 a year is cheap, and you are capable of running it. The question is not whether you can, it is whether 20 to 30 hours in year one and 8 to 12 hours every year after is where you want your attention. That is a real trade and nobody can price your hours but you.

The hired column exists because plenty of people start in the middle column and end up in the right-hand one the first time an SDK migration eats a weekend. At $6,900 over ten years, the free library becomes the most expensive option on the table.

Despia's cost is front-loaded and then nearly flat. After the license, the recurring lines are the $99 Apple fee that every route pays and about $13 a year in builds, and that build figure is a pessimistic placeholder rather than a bill you should expect.

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

The obvious objection stands: a Despia app runs your web code in WKWebView and the Android WebView, and that is not SwiftUI. True. The answer is the 50-plus native capabilities exposed through one JavaScript call, and the fact that the full Xcode and Android Studio projects export whenever you want them. If you outgrow the managed path, you take the project and go.

## When Capacitor is the right call

Plainly, because a comparison that never concedes anything is an ad.

*   You want to own the native project and you are willing to learn it. This is the most defensible reason and it is a good one.
    
*   You need custom native code or a specific plugin outside Despia's feature set. Capacitor's plugin ecosystem is large and open, and writing your own plugin is a supported path.
    
*   You are shipping several apps and want one pipeline you control end to end.
    
*   You want the entire stack auditable and self-hostable. Capacitor is MIT, and Capgo's plugin and backend are open source under MIT and MPL-2.0.
    

## Common questions

**Can my agent handle the annual SDK migration?** Partly. It can bump versions and fix obvious compile errors. It struggles where the correct answer changed after its training cutoff, which is exactly what an annual store deadline is. Budget the hours rather than assuming they are free.

**I already have Cursor and a Mac. Should I just use Capacitor?** Possibly, and the middle column is the honest number for you. Run it if you want to own the native project. Do not run it because the setup step looked cheap, since setup is not the recurring cost.

**Does Despia charge for each store update?** No. Web changes ship over the air with no store review and no per-update fee. Native changes need a rebuild on any route, and on Despia it spends 5 credits from the 200 each license includes, which is 40 builds per app per OS. Beyond that, builds are $25 per pack of 10, flat rate with no per-minute metering.

**Is Capgo required with Capacitor?** No, but something is. Capacitor has no update mechanism of its own, so without an OTA service every web change becomes a store release with a review wait.

**Can I leave Despia later?** Yes. The Xcode project and Android Studio project export at any time, and the app is registered under your own Apple and Google accounts throughout.

## Before you budget, read these

*   [Convert your web app to iOS and Android apps](https://blog.despia.com/convert-your-web-app-to-ios-and-android-apps)
    
*   [The Despia MCP server: stop burning credits on bad code](https://blog.despia.com/despia-mcp-server-stop-burning-credits-on-bad-code)
    

## Ship one app, not two

If you are going to keep building in your web stack, keep building there. Despia ships that app to the App Store and Google Play, gives it native device access, and updates it over the air, from the one codebase you already have.

[Learn more about Despia in the docs](https://setup.despia.com) or [start building at despia.com](https://despia.com).