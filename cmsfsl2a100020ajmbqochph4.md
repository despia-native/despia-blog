---
title: "Base44 Google Play Closed Testing: 12 Testers, 14 Days"
seoTitle: "Base44 Google Play Closed Testing: 12 Testers, 14 Days"
seoDescription: "Base44 apps need 12 testers for 14 days before Google Play production access. What actually gets checked, where to find testers, and what to ship."
datePublished: 2026-08-05T07:55:20.145Z
cuid: cmsfsl2a100020ajmbqochph4
slug: base44-google-play-closed-testing-12-testers-14-days
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/90d801d1-868a-4202-8441-a5eeea57f109.png
tags: android-app-development, android, android-apps, googleplay

---

You prompted an app into existence in Base44, paid the $25 developer fee, and then Google Play told you that you cannot publish until 12 testers have used it for 14 continuous days. Most guides stop at explaining the rule. The rule is the easy part. What fails apps in 2026 is what happens inside those 14 days, and what you uploaded before the clock started.

## Who this rule applies to

If your personal Google Play developer account was created after November 13, 2023, you have to run a closed test with at least 12 testers opted in for at least the last 14 consecutive days before you can apply for production access. The requirement was 20 testers until December 2024, when Google cut it to 12. Any tutorial still saying 20 is out of date.

Organization accounts verified with a D-U-N-S number are exempt. That sounds like an escape hatch, and sometimes it is, but organization verification takes weeks and needs real business documentation. For most solo builders, running the 14 days is faster than becoming a company.

Two things people get wrong immediately:

*   **Internal testing does not count.** You can add 100 people to an internal test and be no closer to production. The gate lives on the closed testing track.
    
*   **The clock does not start when you upload.** Your closed testing release goes through review first, usually around a day, longer on a new account. The 14 days start once the release is live and you actually have 12 testers opted in.
    

None of this cares how the app was built. Play Console sees a signed Android App Bundle. Whether you prompted it in Base44 or wrote it by hand, it goes through the same closed test.

## The count is not the requirement, the usage is

Twelve opt-ins is the number Google publishes. It is not the thing that gets your production request rejected.

Google reviews your production access request against what the testers did. Installed and never opened does not read as testing. Opted in on day one and forgotten does not read as testing. Since spring 2026, production requests get turned down for insufficient engagement even when the tester count was correct the entire time, and unresolved crashes and ANRs in your pre-launch report make that worse.

So the target is not 12 people who click a link. It is 12 people who open the app every day or two, move through real screens, and hit real features, for two straight weeks.

One more piece of insurance: recruit more than 12. Google's wording is continuous across the last 14 days, and if someone opts out and drops you to 11, you are relying on a reading of the policy that not everyone agrees on. Fifteen to twenty testers costs you nothing extra and removes the argument.

## Where the 12 testers come from

Three honest options.

**People you know.** Free, and fine if you genuinely have a dozen friends on real Android devices who will open your app daily for two weeks. Most people overestimate this. The failure mode is not recruitment, it is week two, when everyone stops opening it and you cannot chase them without becoming a nuisance.

