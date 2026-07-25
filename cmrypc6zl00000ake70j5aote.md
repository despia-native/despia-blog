---
title: "Lovable EULA Rejection? Fix Guideline 3.1.2"
seoTitle: "Lovable EULA Rejection? Fix Guideline 3.1.2"
seoDescription: "Apple rejected your Lovable subscription under Guideline 3.1.2 for a missing Terms of Use link. Here is the exact fix and where the EULA has to go."
datePublished: 2026-07-24T08:52:22.513Z
cuid: cmrypc6zl00000ake70j5aote
slug: lovable-eula-rejection-fix-guideline-3-1-2
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/21e10a75-2f14-42a1-bebd-997b121e4ae7.png
tags: ios, ios-app-developer, ios-app-development, eula, lovable, appreview

---

Your Lovable app has a paywall, Apple rejected it under Guideline 3.1.2(c), and the message talks about required subscription information. The missing piece is almost always the Terms of Use (EULA) link. It takes a few minutes to fix once you know the five things Apple checks and the two places the link belongs.

## What Guideline 3.1.2 requires

An auto-renewable subscription has to show, inside the app: the subscription title, its length, the price, a working privacy policy link, and a working Terms of Use (EULA) link. In App Store Connect, the metadata also needs a privacy policy link and a EULA link. A privacy policy on its own does not satisfy 3.1.2, which is a common source of confusion. You need both.

## The EULA link

Use Apple's standard EULA if you do not have a custom one:

`https://www.apple.com/legal/internet-services/itunes/dev/stdeula/`

Then place it in both spots Apple checks:

1.  App Store Connect: add it to your App Description, or add a custom EULA in the EULA field under App Information.
    
2.  In the app: on your paywall and on your account screen, next to the privacy policy link.
    

On the paywall, show the title, length, and price as text. A price sitting on the buy button is not enough. Apple wants the terms readable before the tap.

## The Capacitor rebuild trap

This is where the fix gets slow on a plain Lovable build. Lovable exports through Capacitor, so a change to an in-app link means a full native rebuild, a new binary, and another trip through the review queue. For a one-line EULA link, that is a disproportionate amount of work, and it is why this rejection sometimes eats a second full review cycle.

Running the same Lovable app through Despia removes that. The web layer updates over the air, so adding the EULA and privacy links to your paywall reaches the reviewer without a new binary. Only the App Store Connect description edit is needed on Apple's side, and that never required a build to begin with.

```javascript
// Links rendered in the paywall footer, before the purchase button
const legalLinks = {
    privacy: 'https://yourapp.com/privacy',
    eula:    'https://www.apple.com/legal/internet-services/itunes/dev/stdeula/',
}
```

## Confirm and resubmit

Once the links are in the description and on the paywall and account screen, reply to the reviewer confirming it. If the links live only in your UI and you are on plain Capacitor, rebuild and resubmit. On Despia, push the web change and reply, no rebuild.

## The checklist

*   Standard EULA link in the App Description
    
*   EULA and privacy links on the paywall and account screen
    
*   Title, length, and price shown as text
    
*   Auto-renew and cancellation wording near the button
    

## Get it on the stores

Connect Despia to your Lovable app and ship to iOS and Android from one codebase, with paywall and EULA fixes that go out over the air instead of through a full rebuild.

[See the setup docs at setup.despia.com](https://setup.despia.com)