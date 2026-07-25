---
title: "Google Play Version Code Already Used: The Fix"
seoTitle: "Google Play Version Code Already Used: The Fix"
seoDescription: "Google Play says your version code has already been used because Despia already uploaded the build. Here is where to find it and what not to do."
datePublished: 2026-07-06T11:28:26.725Z
cuid: cmr94zkgh000109ko1fgl2gkm
slug: google-play-version-code-already-used-the-fix
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/6507c12a-c118-4300-bd13-6fa670aebcd5.png
tags: android-app-development, android, android-studio, playstore, playconsole, googleplay

---

You finished a build, grabbed the AAB, opened Play Console to upload it, and got stopped with "Version code has already been used. Please use a different version code." The number itself also looks oddly high, and there is no field anywhere in Despia to change it. If you are using Despia automatic Android deployment, this usually means the build is already in Play Console. The version code is taken because Despia already uploaded that AAB to the Internal testing track, so the fix is to find that release, not to upload the same file again.

This post explains what the version code is, why Despia sets it instead of you, and where your release actually is when Play tells you the code is taken.

## The version code is not yours to set, and that is deliberate

Every Android build carries a version code: a single integer that Google Play uses to tell releases apart. The one hard rule is that it has to increase on every upload and can never go down. Play uses it purely to order your releases, so once a number is used, it is spent for good.

Despia generates and increments this number for you on every build. That is why there is no input for it in the dashboard. It is not a setting that got hidden, it is a value the build system owns so you cannot accidentally ship a code lower than the one already live, which is the mistake that gets uploads rejected.

If the number looks large, that is fine. Play only cares that it keeps climbing, not what it starts at. The ceiling is 2,100,000,000, so a seven-figure code is nowhere near a limit you need to think about. You cannot lower it, and you do not need to.

## Why the manual upload fails

Despia uploads the AAB to Play Console automatically as part of the build. By the time you open the console to drag the file in yourself, that version is already sitting on the track. "Already been used" is not a failure. It is Play telling you the build you are holding is the same one it already has.

The instinct here is usually to bump the number and rebuild to get a fresh code. Do not. It wastes a build, and it does not change anything, because the next build will auto-upload in exactly the same way and you will land on the same screen with a higher number. The file is already where it needs to be. The work now is finding it, not re-sending it.

## Look on Internal testing, not Production

Despia always deploys to the Internal testing track. That is where your build is. If you are looking at Production and it reads empty, nothing failed, the build is sitting on Internal testing with the version code already claimed there.

This is the single most common reason people think the upload never happened. They open Production, see nothing, and rebuild. Before you touch anything, open the Internal testing track and look there first.

## Where your build actually is

Despia uploads and releases the build on Internal testing for you, so on that track it is usually already available or still going through Google's processing. You will land on one of these:

| What Play Console shows | What it means | What to do |
| --- | --- | --- |
| Available on Internal testing | Despia uploaded and released it | Nothing, it is on the track |
| In review or processing | Uploaded, Google is still checking it | Wait for it to clear |

To find it:

1.  Open Play Console and select the app.
    
2.  Under Release in the left sidebar, open Testing, then Internal testing.
    
3.  Open the Releases tab. Your build is listed there with the version code from the error.
    

The full Android submission flow, including how automatic deployment behaves per track, is documented at [setup.despia.com/deployment/google-android/automatic](https://setup.despia.com/deployment/google-android/automatic).

## Auto-release versus manual promotion

Despia uploads and releases the build to Internal testing on every build. That part is automatic and it is always Internal testing. Getting from there to a public track, Closed testing, Open testing, or Production, is your step. You promote the existing release up through the tracks in Play Console, you do not upload the file again to do it.

The distinction that matters: the upload is always automatic and always lands on Internal testing. Taking it public is the part that is yours. If you are staring at a version code error, the correct move is to open Internal testing and promote the release that is already there, not to send the AAB again.

## What you never have to do

Three things people try that are not needed and can cost you time:

Set or edit a version code by hand. Despia owns that number and increments it every build. There is no field for it because there is no reason for you to touch it.

Rebuild to get a higher code. The code is already correct and already higher than the last one. Rebuilding just uploads another version and moves the problem forward.

Upload the AAB yourself. It is already on Internal testing before you open the console. Your job is to find that release and promote it to whichever track you want it on.

## Get it on the stores

Take the app you already built and ship it to iOS and Android without a CLI or a Mac. Code signing, version management, and store submission run for you, so a version code error is a question of where the release is, not whether it uploaded.

[See the setup docs at setup.despia.com](https://setup.despia.com)