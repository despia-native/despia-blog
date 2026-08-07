---
title: "Convert a Base44 App to iOS and Publish It"
seoTitle: "Convert a Base44 App to iOS and Publish It"
seoDescription: "How to convert a Base44 app to an iOS app and publish it to the App Store: the build, the guidelines that cause rejections, and the fixes."
datePublished: 2026-08-07T16:36:04.645Z
cuid: cmsj62g0h00000ajidm21c2eu
slug: convert-a-base44-app-to-ios-and-publish-it
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/1e201083-fef7-4475-95e7-98a29a14ff1e.png
tags: ios, ios-app-developer, mobile-app, ios-app-development, appstore, base44

---

Apple does not care that your app was built in Base44. It cares whether the binary does something Safari does not, whether the listing is honest, and whether a reviewer can get inside it. Almost every rejection a Base44 builder gets is one of those three.

**Short answer.** Base44 can generate store files for the App Store, but what it generates is a secure WebView wrapper that opens only your app's URL, with no native push, no full offline mode, no HealthKit and no built-in StoreKit yet. If you need those while keeping Base44 as the source of truth, converting means running the same Base44 frontend inside a native runtime that ships a signed binary with the capabilities added underneath. Either way you need an Apple Developer Program membership in your own name at $99 a year, a bundle ID with the right capabilities, a listing with device screenshots and accurate purpose strings, App Privacy answers that match reality, and a working demo account. Review is usually 24 to 48 hours.

This guide covers both halves: the conversion, then the submission.

## What Base44 gives you and what it does not

Base44 does generate store files. From the Mobile app tab you can scan your app against Apple's guidelines and produce the files you submit from your own Apple developer account. So the question is not whether you can get a build. It is what is inside it.

