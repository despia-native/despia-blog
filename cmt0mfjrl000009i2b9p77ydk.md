---
title: "Base44 Mobile App Limitations Worth Knowing First"
seoTitle: "Base44 Mobile App Limitations Worth Knowing First"
seoDescription: "The documented constraints of the Base44 mobile app, from native features to billing, permissions and the fixed bundle ID, and what each one costs."
datePublished: 2026-08-19T21:46:14.893Z
cuid: cmt0mfjrl000009i2b9p77ydk
slug: base44-mobile-app-limitations-worth-knowing-first
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/521622ae-c471-4f66-8820-397836abc2cc.png
tags: ai, android-app-development, ios, android, ios-app-developer, ios-app-development, ai-tools, appstore, aitools, googleplay, base44, appreview

---

Base44 documents the constraints of its mobile app openly. The problem is that they sit across a submission guide, an FAQ accordion and a troubleshooting section, so nobody reads them in one sitting and most people meet them at submission time. Here they are together.

None of what follows is a defect. It is the shape of a version-one wrapper inside a much larger product, and Base44's documentation says the platform continues to expand native feature coverage. The useful question is not whether a constraint will lift eventually. It is whether it is lifted on the day you need to ship.

## What native features does the Base44 mobile app support?

**A web view around your published app, plus push notifications. That is the whole list. Base44's own FAQ states it directly and names full offline mode and HealthKit as unsupported.**

The FAQ adds that some native capabilities may need additional review by Apple or Google depending on which permissions your app requests.

That coverage line is the most consequential sentence in the documentation, because guideline 4.2 asks for features, content and UI that elevate an app beyond a repackaged website, and Apple's standard 4.2 rejection letter names push notifications specifically as an addition that does not carry an app on its own. An app whose entire native surface is push is exactly the case that letter was written for.

That does not mean push is worthless to a submission. It means it is one input among several, and it is the input Apple has already said does not carry an app on its own. Three further guidelines constrain what you can do with it: 4.5.4 says push must not be required for the app to function, 5.1.2(i) prohibits requiring users to enable push in order to access functionality or content, and 4.10 prohibits monetizing built-in capabilities and names push explicitly. An app whose value proposition is the notification has the dependency backwards.

## Can you sell subscriptions through a Base44 mobile app?

**Not compliantly today. Base44's documentation states that Stripe must not be used for digital goods inside the mobile app, and that an app using Stripe for digital content is rejected.**

A StoreKit and Play Billing integration is described as in progress. Both stores require their own billing systems for digital content, so this is not a Base44 rule, it is the platform rule every app has to meet. Our [guide to selling subscriptions from a Base44 app](/blog/base44-subscriptions) covers the compliant setup in full.

Read carefully, that line is not a warning about a risky pattern. It is a description of a closed path. If your product sells subscriptions, in-app features, credits or any other digital content, the built-in wrapper has no route to the stores until the native billing integration ships.

The distinction that matters: physical goods and services consumed outside the app are fine on Stripe, and guideline 3.1.3(e) actually requires a payment method other than in-app purchase for them. It is digital content specifically that must go through StoreKit and Play Billing.

## Can you change the bundle ID or signing key?

**No. Base44's troubleshooting documentation states that both are configured automatically and cannot be changed inside the generated IPA or AAB files.**

If they do not match a version you uploaded previously, the stores block the update, and the documented remedy is to create a new app entry rather than to fix the mismatch. Three consequences follow, in ascending order of expense:

1.  The package name follows the documented `com.base[app-id].app` format, which is permanent and publicly visible in the Play Store URL
    
2.  An app already on the stores from another toolchain cannot be updated with a Base44 build
    
3.  Because the signing key is not yours, the identity of the app is not portable if you leave
    

None of that bites on day one. All of it bites at exactly the moment you least want a new store listing.

## Can you choose which URL the app opens?

**No. Base44 documents that the main entry URL is selected automatically from your published app, and that a different start page cannot currently be chosen for the app.**

For most apps this is invisible, because the published app is what you want to ship. It matters in two situations.

If you want the mobile app to open on a mobile-specific route rather than your web landing page, you cannot. And because the binary is bound to a Base44-controlled address rather than one you choose, moving off the platform later means the installed app no longer points at your product.

