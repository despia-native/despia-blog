---
title: "Convert a Base44 App to Android and Publish It"
seoTitle: "Convert a Base44 App to Android and Publish It"
seoDescription: "How to convert a Base44 app to an Android app and publish it to Google Play, including the 12-tester rule and the deadlines that block uploads."
datePublished: 2026-08-07T16:33:55.759Z
cuid: cmsj5zokb00000ajbekdc7yef
slug: convert-a-base44-app-to-android-and-publish-it
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/83599564-3a11-4840-bb0d-6f1f60dacc00.png
tags: android-app-development, mobile-app-development, android, google-play, mobile-development, base44

---

Google Play is usually described as the easier store. For a Base44 builder shipping their first app it now takes longer than iOS, and not for technical reasons. A rule for new personal accounts adds a fixed two-week wait that no amount of preparation removes, and almost nobody finds out until they are ready to launch.

**Short answer.** Base44 can generate a Google Play ready AAB from the Mobile app tab, but what it generates is a secure WebView wrapper that opens only your app's URL, with no native push, no full offline mode, no HealthKit and no built-in Play Billing yet. If you need those while keeping Base44 as the source of truth, converting means running the same Base44 frontend inside a native runtime that ships a signed bundle with the capabilities added underneath. Either way a Play Console account is $25 once, you complete a store listing, content rating, target audience and Data safety declarations, and you run a closed test. If your account is a personal one created after 13 November 2023, you need 12 testers opted in for 14 consecutive days before you can even apply for production access.

This guide covers both halves: the conversion, then the submission. Start the tester clock first and finish the app while it runs.

## Start the tester clock before anything else

This is the scheduling insight the whole guide turns on, so it goes first.

You need 12 testers opted in and running a closed test for 14 consecutive days. The count was 20 until December 2024. Internal testing does not count. The clock only starts once your closed testing release has cleared review and 12 people have actually opted in, so the day you upload is not day one.

Since spring 2026 Google also rejects production access for insufficient engagement even when the count is right, so testers who install and never open the app are not enough. Unresolved crashes and ANRs in the pre-launch report trigger rejections too.

