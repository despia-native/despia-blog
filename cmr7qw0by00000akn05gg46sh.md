---
title: "Convert Base44 to a Mobile App and Publish It"
seoTitle: "Convert Base44 to a Mobile App and Publish It"
seoDescription: "A full walkthrough to convert a Base44 app to a mobile app and publish it on the App Store and Google Play, with native features and OTA via Despia."
datePublished: 2026-07-05T12:05:59.881Z
cuid: cmr7qw0by00000akn05gg46sh
slug: convert-base44-to-a-mobile-app-and-publish-it
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/0c20829a-4658-464c-8b3b-2b1fcca2728f.png
tags: webview, mobile-app, web-to-app, base44

---

You built a working app in Base44. It runs in the browser on Base44's backend and does what you want. The next step, getting it onto the App Store and Google Play as a real native app, is where most people stall. And this is not toy-level native: Despia reaches background GPS that survives a force-kill, NFC read and write, Apple Health, native OAuth, on-device Vision OCR, speech recognition, and persistent WebSockets, alongside the everyday push, biometrics, and in-app purchases. This walks the whole path, from a Base44 web app to a signed native binary on both stores, and it is honest about where Despia fits, where it does not, and the one Base44-specific thing you have to plan for.

## What going native actually requires

Base44 gives you the web app plus a zero-ops backend and database. To ship it as a native app you need three more things: a signed native binary for each store, native device features the browser cannot reach, and a way through App Store and Play Console review. You can assemble all of that yourself with Xcode, Android Studio, and a Mac, or you can keep your Base44 codebase as the source of truth and let Despia produce the binary and the native features from it.

Despia runs your Base44 web app inside the platform WebView (WKWebView on iOS, a Chromium-based WebView on Android) and exposes 50+ native features through a single JavaScript function. You keep one codebase, and Base44 stays your backend.

## Connect the Despia MCP to Base44

If you have not used Despia before, do this first, because it saves you from hand-writing native code. Despia publishes an MCP server that teaches the Base44 AI how the `despia-native` package works, so it writes correct native calls instead of guessing.

Base44 supports custom MCP connections on the Builder plan and above. You add it once at the account level: click your profile icon at the top right, open Settings, go to Account, then MCP connections, and click Add custom MCP. Name it Despia, paste the URL, and set Authentication to Not required.

```plaintext
https://setup.despia.com/mcp
```

Click Test, then Test and add. The connection is then available in the AI chat for every app on your account.

MCP connections only feed the editor chat while you build, so this is context for the AI, not a runtime dependency in your app. Before you build anything native, ask the chat a direct question and name the server, for example: using the Despia MCP, can Despia support every native feature this app needs. The answer decides your whole path, so ask it early.

## Check your feature list against what Despia supports

Despia covers the native surface most apps need: push notifications, Face ID and Touch ID, camera, GPS, NFC, in-app purchases, native OAuth, offline storage, haptics, and more, all through `despia()`. For the large majority of Base44 apps, that is the entire list.

It is not the entire list for every app, and pretending otherwise would waste your time. Despia supports the features it ships. If your app depends on something outside that set, for example heavy real-time AR, a custom Metal or OpenGL renderer, or a niche native SDK that Despia does not bridge, you are past what a web-native runtime can do for you. Despia still gives you a way forward: you can export the full Xcode and Android Studio projects at any time and continue in native code. That export is the honest escape hatch, and it is why checking your feature list against the docs before you commit is worth the five minutes.

## The one Base44-specific catch: native login

This is the part that is different from other builders, so plan for it before you submit. Base44 ships a convenient built-in login system (`base44.auth`), but it is locked down: you do not control the session tokens it issues. Native OAuth on the stores works by opening sign-in in a real secure browser session through Despia's `oauth://` bridge, then handing tokens back to your app. That handoff needs tokens you own, which Base44's managed login does not give you. Embedded-WebView OAuth is also exactly what Google and Apple block, so leaning on the built-in flow tends to fail review or fail outright on device.

You have two clean options.

Own your auth stack. Replace `base44.auth` with your own login: a users table as a normal Base44 entity you control, your own signed JWT sessions, and Google (or any provider) running natively through the `oauth://` bridge. You keep Base44's backend and database, you just take back the auth layer. Despia maintains a working template that does exactly this, so you are not starting from a blank page:

