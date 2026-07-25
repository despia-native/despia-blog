---
title: "Web App EULA Rejection: Fix Guideline 3.1.2"
seoTitle: "Web App EULA Rejection: Fix Guideline 3.1.2"
seoDescription: "Apple rejected your web app subscription under Guideline 3.1.2 for a missing EULA link. Here is what to add, where it goes, and how to confirm it."
datePublished: 2026-07-24T08:53:55.662Z
cuid: cmrype6v500000akld5g45b9v
slug: web-app-eula-rejection-fix-guideline-3-1-2
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/9e610779-f353-4b6b-9541-b92710ba9007.png
tags: ios-apps, ios, ios-app-developer, ios-app-development, eula, appreview

---

You wrapped a web app for the App Store, it sells a subscription, and Apple bounced it under Guideline 3.1.2(c). The rejection lists required subscription information, and the item that is actually missing is nearly always the Terms of Use (EULA) link. This is a metadata and disclosure fix, not a code rewrite.

## What Apple checks for a subscription

Inside the app, an auto-renewable subscription must show the title, the length, the price, a working privacy policy link, and a working Terms of Use (EULA) link. In App Store Connect, the metadata needs the privacy policy link and the EULA link as well. Having a privacy policy is not enough on its own. Guideline 3.1.2 wants both documents linked.

## Add the EULA link

If you do not have a custom EULA, Apple's standard one is accepted:

`https://www.apple.com/legal/internet-services/itunes/dev/stdeula/`

Place it in both locations Apple verifies:

1.  App Store Connect: in your App Description, or as a custom EULA in the EULA field under App Information.
    
2.  In the app: on the paywall and on the account screen, beside your privacy policy link.
    

## Show the terms as text, not a button

On the paywall, display the title, length, and price as readable text. A reviewer will reject a paywall that shows only a price on a purchase button, because the terms are not disclosed before the purchase. Add a short line stating the subscription renews automatically and can be cancelled in App Store account settings. That single sentence is what most often flips a paywall from rejected to review-complete.

```javascript
// Rendered above the purchase button
const paywallTerms =
  'Pro renews automatically at the listed price unless cancelled ' +
  'at least 24 hours before the term ends. Cancel anytime in your ' +
  'App Store account settings.'
```

## Push the fix without a full resubmission

The App Store Connect description edit never needs a build. The in-app links usually do, unless your web layer updates over the air. If it does, adding the EULA and privacy links to the paywall reaches the reviewer without a new binary, which turns a multi-day resubmission into a same-day fix.

Once the links are in the description and in the app, reply to the reviewer confirming the change and the item closes.

## The checklist

*   Standard EULA link in the App Description
    
*   EULA and privacy links on the paywall and account screen
    
*   Title, length, and price shown as text
    
*   Auto-renew and cancellation wording near the button
    

## Get it on the stores

Take the web app you already have and ship it to iOS and Android with subscriptions handled correctly, and push EULA and paywall fixes over the air when a reviewer asks.

[See the setup docs at setup.despia.com](https://setup.despia.com)