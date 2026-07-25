---
title: "iOS Build Error 65: Register Your Bundle IDs"
seoTitle: "iOS Build Error 65: Register Your Bundle IDs"
seoDescription: "iOS build Error 65 means your Apple Bundle IDs are not registered yet. Here is exactly what to register to get provisioning profiles generating."
datePublished: 2026-07-01T11:53:55.586Z
cuid: cmr20p2sm00000ai3df0oa5ck
slug: ios-build-error-65-register-your-bundle-ids
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/11e2908d-1217-4746-9f04-c3565e96fb5c.jpg
tags: ios-app, webview, pwa, ci-cd, appstore

---

Your build ran the whole way to code signing and then stopped with `Error Code 65` and a note that provisioning profiles could not be created. Nothing is broken. The build reached the last step and found that the Bundle IDs it needs to sign against do not exist yet on your Apple Developer account, so Apple returned nothing and the signing step had no profile to use. You register the IDs once, rebuild, and it signs itself from then on.

## What Error 65 actually means

Error 65 is a generic archive failure from `xcodebuild`. The real cause is in the lines around it, and for a first iOS build it is almost always the same thing: provisioning profiles could not be generated because the App IDs are not registered.

Provisioning profiles are created by Apple, not by Despia. Apple can only create a profile for a Bundle ID that already exists in your account, with the right capabilities enabled on it. If the core Bundle ID was never created, Apple has nothing to build a profile from, so the signing step fails and the archive dies with code 65. This is a registration gap, not a certificate problem, and revoking certificates will not fix it.

## The two IDs almost every app needs

You do not need to register everything at once. For a standard app with push notifications, two Bundle IDs cover it.

