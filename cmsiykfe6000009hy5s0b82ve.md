---
title: "Convert Base44 to a Mobile App: The Full Guide"
seoTitle: "Convert Base44 to a Mobile App: The Full Guide"
seoDescription: "Base44 builds web apps, not native ones. The full path to convert Base44 to a mobile app on both stores, from the project you already have."
datePublished: 2026-08-07T13:06:06.728Z
cuid: cmsiykfe6000009hy5s0b82ve
slug: convert-base44-to-a-mobile-app-the-full-guide
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/0cc45e37-d6a8-4c0e-a9fb-0bb2d6b3fc1e.png
tags: android-app-development, ios, android, ios-app-developer, ios-app-development, android-apps, ai-development-services, ai-tools, ai-agents, base44

---

Base44 will generate store files for you. What it generates is a secure WebView wrapper that opens only your app's URL, which is enough for some apps and nowhere near enough for others. The real decision is what you do when it is not enough.

**Short answer.** Base44's built-in mobile app is a WebView wrapper with no native push, no full offline mode, no HealthKit and no built-in StoreKit or Play Billing yet. If your app needs none of those, use it. If it does, you have two options: regenerate your frontend as a React Native project, which leaves you maintaining two codebases, or keep Base44 as the source of truth and run that frontend inside a native runtime, which ships a signed binary with those capabilities added underneath and pushes web changes over the air with no resubmission.

## What Base44 actually gives you

A Vite React frontend and a hosted backend: entities, auth, Deno functions, file storage, realtime, reached through `@base44/sdk`.

Two things worth knowing before you build on top of it. The code export is frontend only, so entities and backend logic stay on Base44's servers and the exported `.env` still points back at them. And the CLI (`npx base44`) has an `eject` command and no mobile build target.

Store files are a different matter, and this is where most posts on this topic are out of date. Base44's Mobile app tab scans your app against Apple and Google guidelines and generates what you submit, including a Google Play ready AAB. Per [their documentation](https://docs.base44.com/documentation/building-your-app/uploading-to-app-stores), that build runs your published app inside a secure web view that opens only your app's URL, and it does not currently support native-only features such as push notifications or full offline mode. HealthKit and built-in billing are not there yet either. In exchange, content and design changes you publish in Base44 appear in the app without a new store version.

So if your app takes no payments, sends no notifications and touches no hardware, use it. It is already in your dashboard and it is the right tool for that app.

## The three routes

**Base44's built-in mobile app.** Generates the store files, including a Play ready AAB, and updates content over the air. A WebView wrapper, so no native push, no full offline, no HealthKit, no built-in billing. Right choice for a simple app.

**Rebuild in React Native.** An agent regenerates your frontend as native views. Real native rendering, and a second codebase: your Base44 app still exists, you are still prompting it, and the generated project never hears about it. Right call if rendering is your product, or if you have a developer who wants to own a native codebase.

**Run the app you have in a native runtime.** Base44 stays the source of truth. A native engine underneath holds the SDKs, so web code calls APNs and FCM push, StoreKit and Play Billing, Face ID, background GPS and the rest. One codebase, a signed binary on both stores, web changes over the air. That is Despia, and the rest of this walks it end to end.

## Step 1: Publish the Base44 app

The runtime loads your published URL, so it has to be publicly reachable. Base44 subdomain or custom domain, either is fine.

A login wall is fine too, since that is your app behaving normally. What matters is that the URL resolves, and that Apple gets a working demo account at submission. Reviewers who cannot get past your login screen reject the build.

## Step 2: Install the SDK through the Builder chat

Base44 has no terminal. Ask the Builder AI to install `despia-native`, approve the prompt, then publish so the change takes effect. Packages persist once installed, and an unused one calls nothing by itself, so there is nothing to undo if you change direction.

## Step 3: Detect the runtime

Everything native goes behind one check.

```javascript
const ua = navigator.userAgent.toLowerCase()

const isDespia = ua.includes('despia')
const isDespiaIOS = isDespia && (ua.includes('iphone') || ua.includes('ipad'))
const isDespiaAndroid = isDespia && ua.includes('android')
```

Put it in one shared helper. Do not check for `iPhone` alone, do not use `innerWidth`, do not invent a `window.despia` flag. iPadOS Safari reports a Macintosh user agent with no iPad token, and an iPhone-only check is the most common reason a Base44 app works on a phone and shows a broken paywall on an iPad.

## Step 4: Add the features that make it an app

This decides whether you get approved and whether anyone keeps the app.

If you do not write code, you do not have to type any of this. Paste it into the Builder chat, or connect the Despia MCP server at `https://setup.despia.com/mcp` and describe what you want. The code is here so you can check what you got back, which is a different skill from writing it and a more useful one.

Native calls go inside the gate. Callbacks the runtime invokes are assigned outside it, at the top level, or they vanish when a component unmounts.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// the bridge has no guaranteed callback, so a read must never block the UI
const capped = (p, ms = 2000) =>
  Promise.race([p, new Promise(r => setTimeout(() => r(null), ms))])

