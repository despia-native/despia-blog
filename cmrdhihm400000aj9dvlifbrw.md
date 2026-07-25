---
title: "Fix Guideline 3.1.2 Subscription EULA Rejections"
seoTitle: "Fix Guideline 3.1.2 Subscription EULA Rejections"
seoDescription: "Apple rejects auto-renewable subscriptions under Guideline 3.1.2 for a missing Terms of Use (EULA) link. Here is exactly what to add and where."
datePublished: 2026-07-09T12:30:09.594Z
cuid: cmrdhihm400000aj9dvlifbrw
slug: fix-guideline-3-1-2-subscription-eula-rejections
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/9d6b9901-f689-40d9-a4a1-98a60686b3b6.png
tags: ios, ios-app-development, appstore, eula

---

Your auto-renewable subscription gets rejected under Guideline 3.1.2(c) for missing required information, usually a Terms of Use (EULA) link. The fix takes a few minutes once you know the five things Apple checks for and where the EULA link needs to appear.

## What Apple requires for auto-renewable subscriptions

Inside the app, the subscription must clearly show:

*   The subscription title
    
*   The length of the subscription
    
*   The price, and the price per unit if relevant
    
*   A functional link to your privacy policy
    
*   A functional link to the Terms of Use (EULA)
    

In App Store Connect metadata, you also need:

*   A privacy policy link in the Privacy Policy field
    
*   A Terms of Use (EULA) link in the App Description, or a custom EULA in the EULA field
    

The one that trips almost everyone is the EULA link.

## The EULA link

If you do not have a custom EULA, use Apple's standard one:

https://www.apple.com/legal/internet-services/itunes/dev/stdeula/

Apple provides its standard EULA by default unless you add a custom one in the EULA field under App Information. In practice, put it in both places: in App Store Connect metadata and inside the app near the paywall. That is what avoids the common 3.1.2(c) rejection.

1.  App Store Connect: add it to your App Description, or add a custom EULA in the EULA field under App Information.
    
2.  Inside the app: a visible link on your paywall and on your login or account screen, next to your privacy policy.
    

## Show the subscription terms in the app

On the paywall, display the title, length, and price as readable text, not just a buy button. Apple checks that the terms are visible before the user purchases. A price on a button alone does not satisfy this. State that the subscription renews automatically and can be cancelled in the App Store account settings, which makes the paywall read as review-complete.

## Step by step

1.  Add the standard EULA link to your App Description in App Store Connect.
    
2.  Add the EULA and privacy policy links to your paywall and account screen in the app.
    
3.  Confirm the paywall shows the title, length, and price as text.
    
4.  If the links live in your app UI, rebuild and resubmit. For a WebView app you can push the link changes over the air if your tool supports it.
    
5.  Reply to the reviewer confirming the EULA link is now in the description and in the app.
    

## FAQ

**Can I use Apple's standard EULA or do I need a lawyer?** Apple's standard EULA link is accepted. Use a custom one only if your terms genuinely require it.

**Where exactly does the link need to be in the app?** Apple wants it reachable near the point of purchase and in account settings. The paywall plus the profile or login screen covers it.

**I already have a privacy policy. Why am I still rejected?** Guideline 3.1.2 needs both the privacy policy and the Terms of Use (EULA). A privacy policy alone is not enough for subscriptions.

**Do I need a new build just to add the links?** If the links live in your app UI, yes, unless your tool supports over-the-air web updates. Despia does, so you can push the link changes without a new binary. The App Store Connect description change on its own never needs a build.

## Get it on the stores

Ship your subscription app to iOS and Android from your existing web codebase, and push paywall and link fixes over the air when a reviewer asks for them.

[See the setup docs at setup.despia.com](https://setup.despia.com)