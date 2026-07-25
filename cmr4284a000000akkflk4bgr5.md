---
title: "In-App Purchases Not Submitted for Review: The Fix"
seoTitle: "In-App Purchases Not Submitted for Review: The Fix"
seoDescription: "App Review keeps rejecting your app because the In-App Purchase products have not been submitted for review. Here is why it happens and the exact fix."
datePublished: 2026-07-02T22:12:15.932Z
cuid: cmr4284a000000akkflk4bgr5
slug: in-app-purchases-not-submitted-for-review-the-fix
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/c2847a39-b5f3-4d9f-9472-1e0e7277c008.jpg
tags: webview, iap, appstore, revenuecat

---

You built your subscriptions, wired them up, tested the purchase flow, and the app works. Then App Review sends the build back with this: one or more of the In-App Purchase products have not been submitted for review. The products are sitting right there in App Store Connect, so the rejection reads like a mistake. It is not. It is a one-time step almost every developer misses on their first paid tier, and it takes about two minutes to clear.

## Why the products exist but review can't see them

Creating an In-App Purchase or subscription in App Store Connect does not submit it. Those are two separate actions, and the second one is easy to skip because nothing forces you through it.

For the first In-App Purchase or subscription on an app, and any time you add a new type of product, Apple requires it to be attached to and submitted together with a new app version. If you send the build in without attaching the products, the review team sees your app reference Pro or a subscription in the UI but has no product to actually evaluate against, so it bounces with that exact message. Once your first paid product of a given type is approved, later products of that type can be submitted on their own from the Monetization section. The first one always has to ride along with a version.

That is the whole cause. The products are ready, they are just not connected to the submission.

## The fix, step by step

1.  In App Store Connect, open Apps and select your app.
    
2.  In the left sidebar, click the app version you are submitting.
    
3.  On the right, scroll down to the In-App Purchases and Subscriptions section.
    
4.  Click Select In-App Purchases or Subscriptions (or Edit if you already started), tick every product that ships with this release, and click Done.
    
5.  Submit the version.
    

Now the products go into review alongside the binary, and the rejection clears. You do not rebuild the binary or bump the build number to do this. The build was fine. It was only ever missing the link between the products and the submission. The one caveat: if the version was already approved without the products, that version is closed, so you attach them to the next editable version instead. You still do not need a new build for the sake of the products themselves.

## If the product does not appear in that list

Only products with the Ready to Submit status show up in step 4. If a subscription is still marked Missing Metadata, it will be invisible there, which sends people down a rabbit hole looking for a bug that is not there.

Open the product and complete the three things Apple wants before it will accept it: a price, at least one localization, and the review screenshot. For subscriptions, check the subscription group as well, since the group itself needs a localized display name or the products under it stay stuck. Save, and the product flips to Ready to Submit and shows up in the version's list. This is the second most common reason developers get stuck on this rejection, so check it before assuming anything is broken.

There is one more trap worth knowing. Sometimes every product reads Ready to Submit and the Select or Submit control is still greyed out. The reliable un-stick is to edit the App Store localization slightly to make the form dirty again: open the localization, add a space to a field, save, remove the space, save again. The button comes back. It looks like superstition and it is a known App Store Connect quirk.

## A note on Apple's changing submission flow

Apple is rolling out a revised submission experience through 2026 that folds In-App Purchases into a single Add for Review draft, with the web and API surfaces catching up over the summer. The steps above still match what most accounts see today, and the underlying rule does not change: your first paid product goes in with a version. If your App Store Connect looks different from what is described here, look for an Add for Review or draft submission control on the version page and add the products there. The section name and the first-product rule are the two things to anchor on.

## Where this sits if you built with Despia

This is an App Store Connect step, and it is identical no matter how the binary was produced. If you shipped your app with Despia, your subscriptions run through the RevenueCat integration on both stores, and the build already carries the StoreKit configuration it needs. Nothing changes on the build side to fix this rejection. You attach the products to the version in App Store Connect, resubmit, and it goes through.

The takeaway is worth keeping: when App Review complains about missing In-App Purchase products, the fix is almost never in your code or your build. It is the one link between the products and the version that has to be made by hand, once, on your first paid release.

## Get it on the stores

Take the app you already built and ship it to iOS and Android without a CLI or a Mac. Code signing, subscriptions, and submission run from the browser.

[See the setup docs at setup.despia.com](https://setup.despia.com)