---
title: "Fix the Google Play Billing Library Warning in Despia"
seoTitle: "Fix the Google Play Billing Library Warning in Despia"
seoDescription: "Google Play requires Billing Library 8.0.0 by Aug 31, 2026. Clear the Google Play Billing Library warning on your Despia app with a single rebuild."
datePublished: 2026-07-22T08:08:00.603Z
cuid: cmrvsvfpu000109kx4gul0tdu
slug: fix-the-google-play-billing-library-warning-in-despia
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/0abe4fdc-7799-4fcc-82fc-4d34ab32fe8b.png
tags: app-development, android-app-development, android-development, android, playstore, in-app-purchases, googleplay

---

A notice landed in your Play Console: your app uses a version of the Google Play Billing Library that will be deprecated soon, and from Aug 31, 2026 all apps must use version 8.0.0 or later. If your app is built with Despia, this is not a code problem and there is nothing to migrate. The fix is a version bump and a rebuild, about ten minutes of your time. Here is what the warning actually means and the exact steps to clear it.

## What Google is actually requiring

Google deprecates old major versions of the Play Billing Library on a rolling schedule. The current round says that from Aug 31, 2026, Play will reject any new app update that still carries a billing library below 8.0.0.

Two details matter more than the scary wording:

1.  The check runs against the newest app bundle sitting on your tracks. Play scans what is inside the AAB, so the warning is telling you about the build you uploaded months ago, not about anything you did recently.
    
2.  The deadline only gates new submissions. Your live app keeps working for users after Aug 31, 2026 either way. What you lose, if you do nothing, is the ability to ship updates.
    

Google's notice also recommends moving to version 9. The requirement is 8.0.0 or later, and that is the bar that clears the warning.

## Why your app got flagged when you never touched billing code

The Play Billing Library ships inside the Despia runtime as part of its in-app purchase integration. It is there so that apps selling through Google Play can do it with one JavaScript call, but Play does not check whether you use it. It checks which version is present in the bundle. That is why apps that handle payments entirely outside Google Play, through Stripe or any external checkout, get the same warning as apps selling subscriptions through the Play Store.

The current Despia runtime bundles Billing Library 8.3.0 (at the time of writing), comfortably past the 8.0.0 requirement. The runtime pins the version its purchase integration is built and tested against, so you do not need to chase version 9 yourself. Any Android build you produce today carries a compliant library automatically.

Which means the entire fix is producing one new build.

## Clear it in four steps

**1\. Increase your version code.** Go to Despia > Settings > Version and increase the version code. Do this first. Google Play rejects any bundle that reuses a version code it has already seen, so a rebuild without a bump has nowhere to go.

**2\. Rebuild.** Go to Publish > Android > Publish Project. Despia builds a fresh AAB on the current runtime, ready for submission.

**3\. If automatic deployments are enabled, check Play Console.** If you set up automatic deployments with the service account during your Android setup, the new AAB is uploaded for you. Within 10 to 15 minutes it appears in your Play Console, in the latest releases overview or on the Internal testing track.

**4\. If you do not see it there, upload manually.** Take the AAB from your build and upload it yourself: Play Console > Test and release > Internal testing > Create release, attach the AAB, and roll it out.

Once the build is on a track, promote it to Production the same way you release any update. Google's review applies as usual, so leave normal review time before the deadline.

## Two things that quietly keep the warning alive

**An old bundle on a forgotten track.** The requirement applies across all your tracks. If you roll out to Production but an old AAB is still the active release on internal, closed, or open testing, the app stays flagged. Promote the same new build to every track you actively use.

**"Version code has already been used" on manual upload.** If you try to upload the AAB by hand and Play Console rejects it with this message, that is not a failure. It means automatic deployment already delivered the build, and it is sitting on one of your tracks. Find it in the releases list and roll it out from there instead of rebuilding.

After the new build is live across your tracks, the warning clears on its own. It can lag a few days behind the rollout, so do not wait for August to do this.

## Try Despia

Turn your web app into a real iOS and Android app, with native features and over-the-air updates, from a single codebase.

[Learn more at setup.despia.com](https://setup.despia.com)