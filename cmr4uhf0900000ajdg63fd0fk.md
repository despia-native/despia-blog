---
title: "Update Your App Without App Store Resubmission"
seoTitle: "Update Your App Without App Store Resubmission"
seoDescription: "Some app changes ship over the air. Others need an App Store resubmission. Learn how to update your app without resubmitting, on iOS and Android."
datePublished: 2026-07-03T11:23:18.983Z
cuid: cmr4uhf0900000ajdg63fd0fk
slug: update-your-app-without-app-store-resubmission
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/abb7367f-f250-4a9c-b338-59301f46c459.jpg
tags: webview, webapp, pwa, appreview

---

You changed your App Start URL and now you are stuck on the question every hybrid builder hits eventually: do I have to rebuild and resubmit to both stores, or does this just ship? The answer is not hand-waving. There is a clean line, and once you see it you will know for every change you make on iOS and Android.

Here is the short version. Most changes to a Despia app update over the air with no App Store or Google Play review. A small set require a new binary and a full resubmission. The App Start URL is one of the few that does. The rest of this post is the exact rule that tells the two apart.

## What ships over the air and what needs a resubmission

Every change to a Despia app is one of two things: a web layer change or a native binary change.

Web layer changes ship over the air. Your JavaScript, your UI, your API calls, your routing, your styling, your business logic. Deploy it and users get it on next launch. No rebuild, no review, no waiting on a reviewer in either store.

Native binary changes require a new build submitted to Apple and Google. New permissions, new entitlements, new native SDKs, and anything compiled into the container itself.

The test is a single question: does this change need a new permission, a new entitlement, or a new native capability baked into the binary? If yes, it is a binary change and needs a resubmission. If no, it ships OTA.

## Why does changing the App Start URL need a resubmission?

The App Start URL is the first thing the native container loads when the app launches. It is not fetched from your web layer at runtime, it is compiled into the binary as the entry point. Changing it changes the binary itself, which puts it firmly on the native side of the line.

That means the same process on both platforms:

1.  Change the App Start URL in Despia.
    
2.  Increment the bundle version in Despia > Settings > Versioning.
    
3.  Rebuild for iOS and Android.
    
4.  Submit both builds for review.
    

Your existing listing, ratings, reviews, and metadata are untouched. Only the binary is replaced. Users on the current version keep running it until the new build is approved and released, then they get it as a normal app update.

## Which changes need a new App Store and Google Play build?

| Change | Where it lives | How it ships |
| --- | --- | --- |
| Layout, styling, copy, components | Web layer | OTA |
| New API call or data transformation | Web layer | OTA |
| Web-based auth flow | Web layer | OTA |
| Adding PostHog, Segment, Mixpanel | Web layer | OTA |
| A feature using an already-declared permission (camera, location, contacts) | Web layer | OTA |
| **App Start URL** | **Native binary** | **App store submission** |
| New permission | Native binary | App store submission |
| New native SDK (payments, biometrics) | Native binary | App store submission |
| Changing entitlements | Native binary | App store submission |

One thing that trips people up: using a native capability is not the same as adding one. Despia declares a broad set of permissions in every binary by default, including camera, microphone, contacts, and location. If your new screen uses the camera and the camera is already declared, that screen ships OTA. You only need a new binary when you add a native integration that was not there before.

## Keep OTA updates within App Store and Play Store rules

Both stores permit web-layer and interpreted-code updates when they stay within the reviewed app's purpose and add no undisclosed native capabilities. On Apple's side the reviewer-facing rule is Guideline 2.5.2, and the underlying contract clause is section 3.3.1(B) of the Developer Program License Agreement, the one long-time developers still call 3.3.2. It allows interpreted code, meaning your JavaScript running in the system WebView, to be downloaded and executed, as long as it does not change the app's primary purpose or add functionality inconsistent with what you submitted. Google Play takes the same position: updating WebView content is expected, provided the app keeps delivering the experience it was reviewed for and stays above the minimum-functionality bar Google tightened for hybrid apps in 2026.

So the line is not "web changes are risky." It is "do not change what the app fundamentally is." Bug fixes, copy, layout, new screens within your app's stated purpose, and gradual rollouts are all fine and expected. Turning a reviewed app into a different product is what the primary-purpose clause exists to stop, and Apple enforces it hard.

Three rules keep you clear of a rejection.

