---
title: "App Works on iPhone but Not iPad? Here Is the Fix"
seoTitle: "App Works on iPhone but Not iPad? Here Is the Fix"
seoDescription: "Your app works on iPhone but not iPad because of one substring in your user agent check. Here is the exact line that breaks it and how to fix it."
datePublished: 2026-08-07T10:14:42.460Z
cuid: cmsisg00d00010ai53sc5a1at
slug: app-works-on-iphone-but-not-ipad-here-is-the-fix
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/7126b4f5-1646-4206-806b-88c5ac4d3257.png
tags: ios, ios-app-development, ios-development, iosdevx

---

Apple reviews on iPad. That is where most builders find out that half their app is gated behind a device name. The paywall opens on iPhone, the reviewer taps the same button on an iPad, and gets a screen telling them to open the app on their phone. Same binary, same build, same account. The difference is one substring check written months earlier by someone (or something) that assumed "iPhone" meant "phone".

This post is about that class of bug: platform checks that pass on the device you tested and fail on the device Apple tests.

## The line that does it

Somewhere in the codebase there is a check like this:

```javascript
const isNativeApp = navigator.userAgent.includes('iPhone')
```

On an iPhone it returns true and the native purchase flow runs. On an iPad it returns false, so the app decides it is running in a desktop web browser and renders the fallback: a message asking the user to open the mobile app, or an "Open in App" button that goes nowhere because they are already in the app.

Nothing is wrong with the build. Nothing is wrong with the store configuration. The app told itself it was not the app.

## Why the check exists in the first place

The pattern is legitimate. A web app that ships to the App Store and to the web needs two payment paths, because Apple requires its own billing for digital goods inside the app while the browser version can charge through a web checkout. So the code branches: native purchase inside the app, web checkout outside it.

The branch is correct. The condition is wrong. The question the code needs to answer is "am I running inside the native runtime", and instead it asks "is this device an iPhone". Those were the same question for exactly one device family, and iPad is the counterexample sitting on the reviewer's desk.

This is also why AI-generated code produces it so reliably. Almost every user agent snippet in public training data is browser sniffing: mobile versus desktop, phone versus tablet layout, `/iPhone|Android/i` regexes from a decade of responsive design work. Ask an agent to detect a mobile app and it reaches for the pattern it has seen ten thousand times. The output is idiomatic, readable, and answers a different question than the one you asked.

## The correct check

The Despia runtime injects `despia` into the user agent string alongside the platform identifier. So the environment check is the one substring that is true on every device the app ships to:

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')
```

iPhone, iPad, Android, all true. Safari, Chrome, desktop, all false. When you need the platform as well, ask for it separately and include both iOS device names:

```javascript
const ua = navigator.userAgent.toLowerCase()
const isDespia = ua.includes('despia')
const isIOS = isDespia && (ua.includes('iphone') || ua.includes('ipad'))
const isAndroid = isDespia && ua.includes('android')
```

Under server-side rendering, `navigator` does not exist during the render pass, so guard it or read it inside an effect:

```javascript
const isDespia = typeof navigator !== 'undefined' &&
    navigator.userAgent.toLowerCase().includes('despia')
```

Full reference: [User Agent](https://setup.despia.com/native-features/user-agent).

## The paywall, written correctly

Here is the same branch with the condition fixed, using the current RevenueCat scheme:

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

function openPaywall(userId) {
    if (isDespia) {
        despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)
        return
    }

    window.location.href = `https://pay.rev.cat/<token>/${userId}`
}

// assigned outside the gate, fires when the native sheet completes
window.onRevenueCatPurchase = () => {
    refreshEntitlements()
}
```

Two details worth keeping. The `despia()` call sits inside the gate, because it only means something inside the runtime. The global callback is assigned outside it, because assigning a property on `window` in a browser costs nothing and keeping it unconditional avoids a class of ordering bug. The full scheme list, including direct purchase and the Customer Center, is in the [RevenueCat reference](https://setup.despia.com/native-features/revenuecat/reference).

## The rest of the family

Once you find one of these, grep for the others. They fail the same way and for the same reason.

**Width checks standing in for platform.** `window.innerWidth < 768` returns false on an iPad in full screen, so anything gated behind it disappears. It also returns true on an iPad in Split View, so a phone-only layout shows up on a tablet. Screen width describes a viewport, not a runtime.

**Touch detection standing in for mobile.** `'ontouchstart' in window` is true on touchscreen laptops and false in some remote testing setups. It tells you about the input device and nothing about where your code is running.

`/Mobi/` **regexes.** Common, and iPad Safari does not include `Mobi` in desktop-class mode.

**An** `isMobile` **flag that ends up meaning three different things.** Layout in one component, native feature gating in another, analytics in a third. Once one boolean carries all three meanings, fixing the paywall breaks the tablet layout. Split them.

**iPad Safari reporting itself as a Mac.** Since iPadOS 13, Safari on iPad requests desktop sites by default and its user agent can read as Macintosh with no `iPad` token in it at all. This is the deeper reason not to key native behaviour off device names: the device names are not stable. `despia` is, because the runtime puts it there.

## How to find every instance in ten minutes

1.  Search the codebase, case-insensitively, for `iPhone`, `Android`, `iPad`, `Mobi`, `isMobile`, `isNative`, `innerWidth`, and `ontouchstart`.
    
2.  For each hit, ask what the code is actually deciding. Layout decisions can keep using width. Anything that decides whether to call a native feature, show a purchase button, or render a "download the app" prompt must use the `despia` check.
    
3.  Replace those conditions with the shared `isDespia` value. Export it from one module so there is one definition, not eight.
    
4.  Rebuild and open the app on an iPad, or on the iPad simulator. Walk the purchase path, the login path, and anything that shows a "get the app" prompt. Those three cover most rejections.
    

Step four is the one people skip. If your listing supports iPad, Apple will test on iPad, and a purchase path that dead-ends there is a rejection under [minimum functionality](https://developer.apple.com/app-store/review/guidelines/#minimum-functionality). Shipping iPhone-only is the alternative, but it is a device family setting on the binary and it costs you the tablet audience permanently. Fixing the substring is cheaper.

## Common questions

**Does the iPad need a separate build?** No. Despia ships a universal binary. The same JavaScript runs on both, which is why a device-name check is the only thing that can make them behave differently.

**Why did this pass on my iPhone during testing?** Because your check happened to be true there. Testing on the device the condition was written for confirms the condition, not the behaviour.

**Apple only rejected the paywall. Is anything else affected?** Probably. Anything gated behind the same flag is also off on iPad: push prompts, biometrics, camera access, deep links. The reviewer only reported the first dead end they hit.

**Can I check** `window.despia` **instead?** No. The supported detection is the user agent substring. Anything else is an invented API and will break.

**My AI assistant wrote the original check. How do I stop it doing this again?** Give it the docs page for the exact API before it writes, and ask it what token it is matching on and why. If the answer is a device name, the answer is wrong.

## Add this to your app

Every native capability in this post is one JavaScript call away, inside your existing web codebase. No Xcode, no native project to maintain.

[Read the full reference at setup.despia.com](https://setup.despia.com)