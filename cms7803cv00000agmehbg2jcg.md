---
title: "Base44 Mobile App: Where the Wrapper Stops"
seoTitle: "Base44 Mobile App: Where the Wrapper Stops"
seoDescription: "Base44 packages your app for the stores. What its own docs list as unsupported, and how to add push, offline and billing when your app needs them."
datePublished: 2026-07-30T07:57:00.042Z
cuid: cms7803cv00000agmehbg2jcg
slug: base44-mobile-app-where-the-wrapper-stops
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/75e6cfb8-0744-414a-9eab-52f8d984787b.png
tags: android-app-development, ios, mobile-app-development, android, ios-app-development, ai-tools, base44

---

Base44 packages your app for the stores itself. You scan against Apple and Google guidelines inside the editor, fix what the scan flags in the chat, and generate the files you submit through your own developer accounts. For a lot of apps that is the whole job, and if it covers yours, use it.

Where it stops is documented by Base44 itself. [Their support docs](https://docs.base44.com/documentation/building-your-app/uploading-to-app-stores) describe the output as a lightweight native wrapper that opens only your app's URL, and list push notifications, full offline mode and HealthKit as not supported yet. Those are worth knowing before you create bundle IDs and store records, not after.

## What the built-in route does not cover

Push is the one that hurts most, because sending a notification is the reason most people wanted an app in the first place. Offline is next: with no connection the app shows a network error, which for a field-work or travel product is the entire use case gone. HealthKit rules out fitness and health apps outright. Biometric login and native billing are not on offer either, and billing is not optional if you sell subscriptions.

What the built-in route does give you, and this part is genuinely good, is that content and design changes reach the installed app with no new store version. That property is not unique to it, and it is the reason this whole category exists.

## Offline needs a service worker at your site root

A service worker has to be served as a real file at the root of your site over HTTPS. Registration from a `blob:` URL does not work, because it requires an http or https script URL, and a worker bundled like any other module never resolves at `/sw.js`. On Base44 you put it in the public folder, which maps to the root, and registration succeeds.

That gets you half of it. The native side has to permit the worker to run, which in Despia is Offline Support set to PWA, followed by a version bump and a rebuild. It is a native setting, so it only exists in builds cut after you flipped it. Turn it on, keep testing yesterday's binary, and registration is ignored while the Editor still shows the setting as on, which is the most common false alarm here.

Write the worker network-first with cache write-back rather than cache-first, and change the cache name on every release. A worker that caches once and then always answers from cache pins every user to the version they first opened, so your over-the-air updates appear to stop arriving, and a stale document pointing at asset filenames that no longer exist opens to a white screen.

```javascript
self.addEventListener('fetch', event => {
  if (event.request.method !== 'GET') return

  event.respondWith(
    fetch(event.request)
      .then(response => {
        const copy = response.clone()
        caches.open(CACHE).then(cache => cache.put(event.request, copy))
        return response
      })
      .catch(() =>
        caches.match(event.request).then(hit => hit || caches.match('/offline.html'))
      )
  )
})
```

Prove the worker before you blame the runtime. Open your production URL in Safari or Chrome on a real phone, add it to the home screen, launch it and wait about 10 seconds for the worker to install and fill its caches, then turn off Wi-Fi and cellular separately in system settings rather than using airplane mode, which on some devices leaves Wi-Fi on and passes the test for the wrong reason. If you still get a browser-level connection error, the worker is not serving cached content and no native setting will fix that.

If offline is a hard requirement rather than a nice-to-have, there is a stronger option. Despia's local server downloads your web build to the device on first launch and serves it from an on-device HTTP server at `http://localhost`, so the app boots with no network latency and keeps working indefinitely on the last version it cached, with no worker involved. Most apps do not need it.

## The three routes out

| Route | Push | Offline | Billing | Second codebase |
| --- | --- | --- | --- | --- |
| Base44 built-in packaging | No | No | No | No |
| React Native rebuild | Yes | Yes | Yes | Yes, permanently |
| Despia | Yes | Yes, with a service worker | Yes | No |

Use the built-in packaging when you need a listing and your app is a straightforward web experience. Nobody should pay for a runtime to solve a problem their builder already solves.

Rebuild in React Native if your product is animation-heavy, gesture-heavy or 3D. Your entities and functions stay where they are either way, so you are rewriting the frontend and keeping the backend, permanently maintained alongside whatever web version you keep.

Despia keeps the Base44 app as the source of truth and adds the capabilities the built-in wrapper lists as unsupported, through 50+ native device features reachable from your own JavaScript with a single function. One codebase, over-the-air web updates with no resubmission, and full Xcode and Android Studio projects that export whenever you want them.

## A WebView is not the compromise you have been told it is

Search this and you will find pages arguing that wrapped apps are second-rate, mostly run by tools selling a React Native rebuild. Check it instead of arguing about it.

Start with the app you are using. Open Base44 on an iPhone, tap 3 or 4 times quickly on an ordinary part of the interface, hold the last tap and drag.

![](https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/66e25d90-a8c1-4c0f-871c-b2a5d944ffe5.jpg align="center")

The text-selection magnifier appears over the greeting heading. That heading is not a text field and not an input, it is ordinary interface chrome. Native labels are not selectable, so pressing and holding one does nothing at all. Web text is selectable by default, which is why the loupe shows up. The test is iOS only, since it is a WebKit behaviour and Android's WebView does not produce the same magnifier.

Run it on the Lovable app and the same thing happens over the header.

![](https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/befd82e0-4924-4d8e-af49-d3128f9fea0c.jpg align="center")

Run it on Canva and it happens over a tile label on the home screen.

![](https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/9eab47ca-6276-416b-b4fd-e89719c85a34.jpg align="center")

Canva does not leave it to inference either. [Its technical requirements](https://www.canva.com/help/technical-requirements/) list Android System WebView 89 or higher as a requirement of the app, and Chrome does not use System WebView, so that line is about the app rather than about browsing Canva in a browser.

So both AI builders people use, and one of the highest-rated design apps on either store, render their own mobile interfaces as web content. [Shopify's engineering team](https://shopify.engineering/mobilebridge-native-webviews) published the same decision in April 2025, describing WebViews as central to their mobile strategy across roughly 600 screens while keeping native and React Native for the handful of features that need it.

None of those companies is short of engineers. The claim that native UI is a prerequisite for a successful app does not survive contact with the apps people actually use. Where the web platform genuinely loses is custom gesture recognizers, frame-by-frame game loops and heavy 3D, and if that is your product you should rebuild it.

## What Apple actually rejects

Not web apps. Apps that offer a user nothing they could not get by opening the same URL in Safari.

[Guideline 4.2](https://developer.apple.com/app-store/review/guidelines/) exists for binaries that load a website and stop there, which is close to a description of a wrapper that, in Base44's own words, opens only your app's URL. That does not mean the built-in route gets rejected automatically. Plenty of apps clear review because the product inside is substantial. It does mean you have nothing to point at if a reviewer pushes back, and at that point the fix is not a settings change.

Two or three real native behaviours on paths a reviewer will walk is the difference: haptics on your primary action, push notifications, biometric login, the native camera, native billing.

## Adding capability without leaving Base44

Base44 has no terminal, so you add packages by asking the Builder AI. Before you do, connect the Despia MCP so it writes real calls instead of inventing them. That is your profile, then Settings, then MCP connections, then Add custom MCP, with `https://setup.despia.com/mcp` as the URL and Authentication left as Not required. It is added once per account, not once per app. Ask the Builder to list the haptic schemes afterwards and you should get 5 real ones back.

Then have it install `despia-native`. Every call is gated on the environment check, because the same code still runs in a browser.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

export function saveRecord(record) {
  persist(record)
  if (isDespia) despia('successhaptic://')
}
```

Push needs one more line and a stable id, and the id is simpler than the internet suggests. Base44's built-in user id from `base44.auth.me()` works fine, both as the push external user ID and as the RevenueCat app user id. You do not need a custom auth layer to send notifications or to take payments.

```javascript
if (isDespia && userId) despia(`setonesignalplayerid://?user_id=${userId}`)
```

Custom auth becomes necessary for exactly one thing: native Google sign-in, because Google blocks its consent screen inside an embedded WebView, so sign-in has to run in the system browser and the token has to travel back. That is real work and it is unrelated to push and billing.

## Payments, which is where submissions actually die

Base44 Payments runs through Wix, with Stripe alongside, and Base44's [own store documentation](https://docs.base44.com/documentation/building-your-app/uploading-to-app-stores) describes built-in StoreKit and Play Billing as in progress. So there is no store billing to switch on inside Base44 today.

A Stripe or Base44 Payments checkout for a digital subscription inside the app is a Guideline 3.1.1 rejection, and it is the most common one this audience hits. RevenueCat covers App Store and Play billing through the same calls, using your Base44 user id as the external id. Physical goods and real-world services stay fine on Stripe. The rule is about digital content.

## The things that silently break

Connect a custom domain before you build anything. The URL is compiled into the binary and is the one thing an over-the-air update cannot change, so shipping on a Base44 subdomain and moving later means a new binary and a new review.

A reviewer who cannot sign in rejects the app as incomplete, so put working credentials in App Review Information, and use a password account rather than a magic link they cannot receive.

Generic purpose strings get auto-rejected under Guideline 5.1.1. Write what the permission is for.

And the Android back button should go back rather than close the app. It is a router problem, and nobody catches it before submitting.

## The 14-day Google Play wait

If your Play account is a personal account created after 13 November 2023, you need [12 testers opted in for 14 continuous days](https://support.google.com/googleplay/android-developer/answer/14151465) before you can apply for production. Internal testing does not count and the requirement is per app.

Friends are the free route if you can reach 12. Beyond that, [Testers Community](https://www.testerscommunity.com) runs a free app where developers test each other's projects to earn credits, and sells a done-for-you version from around $15. We have no affiliation and get nothing if you use it. Judge any service on whether written feedback comes back, because Google's production application asks what feedback you received and what you changed because of it.

Ship the AAB to closed testing on day 1 to start the clock rather than waiting it out, then keep improving the web app, which reaches testers with no new build. Apple has no equivalent rule, so submit there in parallel.

## What you end up with

The Base44 app you already built, listed on both stores under your own developer accounts, with push, billing and the rest of the native surface reachable from the project you are already working in. Content changes reach installed apps on next launch. Only the native layer needs a new binary.

## Get it on the stores

Take the app you just built and ship it to iOS and Android without a CLI or a Mac. Code signing and submission run from the browser.

[See the setup docs at setup.despia.com](https://setup.despia.com)