async function onAuthenticatedLoad() {
  const user = await base44.auth.me()
  if (!isDespia) return

  despia(`setonesignalplayerid://?user_id=${user.id}`)   // link device to user

  const push = await capped(despia('checkNativePushPermissions://', ['nativePushEnabled']))
  if (push && !push.nativePushEnabled) despia('settingsapp://')
}
```

Digital goods have to go through the stores. The paywall is one call, with a web fallback for desktop:

```javascript
if (isDespia) {
  despia(`revenuecat://launchPaywall?external_id=${user.id}&offering=default`)
} else {
  window.location.href = `https://pay.rev.cat/<your_token>/${encodeURIComponent(user.id)}`
}

window.onRevenueCatPurchase = checkEntitlements

async function checkEntitlements() {
  const data = await capped(despia('getpurchasehistory://', ['restoredData']))
  const active = (data?.restoredData ?? []).filter(p => p.isActive)
  if (active.some(p => p.entitlementId === 'premium')) unlockPremium()
}
```

Full setup for each: [the RevenueCat guide](https://blog.despia.com/base44-in-app-purchases-without-webhooks-revenuecat) and [the Base44 push guide](https://blog.despia.com/base44-push-notifications-with-onesignal-native).

Deep links need no client code. Send a `path` in the notification's `data` object and the runtime updates the URL and fires `popstate`, so your router navigates on its own. Add `window.onNotificationEvent` only if you also send `metadata` to restore state, and do not navigate from it as well or you navigate twice.

## The rule to apply to every awaited call

Notice the two second cap above, because it is the one production rule to learn before anything else.

The bridge has no guaranteed callback. If the native side does not answer, because the feature is off, the binary is older than the code, or you are in a web preview, an awaited call can sit for 15 to 30 seconds. On screen that is a frozen app, and a reviewer who hits it files a rejection rather than a bug report.

Cap every awaited read and let the work finish in the background. Fire-and-forget calls like haptics, the paywall launch and `setonesignalplayerid://` need none of this.

## If you need native Google or Apple sign-in

Base44's built-in Google login is browser-based. Ship it to a phone and tapping Sign in with Google opens a web page inside your own app asking for a password. It works, it looks wrong, and it is the moment your product stops feeling like an app.

Only the first of three problems here is cosmetic. Users expect the system sign-in sheet. Apple expects an equivalent privacy-focused option next to any third-party login, which means Sign in with Apple. And web storage clears on delete, so a reinstall signs everyone out. Two related review requirements catch people at the same time: no signup wall in front of content that does not need an account, and account deletion inside the app rather than by support email.

**This is the one part you should not hand to an agent.** Native sign-in means the system browser handles consent, the provider redirects to a page you host, and a single-use code comes back over your deep link. Your backend exchanges that code server-side with a secret that must never reach the device, checks the returned `id_token` audience against your own client id, mints your session token, and re-verifies it on every request against the live user record rather than trusting the token. Then most of it again for Apple.

Every one of those steps has a version that appears to work and is quietly insecure: an unchecked token audience, a secret that reaches the client, a role read from the token instead of the record. The broken version and the correct version both log you in, so you find out later.

Which is why we wrote it and open sourced it:

```bash
git clone https://github.com/despia-native/base44-native-oauth
```

It is a store-review-ready starter, not an auth snippet. Native Google Sign-In and Sign in with Apple, guest device accounts so nobody is forced to register before seeing the product, in-app account deletion, deny-all row-level security on every entity, push and RevenueCat wired in. That is not a feature list picked for a README. It is the rejection list, closed. The session lives in the Storage Vault, so it survives a reinstall.

Three things to change and nothing else is hardcoded: your deep-link scheme in `src/config/app-config.js`, your secrets in the Base44 dashboard under Settings, Environment Variables (`JWT_SECRET` long, random and stable, since changing it signs everyone out, plus your Google client id and secret, `RESEND_API_KEY` and `RESEND_FROM`, and `APP_BASE_URL`), and the Google redirect URI, which is `APP_BASE_URL` plus `/native-callback.html`.

The clone is the one terminal moment in this whole guide. Hand the repository URL to your AI agent, or borrow a developer for an hour. After that you are back in dashboards.

One gap to close yourself: the auth endpoints ship without app-level rate limiting, so add per-IP or per-email throttling before you open signups publicly. And if you are only ever shipping on the web, skip all of this. Base44's built-in login is simpler and perfectly fine.

## Step 5: Get the store accounts

Apple Developer Program, $99 a year. Individual approval is usually 24 to 48 hours, organization enrollment around two weeks because of the D-U-N-S check, so start it early. Google Play Console, $25 once.

Both in your own name. Apple's guideline 4.2.6 requires apps built with an app generation service to be submitted by the provider of the content, which means your account, not a vendor's.

