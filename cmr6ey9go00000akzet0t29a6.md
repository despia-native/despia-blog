---
title: "App Splash Screen Setup (and the White Screen Fix)"
seoTitle: "App Splash Screen Setup (and the White Screen Fix)"
seoDescription: "Set up an app splash screen for iOS and Android with one transparent GIF, and fix the silent white screen that most builders hit on the first try."
datePublished: 2026-07-04T13:44:03.452Z
cuid: cmr6ey9go00000akzet0t29a6
slug: app-splash-screen-setup-and-the-white-screen-fix
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/0f4961c2-b210-4ba6-bf0f-5cade4738d73.jpg
tags: webview, pwa

---

Your app splash screen is the first thing anyone sees when they open the app, the frame that covers the gap between the tap and the first paint. In Despia you set it with a single image, no Xcode and no native project. The part that trips people up is not the design. It is one metadata detail that turns a perfectly good logo into a white screen on iOS.

## App splash screen requirements: size, format, transparency

It is a 1024x1024 animated GIF that sits over your splash background color while the app boots. Three rules matter: it has to be square at 1024x1024, it has to have a transparent background, and it has to actually animate across more than one frame. If you came from Xcode, this is the same thing Apple calls the launch screen, handled here as one file instead of a storyboard.

That second rule is the exact opposite of your app icon, which cannot be transparent. Same size, same tool, opposite background. That reversal is where most of the confusion starts. If you export a splash with a solid background, it renders as a solid tile over your splash color instead of your logo floating on it.

## How to make a splash screen in Canva

Open a 1024x1024 canvas in Canva, drop your logo in, and center it so it fills most of the frame. Add an animation: fade in, blur in, a swipe, or a bounce. It has to be a real animation, because Despia renders the splash through SwiftyGif on iOS and that library needs a multi-frame GIF to decode. Keep it under two to three seconds, since it plays on every cold start and a long one reads as a delay rather than a nice touch. If you want a near-still splash, use a slow fade rather than a single frame, so the file still animates.

Export with Download, then GIF, and turn on Transparent Background. That is the whole design step. One file, both platforms.

## Why your splash screen shows a white screen on iOS

Here is the failure mode worth knowing before you hit it. On iOS, Despia renders animated splash GIFs through a strict Swift GIF library. If the file's metadata is wrong, it does not throw an error and it does not fall back to a static frame. It shows nothing. You get a white screen and no clue why.

The library expects real GIF metadata:

| Requirement | Why it matters |
| --- | --- |
| Global Color Table | Must be present. Some export tools skip it. |
| Frame delays | Each frame needs a valid delay, usually 0.03s to 0.1s. Zero or missing delays break playback. |
| Logical Screen Descriptor | Width and height defined in the header, not inferred from the frames. |
| No corrupted frames | Partial or malformed frame data fails silently. |
| Reasonable file size | Very large GIFs, 10MB and up, can run out of memory on older devices. |
| Transparent background | Ensures the GIF displays correctly over your splash color. |

Canva writes all of this correctly on its own, even on the free plan, which is the reason it is the path we point people to. If you export from somewhere else and land on a white screen, the file is almost always the cause, not the build. Photoshop is fine too: File, Export, Save for Web (Legacy), GIF, transparent background.

One thing to rule out first: the GIF has to be animated. Despia renders the splash through SwiftyGif, which is built for animated GIFs, so a single-frame file gives it no sequence to decode and drops straight to the white screen. If your splash is one static frame, that alone is the cause. Give it a short animation plus the transparent background, and SwiftyGif has something to play.

## Where to upload the splash screen in Despia

Upload the GIF in the editor under the Splash Screen Layout section. It is the same section for iOS and Android, so one file covers both. Upload it, confirm, and verify the splash appears the way you expect on the next build.

## Common questions

**What size should an app splash screen be?** 1024x1024 pixels, square, exported as a GIF with a transparent background. The same file works for both iOS and Android.

**Why is my splash screen showing a white screen?** On iOS the GIF is rendered by a strict library that fails silently when the file's metadata is wrong or the GIF has only one frame. Re-export it from Canva with a short animation, which writes correct metadata automatically, and the white screen goes away.

**Does the splash screen have to be animated?** Yes. Despia renders the splash through SwiftyGif on iOS, which needs a multi-frame GIF, so a single static frame fails to a white screen. Fade, blur, swipe, and bounce all work. Keep it under two to three seconds, since it plays on every cold start. For a near-still look, use a slow fade so the file still animates.

**Do I need a separate splash screen for iOS and Android?** No. One 1024x1024 transparent GIF covers both platforms from the same Splash Screen Layout section.

## Add this to your app

Your splash screen is one transparent GIF away, set in the editor, shipped to both stores from the web app you already have. No Xcode, no native project to maintain.

[Read the full reference at setup.despia.com](https://setup.despia.com)