**Do not ship "coming soon" placeholders.** A locked feature with an unlock date, a greyed-out section, or a beta badge on unreleased functionality is direct evidence to a reviewer that you withheld a feature to bolt on later. Apple treats hidden or undisclosed features under Guideline 2.3.1, which is an account-level problem, not just a rejected build. If a feature is not ready, do not show it. Ship it fully via OTA when it is, and gate the transition with VersionGuard so reviewers on the latest binary always land on the live feature.

**Be conservative with a full redesign.** An overhaul that makes the app look and behave like a different product can attract review questions even when it is technically web-layer only. If a change substantially alters the experience, ship it as a binary so the version bump is honest and the intent is unambiguous.

**Declare tracking before you add it.** Adding PostHog, Segment, or Mixpanel via OTA is fine on its own. The caveat is Apple's App Tracking Transparency framework: if the tool shares data with advertisers, the consent prompt must be declared in the binary. Add the ATT declaration upfront and store consent server-side, so a later OTA deployment can check it before it initialises anything.

## Ship the binary before the web layer that depends on it

If a change spans both sides, meaning new native capability plus web code that calls it, the order is not optional.

1.  Make the native change in Despia.
    
2.  Increment the bundle version.
    
3.  Submit to both stores.
    
4.  Wait for approval.
    
5.  Deploy the web layer via OTA.
    
6.  Gate the new code with VersionGuard.
    

Deploy step 5 before step 4 finishes and users on the old binary hit web code calling a native API their device does not have yet. That is the most common self-inflicted bug in hybrid deployment.

## Gate the transition with VersionGuard

After any binary update there is a window where some users have the new build and some do not. VersionGuard reads `window.bundleNumber` from the runtime, which is available synchronously before your app starts, so you can gate on it with no loading state and no flash of wrong content.

A guard renders its children only when the version is in range and renders nothing otherwise. To handle the transition, pair a `min_version` guard for the new path with a `max_version` guard for the old one. The bounds are inclusive.

```jsx
import { VersionGuard } from 'despia-version-guard';

// New flow: only users on the 2.1.0 binary or later
<VersionGuard min_version="2.1.0">
  <NewFeature />
</VersionGuard>

// Old flow: anyone still on the previous binary
<VersionGuard max_version="2.0.0">
  <LegacyFlow />
</VersionGuard>
```

Each user sees exactly one path, and users on an older binary never hit code that calls a native API their device does not have. Reviewers always run the latest binary, so they land on the new path every time. Once the old version is deprecated enough, drop the second guard.

## Test before it reaches users

Whether the change went out OTA or as a binary, the verification is the same. Deploy a visible change, confirm it is live in a browser, force close the native app completely rather than backgrounding it, wait a couple of seconds, reopen, and confirm the change appears. If it does not, check for service worker caching, purge your CDN cache, and make sure the app was fully closed.

Display a build timestamp or version hash somewhere in your UI during development. It removes all guesswork about which version is running in the container.

## Common questions

### Can you update an app without resubmitting to the App Store?

Yes, for web layer changes. UI, logic, content, and any feature that uses an already-declared permission ship over the air with no App Store or Google Play review. You only resubmit when a change touches the native binary.

### Does changing the App Start URL require a new build?

Yes. The App Start URL is compiled into the native binary as the launch entry point, so changing it requires a rebuild and a resubmission to both the App Store and Google Play. OTA does not apply to it.

### Do OTA updates need App Store review?

No, as long as they stay within the reviewed app's purpose and add no undisclosed native capabilities. Both Apple and Google permit web-layer and interpreted-code updates on that basis, and they reach users on next launch with no review step. Native changes, and any change to what the app fundamentally is, still go through normal store review.

### Will Apple or Google reject an OTA update?

Not for normal web-layer work. Apple's Guideline 2.5.2 and License Agreement section 3.3.1(B) allow interpreted code to update in the WebView as long as it does not change the app's primary purpose, and Google Play expects WebView content to update. You get into trouble by using OTA to change what the app fundamentally is, or by shipping "coming soon" placeholders that signal a withheld feature.

### Is the process different on iOS and Android?

The rule is identical on both. Web layer changes ship OTA on both platforms, and native changes require a resubmission on both. Android review is usually faster than iOS, but which changes need a build does not differ.

## Get it on the stores

Take the app you already built and ship it to iOS and Android from one codebase, with native device access and over-the-air updates for everything that lives in the web layer. Code signing and submission run from the browser, no CLI and no Mac required.

[See the setup docs at setup.despia.com](https://setup.despia.com)