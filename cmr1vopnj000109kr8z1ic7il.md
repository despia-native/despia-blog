---
title: "Fix NSUserTrackingUsageDescription App Store Block"
seoTitle: "Fix NSUserTrackingUsageDescription App Store Block"
seoDescription: "Your Despia build shows an NSUserTrackingUsageDescription warning and you do not track users. Here is why Apple flags it and how to clear it."
datePublished: 2026-07-01T09:33:40.457Z
cuid: cmr1vopnj000109kr8z1ic7il
slug: fix-nsusertrackingusagedescription-app-store-block
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/6e04b1b8-1ea2-4bcc-a08e-41c22f2cc544.jpg
tags: webview, appstore, appreview

---

You submitted your app and App Store Connect stopped you with a message about `NSUserTrackingUsageDescription`. You do not run ads, you never touch the IDFA, you never asked to track anyone. The key is in the build anyway, and now Apple wants you to declare tracking you are not doing. Here is what the key means and how to clear it in Despia in a few minutes.

## What Apple is actually telling you

This is the exact message that stops the submission:

> Your app contains NSUserTrackingUsageDescription, indicating that you will request permission to track users. To publish this information on your app's product page, you must indicate which data types are tracking users. If this is incorrect, update your app binary and upload a new build to App Store Connect.

`NSUserTrackingUsageDescription` is the Info.plist string that goes with App Tracking Transparency. If the key is in your binary, iOS treats your app as one that intends to call the ATT prompt and track users across other apps and websites. App Store Connect then forces the matching declaration: either you fill in the tracking data types in the App Privacy section, or you ship a binary without the key. Until one of those happens, the version will not go to review.

So the message is not really a bug. It is Apple keeping your binary and your privacy answers consistent. The fix depends on which side of that consistency you actually want.

## Why the key is there when you do not track

In a hand-built Xcode project you would have added that key yourself. In a managed build you did not, which is why this feels like it came out of nowhere. In Despia the key is controlled by one build setting, Disable Tracking Declaration. Left off, the key ships in the binary. Switched on, it does not.

For context, this is the key sitting in your Info.plist right now:

```xml
<key>NSUserTrackingUsageDescription</key>
<string>This identifier will be used to deliver personalized ads to you.</string>
```

If nothing in your app calls the ATT prompt, that declaration is dead weight and it is the only reason Apple is asking for a privacy answer you do not have.

## Why Despia keeps the declaration on by default

The default is deliberate. Your app is a web app, and web apps carry tracking the developer often did not write and cannot see in the compiled binary. Apple's bar is specific: tracking means your app's data is linked with data from other companies' apps, websites, or offline properties and then used for advertising or shared with a data broker. Not every analytics tag clears that bar. A first-party tag that never leaves your own properties may not count. But a Meta or TikTok pixel firing for ad attribution, or any script wired into cross-site profiling or third-party data sharing, does count, whether or not you ever trigger the ATT prompt yourself. Configuration decides it, and on a web stack that configuration is easy to inherit without noticing.

On a WebView hybrid app that is the failure mode that catches people out. The tracking lives inside a third-party script somewhere in your web stack, so the binary looks clean to you while it quietly reports to an ad platform on every session. If you disable the declaration on a build that still loads those scripts, you have not fixed anything. You have shipped tracking while telling Apple you do not track, and that is a privacy policy violation, not a warning. It is grounds for rejection at review, or a takedown later if it clears and Apple catches it in a scan.

So Despia ships the declaration on as the safe posture, not by accident. It is the difference between a binary that overstates what it does and a binary that misrepresents what it does, and only the second one gets you in real trouble. Switch Disable Tracking Declaration on once you have gone through your web codebase, confirmed there are no ad pixels, attribution scripts, or third-party tags that share data for advertising or link users across other companies' apps and sites, and you are certain the app does not track by Apple's definition. Until you have done that audit, leave it on.

## The fix: turn off the declaration and ship a new build

Once you have verified your web codebase does not track, remove the declaration and upload a clean binary.

1.  Open the Despia dashboard, go to Settings, and switch on Disable Tracking Declaration.
    
2.  Still in Settings, bump the version number. App Store Connect will only accept a new build under a higher number.
    
3.  Top right, go to Publish, choose iOS, and run Publish Project. Wait for the build to finish.
    
4.  In App Store Connect, open the app, go to the Distribution tab, select the version, and add the new build.
    
5.  Submit for review.
    

One snag to expect. If the version is already locked in Waiting for Review, App Store Connect will not let you swap the build. Remove the app from review first, then attach the new build and resubmit. Nothing is lost by doing this, the review clock just restarts on the new binary.

The rebuilt binary has no `NSUserTrackingUsageDescription` key, so the warning is gone and Apple stops asking you to declare tracking.

## "Can you just remove it from our Info.plist?"

This one comes up constantly. A build has `NSUserTrackingUsageDescription`, it is blocking submission, the team is sure they do not use ATT tracking, and they ask us to strip the key out of the Info.plist by hand.

We do not hand-edit the plist, and you do not need us to. In a managed build the Info.plist is regenerated on every publish, so a manual edit would be overwritten on your next build anyway. Disable Tracking Declaration is the durable version of the same change: switch it on, and the key is absent from every binary from that point on. That is the whole fix, and it holds across rebuilds instead of lasting one upload.

## If your app does track, declare it instead

If the audit turns up tracking you want to keep, personalized ads through AdMob, an advertising SDK that reads the IDFA, or web pixels you rely on for attribution, do not remove the declaration. Leave Disable Tracking Declaration off, go to App Store Connect, open App Privacy, and declare the tracking data types Apple is asking for. That path clears the same warning and it is the honest one. Removing the key while the app still tracks is the violation, not the fix.

## Clear the rejection and resubmit

Set Disable Tracking Declaration to match what your app actually does, bump the version, and publish a fresh iOS build from the browser. No Xcode, no Mac, no manual plist surgery.

[See the setup docs at setup.despia.com](https://setup.despia.com)