Go to [developer.apple.com](https://developer.apple.com), sign in, open Certificates, Identifiers and Profiles, then Identifiers.

Throughout this post, `{your-bundle-id}` stands in for your own app's Bundle ID, the real reverse-domain string for your app (for example `com.acme.myapp`), which you will find in your Despia project settings. Do not register the token literally. Every extension ID and App Group below has to share your real base, so if your Bundle ID is `com.acme.myapp`, the OneSignal ID is `com.acme.myapp.OneSignalNotificationServiceExtension` and the group is `group.com.acme.myapp.onesignal`.

**1\. The core Bundle ID:** `{your-bundle-id}`

This is your main app. Register it and enable these capabilities:

*   App Attest
    
*   App Groups
    
*   Associated Domains
    
*   iCloud (Include CloudKit Support)
    
*   Push Notifications
    

**2\. The OneSignal ID:** `{your-bundle-id}.OneSignalNotificationServiceExtension`

This one runs rich push notifications (images, media, action buttons). Register it and enable:

*   App Groups
    
*   Associated Domains
    
*   Push Notifications
    

Then create one App Group, `group.{your-bundle-id}.onesignal`, and link it to both IDs above. Save, go back to Despia, and rebuild. Provisioning profiles generate on their own.

Do not skip this ID if you want push. If the OneSignal Bundle ID is missing, push notifications will not work on iOS at all, and there is no error at build time telling you why, the build signs fine and the notifications just never arrive. This is the single most common reason push silently fails after launch.

OneSignal will still deliver basic push without its App Group linked, but the extension is what powers the rich features: images, media attachments, action buttons, badge and delivery confirmation. Skip the App Group and you drop to plain text-only notifications with reduced delivery reliability. Register the ID and link the group to get the full push capability.

If you do not need push notifications at all, the core Bundle ID alone is enough to get a signed build.

## The extensions you can skip

Your build schema may reference more IDs than you use. Widgets, a share extension, and an App Clip each get their own Bundle ID, but you only register them if you actually ship those features. Skip any you are not using and the build still signs.

The important distinction: skipping a feature you do not ship is fine, but if you do use one of these, its App Group is not optional. Unlike push, these features depend entirely on the shared container to function. Skip the App Group on a widget, share extension, or App Clip and it will not work fully, and in most cases will not work at all. The widget has nothing to read, the share target cannot hand data back to the app, and the App Clip cannot pass state to the full app. If you ship the feature, register the ID and link the group, there is no partial mode here.

| Feature | Bundle ID | App Group | Capabilities |
| --- | --- | --- | --- |
| Home screen widget | `{your-bundle-id}.ImageWidget` | `group.{your-bundle-id}.widgetsharing` | App Groups |
| Share extension | `{your-bundle-id}.ShareExtensionTarget` | `group.{your-bundle-id}.sharetarget` | App Groups |
| App Clip | `{your-bundle-id}.Clip` | `group.{your-bundle-id}.Clip` | App Groups, Associated Domains, Push Notifications |

If you do register the full set, the linking has to be exact. Each App Group is created once, then linked to the bundles that use it. Enable App Groups on every ID first, then link them like this:

| App Group | Linked to |
| --- | --- |
| `group.{your-bundle-id}.onesignal` | Core Bundle ID, OneSignal ID |
| `group.{your-bundle-id}.widgetsharing` | core Bundle ID, Widget ID |
| `group.{your-bundle-id}.sharetarget` | Core Bundle ID, Share Extension ID |
| `group.{your-bundle-id}.Clip` | Core Bundle ID, App Clip ID |

The pattern is: the core Bundle ID links to all four groups, and each extension links only to its own single group. If an extension is linked to a group the core ID does not also carry, or the core ID is missing one, the entitlements will not match the profile and the build fails signing.

## How to actually link a group to an ID

Creating a group and enabling the capability are two separate actions from linking, and the linking is the step people miss. Here is the full sequence in the Apple Developer portal.

1.  Create the groups first. Go to Identifiers, switch the top-right dropdown from App IDs to App Groups, click the plus, and create each `group.{your-bundle-id}.*` string you need. A group has no settings, you just register the identifier.
    
2.  Open the core Bundle ID. Back in Identifiers, switch the dropdown to App IDs and open your core ID.
    
3.  Enable App Groups on it, then click Edit or Configure next to the capability. A list of every App Group in your account appears with a checkbox each.
    
4.  Tick all four groups on the core ID. This is the part that matters. The core Bundle ID has to be linked to every group, not just one, because the core app is the shared container each extension reads from and writes to. If a group is only linked to its extension and not to the core ID, the two never share a container and the data path is dead, on top of the profile mismatch that fails the build.
    
5.  Save the core ID.
    
6.  Open each extension ID in turn, enable App Groups, click Edit, and tick only that extension's own single group. OneSignal gets `onesignal`, the widget gets `widgetsharing`, and so on.
    
7.  Save each one.
    

After every ID is saved with its groups ticked, go back to Despia and rebuild. Apple regenerates the provisioning profiles with the matching entitlements and the signing step passes.

Why the core ID carries all four: App Groups are how the main app and its extensions share a storage container on device. The extension writes, the core app reads, or the reverse. That only works if both sides are members of the same group, and since the core app pairs with all of them, it has to sit in every group. Miss one link and that one extension silently stops syncing with the app.

## The three things that silently break

Most repeat failures after registration come down to one of these.

**App Groups enabled but not linked.** Turning on the App Groups capability is not the same as adding the group to the identifier. Open each ID, enable App Groups, then explicitly add the specific `group.` string. A profile that lists no groups will still fail signing.

**The App Clip registered as a regular App.** If you use an App Clip, it has to be registered as an App Clip, not a standard App. Set its Parent App to your core Bundle ID and its Product Name to exactly `Clip`. Registered as a normal app, it will never resolve as a Clip.

**A leftover identifier from an earlier attempt.** If Apple says an App ID with that identifier is not available, it already exists in your account from a previous try. Delete the old one in Identifiers before registering a new one with the same string, otherwise you end up with a half-configured ID and the same error.

## After you rebuild

Once the IDs are registered and the groups are linked, the build signs itself and every future build reuses the same profiles. This is one-time setup. You will not touch the Apple Developer portal again unless you add a new capability or a new extension later.

The full walkthrough with screenshots lives in the docs: [setup.despia.com/deployment/apple-ios/automatic](https://setup.despia.com/deployment/apple-ios/automatic).

## Get it on the stores

Take the app you already built and ship it to iOS and Android without a CLI or a Mac. Code signing and submission run from the browser, and Bundle ID setup like this is a one-time step.

[See the setup docs at setup.despia.com](https://setup.despia.com)