**A tester community.** [Testers Community](https://www.testerscommunity.com/) runs a [free companion app](https://play.google.com/store/apps/details?id=com.testerscommunity) with a pool of developers who test each other's apps. You earn credits by testing other people's builds and spend them on your own. It costs time instead of money, and the reciprocity is what keeps people opening the app.

**A paid testing service.** The same team runs a paid track: 15 testers on the Starter plan at $15 per app, 25 testers on the Pro plan at $25 with an ASO report. Testers are assigned within about six hours, the test runs 16 days rather than 14 so you have a two-day buffer before you submit, and you get a feedback report plus draft answers for the production access form. They back it with a refund if production access is denied. Their [closed testing walkthrough](https://www.testerscommunity.com/google-play-closed-testing) is worth reading before you set up the track, and they have a separate page for [production access rejections](https://www.testerscommunity.com/google-play-production-access-rejected) if you have already been turned down once.

Whichever route you pick, the testers are only half of it. Real people using a thin app still produces a thin test.

## The second gate nobody plans for

Closed testing is a distribution gate. It is not the only thing standing between you and a live listing.

Google's Spam and Minimum Functionality policy sits underneath everything. Apps that repackage a website with no app-specific value are treated as low quality, and the webview and affiliate spam rules add that you need to own or control the site you are wrapping. Both are enforced in the same review that decides whether your app stays up. If you are shipping the same build to iOS, Apple's guideline 4.2 is the harsher version of the same test.

This is where Base44 apps get caught, because the default output is a web app and Base44's own [documentation on uploading to app stores](https://docs.base44.com/documentation/building-your-app/uploading-to-app-stores) states that their mobile output is a web view without push notifications and without full offline support. That is their statement, not a competitor's characterisation.

The order this bites people is predictable. You spend 14 days getting testers through the closed test, apply for production, and only then find out the binary was never going to survive on its own merits. You burned two weeks proving that 12 people opened a browser.

Fix the functionality question before you start the clock, not after.

## Ship a binary worth testing

The bar is not high, but it is real. Your app needs behaviour a browser tab cannot produce. Push notifications are the single most useful one, because reaching users between sessions is usually the actual reason to have an app at all, and it directly improves tester engagement during the 14 days. After that: biometric unlock, haptics on real interactions, native share, offline behaviour that survives losing signal, and native billing if you sell anything digital.

That does not mean rebuilding the app. In [Despia](https://setup.despia.com/introduction) it is the same Base44 codebase with native calls added behind a runtime check:

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

if (isDespia) {
  // map the device to your user so push can target them
  despia(`setonesignalplayerid://?user_id=${userId}`)
  despia('successhaptic://')
}
```

Sensitive values can sit behind Face ID or Touch ID without a native project:

```javascript
await despia('setvault://?key=sessionToken&value=abc123&locked=true')
const token = await despia('readvault://?key=sessionToken', ['sessionToken'])
```

If you sell subscriptions or digital goods, Google requires Play billing rather than your web checkout. For Base44 apps the [in-app purchases without webhooks setup](https://blog.despia.com/base44-in-app-purchases-without-webhooks-revenuecat) is the correct path, and the [RevenueCat reference](https://setup.despia.com/native-features/revenuecat/reference) has the current call shapes.

## Use the 14 days instead of surviving them

Here is the part almost nobody plans for. Testers find bugs on day three. In a normal Android workflow, fixing that bug means a new AAB, a new closed testing release, another review, and a group of testers who now have a week of muscle memory around a broken screen.

With remote hydration, the binary loads your hosted Base44 build on launch. Deploy a fix and installed apps pick it up on the next open, with no new AAB and no review. Only changes to native configuration require a new build. Two weeks of daily feedback becomes two weeks of iteration, which is what the production access questionnaire is asking about anyway.

## The production access questionnaire

After 14 days an option to apply for production appears. Google asks about ten questions, including how you recruited testers, what feedback you got, and what you changed because of it. Vague answers read as a test that did not happen. Specific ones do not.

Keep notes during the two weeks. Which testers reported what, which builds shipped, which bugs closed. That log is the answer to half the questions, and it is also the reason a service that hands you a written feedback report is worth more than a group chat.

## Common questions

**Does the 14 days restart if a tester opts out?** Google's requirement is 12 testers opted in continuously across the last 14 days, and guidance in the wild disagrees on whether a brief dip resets it. Do not find out. Start with 15 to 25 testers.

**Do emulators or second accounts count?** No. Testers need real devices and genuine Google accounts. Attempts to fake the count are the fastest way to lose the developer account, not just the app.

**Can I keep prompting in Base44 during closed testing?** Yes, and you should. Uploading a new AAB does not reset the 14 days, and if your web content is served over the air, most fixes do not need an AAB at all.

**Does my Base44 backend need to change?** No. Entities, records and functions stay on Base44 and behave identically from a native client. It is an HTTPS API either way.

**Does this apply to the App Store too?** No. Apple has no equivalent tester requirement, it has TestFlight and a human review. The Base44-specific rejection reasons are in the [Base44 rejection checklist](https://blog.despia.com/base44-app-store-approval-the-rejection-checklist).

**Does my app need to be finished before I start?** It needs to be stable and genuinely usable, not complete. The 14-day clock cannot start until you upload something, so upload the first honest version and keep shipping.

## Get it on the stores

Take the Base44 app you already built, give it push, biometrics, offline behaviour, and native billing, and ship it to Google Play and the App Store without a Mac or a CLI. Web updates reach installed apps over the air, so the two weeks of closed testing are spent fixing things rather than waiting on review.

[See the setup docs at setup.despia.com](https://setup.despia.com)