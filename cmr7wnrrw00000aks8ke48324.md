---
title: "OneSignal Push on a Lovable App: The Simpler Path"
seoTitle: "OneSignal Push on a Lovable App: The Simpler Path"
seoDescription: "Adding OneSignal to a Lovable Capacitor app can break with linking errors and white screens. Here is why, and how Despia ships push in one call."
datePublished: 2026-07-05T14:47:33.236Z
cuid: cmr7wnrrw00000aks8ke48324
slug: onesignal-push-on-a-lovable-app-the-simpler-path
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/d7a38cc9-186d-487f-9d62-6468af9d2adb.png
tags: webview, push-notifications, capacitor, onesignal, lovable

---

A solo founder building a wellness app on Lovable posted a familiar plea on Reddit: multiple iOS builds already on TestFlight, Stripe and Supabase auth working, and then push notifications turned into a wall. The [full thread is here](https://www.reddit.com/r/lovable/comments/1tswg49/lovable_capacitor_onesignal_ios_push/). Two OneSignal plugins, two different failures, and a white screen that only appears in the native build. This is not a skills problem. It is the cost of the path, and there is a shorter one.

## What actually broke, and why it is not the founder's fault

The Lovable app runs through Capacitor 8, which produces a real native iOS project. That is the official, capable path. The trouble starts at the native plugin boundary.

First attempt: `onesignal-cordova-plugin`. On Capacitor 8 it gets pulled in as a Swift Package, which cannot link against the OneSignalXCFramework native SDK even with the pod declared. The build dies on `'OneSignalFramework/OneSignalFramework.h' file not found`. That is a native linking mismatch between how the plugin ships and how Capacitor 8 resolves native dependencies. Nothing in the web code causes it and nothing in the web code fixes it.

Second attempt: `@onesignal/capacitor-plugin`. The build succeeds, the app launches, and it white-screens. React never mounts. This is the more insidious failure, because it passes CI. The likely cause sits somewhere in the native, plugin, and bootstrap boundary: the WebView crashes or JavaScript fails before the app boots, while the browser preview keeps working fine. So the one environment where you can reproduce it is the one that takes a full native build and device inspection to see.

Both failures are the same shape: you are now debugging native plugin linking and native bundling, on a project you did not write, for a feature that is supposed to be a checkbox.

## The Despia difference: push is already in the runtime

Despia takes the Lovable web app you already built and ships it as a native binary, with the OneSignal SDK bundled into the runtime. There is no plugin to install, no pod or Swift Package to link, no XCFramework to resolve, and no second native project to maintain. The web app stays the source of truth.

Because the SDK is already compiled in, the device registers with OneSignal automatically at launch. You do not install a plugin and you do not call a register function. You link the signed-in user to their device with one call, on every authenticated load:

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// Link the signed-in user to this device on every authenticated load.
function linkPushUser(userId) {
  if (isDespia) {
    despia(`setonesignalplayerid://?user_id=${userId}`)
  }
}
```

Check whether push is actually enabled and route the user to settings if it is not:

```javascript
async function ensurePushEnabled() {
  if (!isDespia) return

  const res = await despia('checkNativePushPermissions://', ['nativePushEnabled'])
  if (!res.nativePushEnabled) {
    // Denied or never asked. Send them to settings to turn it on.
    despia('settingsapp://')
  }
}
```

Then send from your backend, targeting the same user ID you linked:

```javascript
// Backend: target the user by the external ID you linked on the client.
await fetch('https://onesignal.com/api/v1/notifications', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Basic YOUR_REST_API_KEY',
  },
  body: JSON.stringify({
    app_id: 'YOUR_ONESIGNAL_APP_ID',
    include_external_user_ids: [userId],
    headings: { en: 'Title' },
    contents: { en: 'Message body' },
  }),
})
```

No `.h` file to find, no bundle that white-screens when you add a plugin, because you are not adding a native plugin to a native project. The runtime already has it.

## Setup, start to finish

The OneSignal side is standard. The Despia side is one field.

1.  Create a OneSignal app and choose Native iOS and Native Android as the platforms.
    
2.  Upload your Apple `.p8` Auth Key for iOS, and your Firebase credentials for Android.
    
3.  Copy your OneSignal App ID.
    
4.  In Despia, open App > Settings > Integrations > OneSignal and paste the App ID, then build.
    

The [full OneSignal setup is in the docs](https://setup.despia.com/native-features/onesignal/introduction), including the permission check, deep linking from a notification, and segmentation.

## The one thing people get wrong

The mistake is not a missing register call. Registration is automatic, so anything you find that tells you to hand-roll a `registerpush://` step is wrong and does nothing. The real requirement is calling `setonesignalplayerid://` on every authenticated load. If you link the user once at signup and never again, a reinstall or a new device leaves OneSignal with no external ID to target, and your backend sends land nowhere. Treat it like an identify call: run it whenever you know who the user is.

## Lovable plus Capacitor push versus Despia push

|  | Lovable + Capacitor + OneSignal | Despia + OneSignal |
| --- | --- | --- |
| Native plugin install | Yes, and it must link correctly | None, SDK is in the runtime |
| Projects to maintain | Web app plus a native Capacitor project | Web app only |
| Where push failures surface | Native build, after CI and upload | Web call, testable directly |
| Device registration | Plugin lifecycle you configure | Automatic at launch |
| Send a notification | OneSignal REST API | OneSignal REST API, same |
| Updating app logic | Rebuild and resubmit | Over the air, no resubmission |

## When Capacitor is the right call

This is the honest part. Capacitor hands you the whole native project, so if your app needs custom native code with non-standard wiring, a hand-edited AppDelegate, or a native SDK that has to be injected at a specific point in launch, you have the files to do it. That control is a real strength, and some apps need it.

Most do not. Despia's 50+ native features cover roughly 99% of what real apps reach for, and push is one of them, already wired into the app lifecycle so you do not have to do that wiring yourself. That is the exact AppDelegate work the Capacitor path leaves on your plate. And when you do hit the rare case that needs a hand-edited delegate, the escape hatch is there: export the full Xcode and Android Studio projects and go native the day you actually need to.

For a solo non-technical founder who just wants push notifications on the wellness app they already built, that control is a cost, not a benefit. The whole point is to not spend a weekend reading Xcode linker errors.

## Get push working on the app you already built

Keep the Lovable app. Despia ships it to the App Store and Google Play as a native binary, gives it OneSignal push and 50+ other native features through one JavaScript call, and updates the web layer over the air without a resubmission.

[Read the OneSignal setup in the docs](https://setup.despia.com/native-features/onesignal/introduction) or see [Despia for Lovable](https://setup.despia.com/platform-support/lovable).