Per [Base44's own documentation](https://docs.base44.com/documentation/building-your-app/uploading-to-app-stores), that build runs your published app inside a secure web view, a lightweight native wrapper that opens only your app's URL, and it does not currently support native-only features such as push notifications or full offline mode. HealthKit and built-in StoreKit are not there yet either. The upside their docs also name is real: publish a content or design change in Base44 and it appears in the app without a new store version.

If your app takes no payments, sends no notifications and touches no hardware, use that and stop reading. It is free and already in your dashboard.

Everything below is for the other case: a real signed binary with native capability, built from the same Base44 project through a native runtime, which is what Despia does.

**What converting actually means here.** Nothing is regenerated and nothing is rewritten. Your Base44 project stays exactly where it is and keeps being the thing you edit. The runtime ships it as a signed iOS binary and hands the web layer the device, so push, biometrics and purchases become available from the code you already have. You end up with one codebase, not two, and web changes keep going out over the air with no resubmission.

The alternative, having an AI regenerate your frontend as a React Native project, is a real option and it is also a second codebase to maintain alongside Base44. That trade is covered in [the full conversion guide](https://blog.despia.com/convert-base44-to-mobile-app).

## Enrollment, and why to start it today

$99 a year, the same price as an individual or an organization.

Individual enrollment usually clears in 24 to 48 hours. Organization enrollment can take around two weeks, because Apple requires a D-U-N-S number and cross-checks it against your form and your company website, sometimes by phone. D-U-N-S registration itself is free and takes one to five business days.

It must be your account. Guideline 4.2.6 rejects apps built with a commercialized template or app generation service unless they are submitted directly by the provider of the app's content, which is a rule about whose developer account ships the app. Any service offering to publish on your behalf creates that problem for you.

## Identifiers, before you build

**Bundle ID.** Reverse-domain format, permanent once an app record exists. Enable the capabilities your app uses. Push Notifications is the one people forget, and registration fails at runtime without it even with a valid key.

**Team ID.** Ten characters, top right of developer.apple.com/account.

**A second bundle ID for push.** OneSignal's notification service extension needs `com.yourcompany.yourapp.OneSignalNotificationServiceExtension`, spelled exactly, with Push and Associated Domains enabled. The runtime provisions that target at build time.

**Three keys that look identical.** All `.p8`, all download once:

| Key | Created under | Goes to |
| --- | --- | --- |
| APNs auth key | Keys, with APNs enabled | OneSignal |
| App Store Connect API key | Users and Access, Integrations | Despia |
| In-App Purchase key | App Store Connect, In-App Purchase | RevenueCat |

Not interchangeable, and the in-app purchase page has its own Issuer ID that differs from the main one. Label them at download or you will be regenerating keys.

You do not generate certificates. The pipeline creates Apple Development and Apple Distribution for you. If you see "Developer ID" anywhere, that is a macOS certificate for distributing outside the Mac App Store and has nothing to do with iOS, which is the top confusion for anyone searching that phrase.

## Make the Base44 app reachable

The runtime loads your published Base44 URL, so it has to resolve publicly. Subdomain or custom domain, either works.

A login wall is fine, because that is your app behaving normally. What is not fine is submitting without a demo account. Put real, working credentials in the App Information notes in App Store Connect. Reviewers who cannot get past your login screen reject the build, and that rejection has nothing to do with how it was built.

## What makes a Base44 app pass 4.2

Guideline 4.2 asks that your app include features, content and UI that elevate it beyond a repackaged website. The subject of that sentence is your features, not your rendering engine. A binary that only loads your Base44 URL fails it correctly.

Push is the strongest single answer, because reaching a user between sessions is visibly not a website behaviour. Base44 has no terminal, so you add the SDK by asking the Builder AI to install `despia-native`, approving the prompt, and publishing.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// the bridge has no guaranteed callback, so a read must never block the UI
const capped = (p, ms = 2000) =>
  Promise.race([p, new Promise(r => setTimeout(() => r(null), ms))])

async function onAuthenticatedLoad() {
  const user = await base44.auth.me()
  if (!isDespia) return

  despia(`setonesignalplayerid://?user_id=${user.id}`)

  const push = await capped(despia('checkNativePushPermissions://', ['nativePushEnabled']))
  if (push && !push.nativePushEnabled) despia('settingsapp://')
}
```

After push: Face ID on sensitive screens, StoreKit purchases, deep links that land on the right screen, camera and photo access used in a real flow. Pick the ones your product uses. Reviewers can tell a feature from a feature bolted on for review.

Two Base44-specific traps here. Never gate on `iPhone` alone, because iPadOS Safari reports a Macintosh user agent with no iPad token, and if your listing supports iPad then Apple reviews on one. And never leave an awaited bridge call uncapped: with no guaranteed callback it can hang for 15 to 30 seconds, and a reviewer cannot tell a hung bridge from a broken app.

Full setup: [the Base44 push guide](https://blog.despia.com/base44-push-notifications-with-onesignal-native).

## Selling anything digital

Digital goods and subscriptions unlocked inside the app go through StoreKit. That is 3.1.1 and no architecture changes it. Physical goods and real-world services do not.

```javascript
if (isDespia) {
  despia(`revenuecat://launchPaywall?external_id=${user.id}&offering=default`)
} else {
  window.location.href = `https://pay.rev.cat/<your_token>/${encodeURIComponent(user.id)}`
}

window.onRevenueCatPurchase = checkEntitlements
```

Apple also requires a restore path, which is the same entitlement check you use to unlock the feature. The Base44 setup, including the version that needs no webhooks, is [here](https://blog.despia.com/base44-in-app-purchases-without-webhooks-revenuecat).

## Login, and the two rules attached to it

Base44's built-in Google login is browser-based. On a phone that opens a web page inside your own app asking for a password, which works and looks wrong. Apple also expects an equivalent privacy-focused option next to any third-party login, meaning Sign in with Apple.

Two related review requirements catch Base44 builders at the same time. Apple rejects apps that put a signup wall in front of content that does not need an account. And any app with accounts must allow deletion inside the app, not by emailing support.

Native login means owning your auth, which is the hardest thing to build correctly on Base44, so start from [the open source boilerplate](https://github.com/despia-native/base44-native-oauth). It ships native Google Sign-In, Sign in with Apple, guest device accounts and in-app deletion, which is that rejection list closed. The [full guide](https://blog.despia.com/convert-base44-to-mobile-app) covers it.

## The listing is part of the review

Screenshots from a device at the required sizes, showing your app rather than your marketing site. iPad screenshots if your listing supports iPad.

The description should describe the app. "Our Base44 site, now as an app" is an argument for rejecting it.

Purpose strings are the quiet killer. Every permission has a string the user sees, and it must say what your app does with that access. A placeholder left from a template is a 5.1.1 rejection, issued before anyone evaluates your features. Write "Used to attach photos to your reports", not "This app requires camera access". [The fix](https://blog.despia.com/base44-app-permissions-fix-the-5-1-1-rejection).

App Privacy answers must cover everything your app and its SDKs collect, including analytics, attribution and push.

## Build, test, submit

In the Despia editor you set name, bundle ID, icon, splash and addons, then build. Signing, provisioning and the App Store Connect upload run from the browser, so there is no Mac and no Xcode.

One rule causes more confusion than the rest: web changes ship over the air, native configuration does not. Saving your OneSignal App ID or RevenueCat keys does nothing until you rebuild, and the failure is silent. Calls resolve normally, backend sends return success from the API and never reach the device, entitlement checks come back empty. If something breaks right after a settings change, rebuild before debugging.

Use TestFlight even if you are alone. Installing your own build surfaces what a browser preview never will. Then test push on a physical device, test a purchase in sandbox, test on an iPad if your listing supports one, and open your Base44 app in a desktop browser to confirm every native call is gated.

Review is usually 24 to 48 hours, longer for a first submission.

## When you get rejected

Read the guideline number, because it tells you which conversation you are in. Guessing costs a cycle.

Reply in Resolution Center rather than resubmitting silently. If the reviewer missed native features because they never created an account or never reached that screen, say where they are and how to get to them. That reply often works.

Fix the metadata in the same pass. A 4.2 rejection usually arrives alongside desktop screenshots and a website-shaped description, and fixing only the app leaves the impression intact.

The full pre-submission list is in [the Base44 rejection checklist](https://blog.despia.com/base44-app-store-approval-the-rejection-checklist).

## After you are live

Base44 changes ship over the air with no resubmission. Edit, publish, users get it on next launch.

Native configuration still needs a build and a review. Apple also runs an annual SDK clock: since 28 April 2026 every upload must be built with Xcode 26 and the iOS 26 SDK, and that moves every year. If the runtime is maintained upstream, that is a rebuild you trigger rather than a project you maintain.

## Common questions

**Can Base44 publish to the App Store on its own?** Yes. It generates the files you submit from your own Apple developer account. What it does not give you, per Base44's own docs, is native push, full offline mode, HealthKit or built-in StoreKit, so the question is whether your app needs any of those.

**Do I need a Mac or Xcode?** No. Builds, signing and submission run from the browser. You need your own Apple Developer account because the app ships under your name.

**Will Apple reject my Base44 app for using a WebView?** Not for the WebView. Guideline 4.2 asks whether the app does enough beyond a repackaged website, which is a capability question.

**Does my Base44 backend change?** No. Entities, auth and functions stay where they are and the iOS app calls them over HTTPS exactly as the web app does.

**How long does it take?** Enrollment 24 to 48 hours as an individual. A build on your own phone the same afternoon. Review usually a day or two.

**Can I ship iOS only?** Yes. Licensing is per app per OS, so iOS alone is a normal choice.

## Get it on the App Store

Take the Base44 app you already built and ship it to iOS with push, biometrics and in-app purchases, from one codebase. Code signing and submission run from the browser.

[See the setup docs at setup.despia.com](https://setup.despia.com)