[Base44 native auth boilerplate on GitHub](https://github.com/despia-native/base44-native-boilerplate)

The repo includes `DESPIA_OAUTH.md` for the full mental model of how Despia, Base44, and the provider fit together, `JWT_AUTH.md` for the session setup, and `TEMPLATE_SETUP.md` for the handful of spots you change to make it yours.

Ship without social login first. If you want to be on the stores sooner, launch with email login only, skip third-party social sign-in entirely, get approved, and migrate to the custom-auth setup later. This avoids the embedded-WebView OAuth problem completely for your first release.

Either path is fine. What you should not do is submit an app whose only sign-in is Base44's managed social login running inside the WebView and expect it to pass.

## Keep building in Base44, ship updates over the air

Once you are on Despia your workflow does not change. You keep building in Base44. Because the web app is the source of truth, web changes ship to your installed apps over the air through remote hydration, with no App Store or Play Console resubmission. Change a screen, publish in Base44, force close the app, reopen, and the change is there.

You only rebuild the binary when you add a native capability that has to be compiled in, such as switching on a new integration. Content, layout, copy, and logic changes all go out over the air.

One detail AI builders get wrong: native calls must be gated to the Despia runtime, and the right check is the user agent, never a `window.despia` object.

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// native schemes only resolve inside the runtime, so gate them
if (isDespia) {
  despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)
}
```

## Despia and Base44 Mobile Submit

The alternative most Base44 users reach for first is Base44's own Mobile Submit, so it deserves a straight answer. Mobile Submit is a convenience layer and a good one for what it targets: it wraps your published app in a native shell that opens your URL, scans the result against the current Apple and Google guidelines, uses the Base44 AI to walk you through fixes until your readiness score is high enough, and generates the store bundles. Content and design changes you publish in Base44 show up without a resubmission, because the shell loads your live URL. If your app is a content experience, a directory, a dashboard, or a form-driven tool, that is often enough, and you never leave the editor.

The ceiling is in Base44's own docs. The shell runs your app in a secure web view, so push notifications and full offline mode are not supported, and neither is HealthKit. There is no StoreKit or Play Billing, so you cannot sell digital goods, and routing them through Stripe gets the app rejected. Permissions come from an AI scan and are not editable, the start URL is chosen for you, and the bundle ID is fixed at `com.base[app-id].app`, which causes signing-key mismatches if you ever shipped the same brand from another tool.

Despia also runs your published Base44 app in the platform WebView, but exposes 50+ native capabilities to that web layer through one JavaScript function, ships web updates over the air, and exports a full Xcode and Android Studio project if you outgrow it. It is still one codebase with Base44 as the source of truth. The features it adds are exactly the ones Mobile Submit lists as unsupported.

Push is the clearest example. Link each device to its Base44 user on every authenticated load, then your backend can reach that user through OneSignal even when the app is closed:

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// tie the device to the logged-in Base44 user so backend pushes can target them
if (isDespia) {
  despia(`setonesignalplayerid://?user_id=${userId}`)
}
```

| Capability | Base44 Mobile Submit | Despia |
| --- | --- | --- |
| Runs your published Base44 app | Yes, in a webview | Yes, in a native runtime |
| One codebase, no second project | Yes | Yes |
| Push notifications | Not supported | OneSignal with external user ID targeting |
| Digital goods (subscriptions, unlocks) | No StoreKit or Play Billing | RevenueCat, App Store and Play Billing |
| Physical goods payments | Stripe in webview risks rejection | Native Stripe PaymentSheet |
| Offline mode | Not supported | Local server, serves from localhost |
| Biometrics | Not available | Face ID, Touch ID, fingerprint |
| Other device features | Webview only | 50+ including GPS, haptics, NFC, HealthKit |
| Over-the-air web updates | Yes | Yes, via remote hydration |
| Bundle ID control | Fixed | Configurable |
| Native export escape hatch | No | Full Xcode and Android Studio projects |

There is a second reason this matters, and it is about getting approved at all. Apple and Google reject apps that read as a repackaged website under their minimum-functionality rules, and a pure webview shell with no native behaviour is the textbook trigger. The standard fix is to add real native features so the app does more than a browser tab, which is precisely the set Despia provides and Mobile Submit does not. So Despia is not only adding capability, it is lowering your odds of a minimum-functionality rejection.

A few of the native features are worth calling out.

OAuth is the clearest one, and it is the same reason the Base44 login catch above exists. Native WebViews are where Google and Apple block embedded OAuth, so managed-login sign-in tends to break. Despia opens sign-in in a real secure browser session through the `oauth://` bridge, using ASWebAuthenticationSession on iOS and Chrome Custom Tabs on Android, which is what the stores require.