## Step 6: Configure and build

Set the app name, bundle ID, icon, splash and addons in the Despia editor, then build. Signing, provisioning and the store upload run from the browser, so there is no Mac, no Xcode and no CLI.

One rule causes more confusion than anything else: web changes ship over the air, native configuration does not. Saving your OneSignal App ID or RevenueCat keys does nothing until you rebuild, and the failure is silent. `setonesignalplayerid://` resolves normally, backend sends return success from the API and never reach the device, entitlement checks come back empty. If either breaks right after a settings change, rebuild before debugging anything.

## Step 7: Test on a real device

A screen rendering proves nothing about whether a native call reached the operating system. Test push on a physical device, test a purchase with a sandbox account, test on an iPad if your listing supports iPad, and open your web app in a desktop browser to confirm every native call is gated and nothing throws outside the runtime.

## Step 8: Submit

iOS goes through App Store Connect: metadata, screenshots from a phone rather than a desktop browser, purpose strings describing your actual use, App Privacy answers, and a demo account. Review is typically 24 to 48 hours.

Android has an extra gate. New personal Play Console accounts need 12 testers opted in for 14 consecutive days on a closed track before you can request production access, and since 2026 Google also rejects requests for insufficient engagement even when the count is right. Start that clock early.

## The things that go wrong

**Placeholder purpose strings.** `NSCameraUsageDescription` and friends still saying what the template shipped with. A 5.1.1 rejection, handed out before anyone evaluates your app. [The fix](https://blog.despia.com/base44-app-permissions-fix-the-5-1-1-rejection).

**Testing yesterday's build.** Enable an addon, test the binary you already have, see nothing, conclude it is broken. Entitlements compile in. Rebuild first.

**Hallucinated APIs.** Your agent writes a real method from RevenueCat's native SDK into a scheme-based runtime. The name is real and the shape is idiomatic, so review passes it, and the first evidence is a button that does nothing on a device. Ask for the documentation URL for that exact symbol and open it.

**A frozen screen on an awaited call.** The one that becomes a rejection rather than a bug report, because a reviewer cannot tell a hung bridge from a broken app.

**Shipping thin.** A binary that only loads your URL fails guideline 4.2 on the merits. [The checklist](https://blog.despia.com/base44-app-store-approval-the-rejection-checklist).

## How the routes compare

|  | Base44 built-in | React Native rebuild | Native runtime |
| --- | --- | --- | --- |
| Codebases | One | Two | One |
| Push notifications | No | Yes | Yes, APNs and FCM |
| In-app purchases | No | Yes | Yes, StoreKit and Play Billing |
| Device access | No | Full | 50+ native features |
| Web updates | Over the air | Rebuild, or JS-only OTA | Over the air, no review |
| Annual OS requirements | Their problem | Yours | Handled upstream |
| Needs a developer | No | Eventually | No |
| Cost | Free | Subscription plus maintenance | $249 per app per OS, one time |

Where a native runtime is not the answer: a game loop, real-time 3D, AR, per-frame camera processing, Live Activities, or a watch companion. And if deep offline is core to your product rather than a nice-to-have, weigh that honestly before you start.

## Common questions

**Does Base44 build native mobile apps?** It generates store files, including a Play ready AAB, but what runs inside them is a secure web view of your published app. Its own docs list the gaps: no native push, no full offline mode. HealthKit and built-in billing are not there yet either.

**Can I convert a Base44 app to iOS only?** Yes. Licensing is per app per OS. Most people ship both, because the same web codebase already serves both.

**Will Apple reject my app for using a WebView?** Not for the WebView. Guideline 4.2 asks whether your app does enough beyond a repackaged website. Push, purchases, biometrics and deep links answer that.

**Do I keep my Base44 backend?** Yes, unchanged. Your native app calls entities, auth and functions over HTTPS exactly as the web app does.

**How long does it take?** A build on your own phone is an afternoon. Apple review is a day or two. Google's 14-day testing window is the long pole, so recruit testers first, not last.

**Do I need a Mac or Xcode?** No. You need your own Apple and Google accounts, because the app ships under your name.

**Does Base44's Google login work in the native app?** As a web sign-in flow, not the system sheet, and with no Sign in with Apple. Native login means owning your auth, which is what the boilerplate above is for.

**What if I need something Despia does not support?** First ask whether you need it. Most requests have a web-first version that is good enough, or turn out to be a feature the product does not need. That is the cheapest answer and it is right more often than people expect.

If you do need it, two paths, neither instant. Request it at features@despia.com, where it goes on the roadmap and a human builds it, with no guaranteed date. Or export the project and write the Swift or Kotlin yourself, which is immediate, but is code, and is not the easy option.

## Get it on the stores

Take the Base44 app you already built and ship it to iOS and Android without a CLI or a Mac. Code signing and submission run from the browser.

[See the setup docs at setup.despia.com](https://setup.despia.com) or [start building at despia.com](https://despia.com).