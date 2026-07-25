---
title: "Web App to Mobile App: Stop Thinking Wrapper"
seoTitle: "Web App to Mobile App: Stop Thinking Wrapper"
seoDescription: "Turning a web app into a mobile app in 2026 is not about picking a wrapper. Here is the mental model and codebase structure that actually ships"
datePublished: 2026-07-22T13:51:05.258Z
cuid: cmrw54mxm00000ajbgyl605ik
slug: web-app-to-mobile-app-stop-thinking-wrapper
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/5ec109c2-fe0d-4403-bb11-44c2149d6e62.png
tags: webview, webapp, pwa, capacitor

---

You built a web app, it makes money, and now you want it on the App Store and Google Play. The first question everyone asks is "which wrapper do I use." That is the wrong question, and starting there is how people end up with an app that gets rejected or feels like a website in a phone-shaped frame. This post covers the two things that actually matter: the mental model that keeps you shippable, and the codebase structure that keeps you sane.

## The wrapper is dead in 2026

If you point a native shell at your web URL and submit it, expect a rejection. Apple and Google both look for apps that exist only to display a website, and both have gotten good at spotting them. A page in a shell is not an app to them, and increasingly not to your users either.

So the thing people call a "wrapper" no longer works as a mental model. What works is providing real native value: in-app purchases, Face ID, haptic feedback, iCloud key-value storage, push notifications, native share sheets. Once your app does those things, it is not a wrapped page. It is a native app that happens to be written in HTML.

That reframe is the whole game. The mental model to hold in your head is: **I am building a native app without native UI.** Your rendering layer is the web platform, which is excellent at UI. Your capability layer is native. You are not wrapping anything. You are building the real thing with a web frontend.

## Native value is a JavaScript call, not a rewrite

The reason this reframe is practical and not just semantics: adding native capability does not mean leaving your web stack. In Despia, every native feature is one `despia()` call from inside the web codebase you already have. The pattern is always the same, and the detection is always the canonical user agent check.

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// native calls live inside the gate so the same code runs fine on the web
if (isDespia) despia('successhaptic://')
```

Some features return a value, so you `await` the call instead of firing and forgetting. Gating a session token behind Face ID is the clean example: store it once with `locked=true`, and every read triggers the biometric prompt before the value is handed back to your JavaScript.

```javascript
// write once; locked=true means any future read requires Face ID / Touch ID
await despia('setvault://?key=sessionToken&value=abc123&locked=true')

// the read prompts for biometrics and only resolves the token on success
const data  = await despia('readvault://?key=sessionToken', ['sessionToken'])
const token = data.sessionToken
```

Storage Vault is backed by iCloud Key-Value Store on iOS and Android Key/Value Backup on Android, so the value survives uninstall and syncs across the user's devices on the same account. In real code these run inside the `isDespia` gate with a web fallback, same as the haptic call above. Storage Vault, in-app purchases, haptics, GPS, and dozens of other features all follow that one shape: a `despia()` call from the web code you already have. You are not maintaining a native project to get them.

## One codebase, two folders, not two codebases

Here is the common instinct, and it is half right: duplicate the app into two codebases, one tuned for web and one tuned to feel native (sticky bottom nav, bottom sheets instead of modals, native pickers, proper input types). The goal is correct. A first-class web experience and a first-class mobile experience really do want different UI. But two separate codebases is the wrong way to get there.

Split the UI, not the repo. One project, two folders:

```plaintext
/src
  /web       # web-first UI: modals, hover states, wide multi-column layouts
  /app       # mobile-first UI: bottom sheets, sticky nav, native pickers
  /shared    # data layer, business logic, everything platform-agnostic
```

You get the clean separation you wanted. `/web` is built for the browser, `/app` is built to feel native, and neither compromises for the other. But your business logic lives once in `/shared`, so a pricing change or an auth fix happens in one place instead of two.

The underrated benefit in 2026: everything stays in one AI context. When your components live in the same repo, your coding assistant can see the whole app at once, which makes refactors and feature work dramatically faster than juggling two projects that drift apart. Two codebases double your maintenance and split the context your tools depend on. One codebase with two UI folders costs you a folder and buys you both experiences.

## The alternative worth knowing: Capacitor plus Capgo

Despia is not the only way to do this, and the honest comparison is with Capacitor. Capacitor is a mature, open-source native runtime from the Ionic team. It runs your web app in a native WebView, has a large plugin ecosystem, and gives you full control over the native projects. It is a solid, well-supported choice, especially if you want to own the Xcode and Android Studio projects directly and do not mind the native tooling that comes with that.

Capacitor does not ship over-the-air updates on its own. For that you add Capgo, the open-source live-update platform for Capacitor, which pushes JS, HTML, and CSS changes straight to users without a store review. Capgo is best-in-class for what it does and worth knowing about regardless of which runtime you pick.

One thing to check early if you sell digital goods: Apple and Google require their own billing systems for subscriptions and premium features, and will reject an app that runs Stripe for digital content inside the native binary. So you will need native in-app purchases wired up (through RevenueCat or equivalent) on whichever path you pick. Despia ships that as a built-in; on raw Capacitor it is another plugin to add.

Despia includes over-the-air updates as part of the runtime, no separate service to wire up. The default model is remote hydration: the binary ships without embedded web assets and fetches your current build from your hosting URL on each launch, so web changes never need an App Store or Play resubmission. For offline-first apps, `@despia/local` caches the full asset graph on device, serves it from `http://localhost`, and still applies OTA updates atomically on the next launch.

|  | Despia | Capacitor + Capgo |
| --- | --- | --- |
| Native runtime | Built in | Capacitor |
| Native features | Dozens via one `despia()` call | Plugin ecosystem, some assembly |
| OTA updates | Included (remote hydration by default) | Add Capgo separately |
| Native project ownership | Export the full Xcode / Android Studio project anytime | You own the native projects from day one |
| Build and submit without a Mac | Yes, from the browser | Needs local native tooling |

## When Capacitor is the right call

If you want to live in the native projects, write your own plugins, and treat the web layer as one part of a larger native app you actively maintain, Capacitor is the better fit. It gives you that control by design, and the Capgo pairing for OTA is excellent. Choose it when native ownership is a feature you want, not overhead you are trying to avoid.

Choose Despia when you want the native binary, the native features, and the over-the-air updates without standing up and maintaining a native project. Same web codebase, native capability included, export hatch still there if you ever want to drop into Xcode.

Either way, the decision that matters is not the runtime. It is dropping the word "wrapper" and building a native app that happens to be written in HTML.

## Ship one app, not two

If you are going to keep building in your web stack, keep building there. Despia ships that app to the App Store and Google Play, gives it native device access, and updates it over the air, from the one codebase you already have.

[Learn more about Despia in the docs](https://setup.despia.com)