Sources disagree on whether dropping below 12 resets the clock. The wording is continuous coverage and the published guidance conflicts, so recruit 15 to 25 and the question never arises. If you do not have that many willing testers, [Testers Community](https://www.testerscommunity.com/) exists for this and runs 16 days rather than 14 for buffer.

So: get any working build of your Base44 app into closed testing, start the clock, and spend the two weeks adding the features below.

## Account setup

$25 once, for the lifetime of the account.

The account type decides whether the tester rule applies to you. Personal accounts created after 13 November 2023 must complete closed testing. Organization accounts verified with a D-U-N-S number are exempt, so if this is a company, register the D-U-N-S and open an organization account rather than starting personal and migrating.

Every account completes identity verification with a name and address that Google displays on your store listing. Worth knowing before you sign up as an individual.

Your package name is permanent once an app exists under it, and it must match what you use in Firebase, because a mismatch there produces push failures that look like anything except a naming problem.

## What Base44 gives you and what it does not

Base44 does generate store files. From the Mobile app tab you can produce a Google Play ready AAB, download it, and upload it to a Play Console release. So the question is not whether you can get a bundle. It is what is inside it.

Per [Base44's own documentation](https://docs.base44.com/documentation/building-your-app/uploading-to-app-stores), that bundle runs your published app inside a secure web view, a lightweight native wrapper that opens only your app's URL, and it does not currently support native-only features such as push notifications or full offline mode. HealthKit and built-in Play Billing are not there yet either. The upside their docs also name is real: publish a content or design change in Base44 and it appears in the app without a new store version.

If your app takes no payments, sends no notifications and touches no hardware, that is free and already in your dashboard. Everything below is for the other case: a signed app bundle with native capability, built from the same Base44 project through a native runtime.

**What converting actually means here.** Nothing is regenerated and nothing is rewritten. Your Base44 project stays exactly where it is and keeps being the thing you edit. The runtime ships it as a signed Android bundle and hands the web layer the device, so push, fingerprint unlock and Play Billing become available from the code you already have. One codebase, not two, and web changes keep going out over the air with no new upload.

The alternative, having an AI regenerate your frontend as a React Native project, is a real option and it is also a second codebase to maintain alongside Base44. That trade is covered in [the full conversion guide](https://blog.despia.com/convert-base44-to-a-mobile-app-the-full-guide).

## Add the features worth testing

Play's relevant policy is Spam and Minimum Functionality, plus a webviews rule that turns on ownership: you must own the site your app loads. That is a lower bar than Apple's, but the app still has to be worth installing, and shipping something worth testing is also what gets you real engagement during the 14 days.

Base44 has no terminal, so ask the Builder AI to install `despia-native`, approve the prompt, and publish.

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

The Android half of push setup is Firebase: an Android app with the same package name, then the Server Key and Sender ID pasted into OneSignal. Full sequence in [the Base44 push guide](https://blog.despia.com/base44-push-notifications-with-onesignal-native).

Never leave an awaited bridge call uncapped. With no guaranteed callback it can hang for 15 to 30 seconds, which reads as a frozen app and shows up in your pre-launch report as an ANR.

## The bundle and signing

Play takes an Android App Bundle, not an APK, for new apps.

Play App Signing holds the release key: you upload with an upload key, Google re-signs with the app signing key it holds, and that key never sits on your laptop waiting to be lost. Losing an upload key is recoverable. Losing a self-managed app signing key is not.

If your build runs in the browser, the bundle is produced and uploaded for you and there is no Android Studio in your path.

## The declarations, and the one people underestimate

Play front-loads paperwork that Apple asks for later.

**Store listing.** Title, short and full description, screenshots from a device at the required sizes, feature graphic, icon.

**Content rating.** A questionnaire generating per-region ratings. Answer honestly, since a wrong answer found later can pull the listing.

**Target audience and content.** Whether the app targets children. If it does, Families policy applies and it is extensive.

**Ads declaration.** Including ads served by any SDK you added.

**Data safety.** The underestimated one. You declare what you collect, what you share, whether it is encrypted in transit, and whether users can request deletion. It must cover everything your app and every SDK inside it collects, including your Base44 backend, analytics, attribution and push. Google compares the declaration to observed behaviour, and a mismatch is a policy violation rather than a form error.

**Privacy policy.** A real, reachable URL that matches those answers.

## Selling anything digital

Digital goods go through Google Play's billing system. On Base44 that is the same RevenueCat call your iOS build uses, with one entitlement across both catalogs:

```javascript
if (isDespia) {
  despia(`revenuecat://launchPaywall?external_id=${user.id}&offering=default`)
} else {
  window.location.href = `https://pay.rev.cat/<your_token>/${encodeURIComponent(user.id)}`
}
```

Three Android differences that cause real bugs. A purchase not acknowledged within three days is automatically refunded by Google and access revoked, silently. Subscriptions nest as product, then base plan, then offer, so your catalog is not a copy of the App Store one. And testing requires an upload to a track signed by Play plus license testers, because a browser preview or an unsigned build never resolves products, which is the usual reason behind "my products are not showing up".

The Base44 setup, including the version that needs no webhooks, is [here](https://blog.despia.com/base44-in-app-purchases-without-webhooks-revenuecat).

Commission rates are in flux following the Epic litigation and Google's external-link programs, so check current terms rather than a number from a blog post.

## Two deadlines that block uploads

Both are enforced at upload, not at runtime, so nothing breaks until the day you try to ship.

From 31 August 2026, new apps and updates must target Android 16, API level 36, with an extension available to 1 November. Apps below the threshold stop being discoverable to new users on newer devices, and this repeats annually.

From the same date, anything selling digital goods must use Play Billing Library 8 or later, same extension. PBL 9 shipped in May 2026.

If you own the native project, both are your job every year. If the runtime is maintained upstream, both are a rebuild you trigger from the browser.

## Native Google Sign-In

Base44's built-in Google login is browser-based. It works, but on a phone users expect the system sheet, and a web sign-in page inside your own app is where your product stops feeling like an app.

Native sign-in means owning your auth: your own `Account` entity, your own JWTs signed in a Base44 backend function, and the system browser handling consent with a single-use code coming back over your deep link. Start from [the open source boilerplate](https://github.com/despia-native/base44-native-oauth), which ships it along with guest device accounts, in-app account deletion and deny-all row-level security. The [full guide](https://blog.despia.com/convert-base44-to-mobile-app) covers it.

## Timeline and rollout

Signup and identity verification: a day or two, longer for organization verification. First closed testing release through review: hours to a few days. Closed testing: 14 consecutive days minimum, realistically 16 to 20 with the opt-in ramp. Production access questionnaire: about ten questions, reviewed in days.

Three to four weeks end to end for a new personal account, and the app is the small part.

Use staged rollout when you get there. A crash that only appears on one manufacturer's Android build reaches 5% of your users instead of all of them, and you can halt a Play rollout, which you cannot do on the App Store.

That matters more on Android than people expect: Samsung, Huawei, Xiaomi and OnePlus apply aggressive background restrictions by default, which is why a hand-rolled background feature works on a Pixel and nowhere else. Despia handles delivery across those manufacturers with no extra configuration, but test on a Samsung before widening a rollout regardless.

## Common questions

**Can Base44 publish to Google Play on its own?** Yes. The Mobile app tab generates a Google Play ready AAB. What it does not give you, per Base44's own docs, is native push, full offline mode, HealthKit or built-in Play Billing, so the question is whether your app needs any of those.

**Do I really need 12 testers?** Only for personal accounts created after 13 November 2023. D-U-N-S-verified organization accounts are exempt, and internal testing does not count.

**Do I need Android Studio?** No. The bundle is built, signed and uploaded from the browser. You need your own Play Console account because the app ships under your name.

**Will Google reject a Base44 app for being web-based?** Not for that. The webview rule turns on whether you own the site being loaded, and minimum functionality asks whether the app is worth installing.

**Does my Base44 backend change?** No. Entities, auth and functions stay where they are and the Android app calls them over HTTPS exactly as the web app does.

**Can I ship Android only?** Yes. Licensing is per app per OS. Most people add iOS because the same Base44 codebase already serves it.

## Get it on Google Play

Take the Base44 app you already built and ship it to Android with push, purchases and device access, from one codebase. Signing and submission run from the browser.

[See the setup docs at setup.despia.com](https://setup.despia.com)