Owning a custom domain does not remove the constraint, but it is the thing that makes it survivable, which is why it is the first recommendation in the ownership guide.

## Can you control the permissions your app requests?

**No. Base44 documents that every mobile app package includes device-level permissions, that an AI scan sets them, and that they are not editable in the interface.**

This is worth understanding because permissions are a rejection surface of their own. Guideline 5.1.1(iii) asks for data minimization, meaning apps should request access only to data relevant to core functionality, and reviewers do ask why a permission is present. If a permission is declared that your app does not visibly use, the answer has to come from your privacy policy and store listing rather than from a settings toggle.

There is a documented instance of exactly this: the iOS build has included a HealthKit entitlement without the matching `NSHealthShareUsageDescription` key, which surfaces as an App Store Connect rejection for apps that never touch health data. Base44 notes that HealthKit is not supported and that no action is required from developers whose apps do not use it.

## What operational limits show up during submission?

**Three, all documented. Downloading store files requires the Builder plan or higher, push notifications do not reach existing installs until you ship an update, and every iOS build consumes an Apple distribution certificate slot.**

That last one catches teams that iterate. Apple allows up to 3 active production distribution certificates on the standard Apple Developer Program, and Base44 documents that generating an IPA creates one each time. The fourth generation returns an ALREADY\_EXISTS error, and the fix is revoking a certificate you no longer need in your Apple Developer account.

Worth planning around rather than discovering: IPA generation is not a free action you can repeat casually while testing.

## Who handles the store review conversation?

**You do. Base44 documents that its support does not track submission status, contact Apple or Google on your behalf, or manage review feedback, and that it cannot guarantee approval.**

That is an honest boundary and every platform is entitled to draw one. It is worth reading before you submit, because the moment a rejection letter arrives is the moment people go looking for help, and this determines where that help comes from.

If your team has never handled an App Review rejection, the difference between a vendor that draws this line and one that treats submissions as its core business is the difference between a two-day fix and a three-week loop.

## Which of these does a specialist runtime change?

**Most of them, though it moves them rather than removes them. Apple and Google still own the store rules. What changes is which side of the constraint you sit on.**

| Constraint | Base44 mobile app | Despia |
| --- | --- | --- |
| Native features | Web view plus push notifications | Over 50 capabilities via one JavaScript function |
| Digital purchases | No compliant path yet | RevenueCat paywalls, purchases and Customer Center |
| Bundle ID | Generated, not changeable | Chosen by you at project creation |
| Entry URL | Selected automatically | Any URL you control |
| Permissions | Set by AI scan, not editable | Declared per app |
| Offline | Not supported | Your own service worker, enabled by the runtime |
| HealthKit | Not supported | HealthKit and Health Connect |
| Native project export | Not offered | Full Xcode and Android Studio projects |
| Store review help | Out of scope for support | Core to the business |

The last row is the hardest to put in a table and the easiest to feel. Despia's runtime started as Advanced WebView in 2011 and carried over 7,500 apps before becoming Despia in 2023, which spans the introduction of guideline 4.2, several rewrites of the review rules, App Tracking Transparency and privacy manifests. Store submission is not a feature sitting next to the editor and the AI. It is the whole job.

## When do none of these limitations matter?

**When your app sells nothing digital, needs no offline capability, requests no unusual permissions, and will never move off Base44.**

Add one more condition: the app is substantial enough on its own that push is a nice addition rather than the entire native story.

That describes a real and reasonable category of app, and for it the built-in path is genuinely fewer moving parts. One vendor, one dashboard, one bill. The reason to read this list is not that the constraints are unreasonable. It is that four of them are permanent decisions disguised as defaults, and the cheapest time to think about them is before the first submission rather than after.

## When do you need more than the store files?

**When purchases, offline, biometrics, persistent storage or a bundle ID you chose are core to the product rather than optional.**

Base44 can scan your app against store guidelines, generate the IPA and the AAB, and add push notifications, and for apps that need nothing else that may be everything. Despia is what you reach for when the list above is not optional, from the same Base44 codebase you already have.

[Learn more about Despia in the docs](https://setup.despia.com)