Performance is the myth. The idea that a webview app is inherently slow does not hold. Despia runs on the same WebView engines that power fast production apps, and when a Despia app feels slow the cause is almost always the web app itself. The docs include a performance guide that covers the usual culprits.

Free-trial fraud and the expensive features are handled too. Despia's Storage Vault survives uninstall and reinstall, so a user cannot wipe the app to reset a free trial, and you can lock sensitive values behind Face ID. NFC, GPS, and other capabilities that normally sit behind paid tiers on other tools are included at no extra cost.

Use Mobile Submit when the app genuinely is a read-mostly content experience and you want zero extra surface area, since for a v1 you want in front of users this week that can be the right trade. The line is simple: the moment the app needs to notify, charge, authenticate with a fingerprint, or work offline, Mobile Submit's webview cannot reach those and Despia's runtime can, from the same Base44 codebase.

One thing no framework fixes for you is the layout. If your app looks like a website, with sidebars, hamburger menus, and desktop navigation, reviewers reject it regardless of how it was built. Get the mobile-first structure right, a fixed shell with one scrolling area, bottom navigation, and safe-area handling, and the native features do the rest.

## Why not Capacitor

Capacitor is the serious alternative, and its plugin ecosystem is genuinely larger than Despia's. If your app needs a long tail of specialised native integrations, that breadth is a real advantage and worth being straight about.

For most apps it is breadth you never use. The features 99% of apps actually ship, push, biometrics, camera, GPS, NFC, in-app purchases, OAuth, offline storage, are the ones Despia provides out of the box. Because those decisions are made for you, the rest of the pipeline gets simpler in ways that matter more day to day than plugin count:

Deployment runs entirely in the browser. Despia builds and submits with no Xcode and no Android Studio, while Capacitor keeps you in the native toolchains and a Mac for iOS.

Hosting stays yours. Despia serves your web layer from where you already run it, in this case Base44, with no migration.

Over-the-air updates are free at any scale. Capacitor's live updates run through third-party services that bill by monthly active users, so your update costs climb as you grow. Despia ships OTA updates over unlimited MAUs at no cost, which is the difference that shows up once the app takes off.

So the honest split is this: reach for Capacitor when you truly need its plugin depth or your own native code alongside it. Choose Despia when you want the native features almost every app needs, a browser-only pipeline, hosting that stays on Base44, and OTA that does not get more expensive as you scale.

## Publish to the App Store and Google Play

When the app is ready, Despia builds the signed binaries and submits from the browser, with no CLI and no Mac required. The two walkthroughs below cover each store end to end. Do iOS first if you are shipping both, since App Store review usually takes longer.

iOS: [https://setup.despia.com/deployment/apple-ios/automatic](https://setup.despia.com/deployment/apple-ios/automatic)

Android: [https://setup.despia.com/deployment/google-android/automatic](https://setup.despia.com/deployment/google-android/automatic)

Short version: set your app icon and splash, register your bundle ID, build, then submit through TestFlight and the Play Console testing track before promoting to production. On Google Play you now register the package name before you create the listing, so copy your bundle ID from Despia first and register it exactly, since it cannot be changed later.

## If you get rejected

Rejections happen to everyone, wrapper or not. What matters is what you do next. Despia keeps an indexed list of the common App Store and Play Store rejections with the specific fix for each, covering minimum functionality, login-service requirements, privacy strings, and more. Start there, apply the fix, resubmit.

[Read the store rejection playbook](https://setup.despia.com/store-rejections/introduction)

The point of Despia is not to wrap your app and wish you luck. It is to get it approved, with the native features and the fix playbook that actually clear review.

## Get it on the stores

Take the app you built in Base44 and ship it to iOS and Android without a CLI or a Mac. Keep Base44 as your backend, own your auth layer with the native boilerplate, keep updating over the air, and submit from the browser.

[See the setup docs at setup.despia.com](https://setup.despia.com) or [start building at despia.com](https://despia.com).