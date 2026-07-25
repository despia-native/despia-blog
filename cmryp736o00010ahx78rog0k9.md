---
title: "Guideline 3.1.2 EULA Rejection: The Fast Fix"
seoTitle: "Guideline 3.1.2 EULA Rejection: The Fast Fix"
seoDescription: "Apple's Guideline 3.1.2 EULA rejection hits nearly every subscription app. Here is the exact fix, and how to ship it without a full resubmission."
datePublished: 2026-07-24T08:48:24.293Z
cuid: cmryp736o00010ahx78rog0k9
slug: guideline-3-1-2-eula-rejection-the-fast-fix
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/55d898c9-34c7-4b5f-bfbb-5a087511c24a.png
tags: ios, ios-app-developer, ios-app-development, appstore, eula, appreview

---

Almost every subscription app meets Guideline 3.1.2(c) at least once. The rejection cites required subscription information, and the piece that is actually missing is the Terms of Use (EULA) link. It does not matter whether you built in Lovable, Base44, or your own stack. Apple checks the same things, and the fix is the same. What differs is how long it takes to ship, and that is the part worth getting right.

## The five things Apple checks in the app

For an auto-renewable subscription, the app itself must clearly show:

*   The subscription title
    
*   The length of the subscription
    
*   The price, and the price per unit if relevant
    
*   A functional privacy policy link
    
*   A functional Terms of Use (EULA) link
    

And App Store Connect metadata needs the privacy policy link plus the EULA link. A privacy policy alone does not satisfy 3.1.2. You need the EULA too, and that is the one people miss.

## The EULA link

If you do not have a custom EULA, Apple's standard one is accepted:

`https://www.apple.com/legal/internet-services/itunes/dev/stdeula/`

Put it in both places Apple verifies:

1.  App Store Connect: your App Description, or a custom EULA in the EULA field under App Information.
    
2.  In the app: on the paywall and on the account screen, next to the privacy policy.
    

Apple provides its standard EULA by default unless you add a custom one, but the reliable move is to place it in both spots. That is what clears the common 3.1.2(c) rejection.

## Show the terms, do not just price a button

On the paywall, render the title, length, and price as readable text. Apple checks that the terms are visible before purchase, and a price on a buy button does not count. Add a line stating the subscription renews automatically and can be cancelled in App Store account settings:

```javascript
// Paywall disclosure, shown above the purchase button
const disclosure =
  'Premium renews automatically at the price shown unless cancelled ' +
  'at least 24 hours before the period ends. Manage in App Store settings.'
```

## The speed difference

The App Store Connect description change never needs a build. The in-app links are the slow part. On a rebuild-per-change setup, a one-line EULA link means a new binary and another review cycle for what is genuinely a two-minute edit.

This is where an over-the-air web layer changes the economics. With Despia serving your app, the EULA and privacy links on your paywall ship as a web update. The reviewer sees the fix without a new binary, you reply confirming it, and the item closes the same day. The EULA rejection stops being a lost review cycle and becomes a quick correction.

## Step by step

1.  Add the standard EULA link to your App Description in App Store Connect.
    
2.  Add the EULA and privacy links to the paywall and account screen.
    
3.  Confirm the paywall shows title, length, and price as text.
    
4.  Push the web change over the air, or rebuild if your links live in a native shell.
    
5.  Reply to the reviewer confirming the EULA link is now in the description and the app.
    

## Try Despia

Ship your subscription app to iOS and Android from a single web codebase, and push paywall, EULA, and privacy fixes over the air the moment a reviewer asks for them.

[Learn more at setup.despia.com](https://setup.despia.com)