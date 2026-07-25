---
title: "Base44 Push Notifications: The Native Fix"
seoTitle: "Base44 Push Notifications: The Native Fix"
seoDescription: "Base44 has no native push, so builders hit a wall on iOS. Here is why web and Firebase push fall short, and how Despia adds native push in one call."
datePublished: 2026-07-05T15:04:37.593Z
cuid: cmr7x9q6500000ajrhf9y0wyt
slug: base44-push-notifications-the-native-fix
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/5b2a59f5-a2a3-43d7-bb17-7b5c18b5160a.png
tags: firebase, ios, android, webview, push-notifications, pwa, onesignal, appstore, googleplay, base44

---

Someone on the Base44 subreddit built a few apps, got ready to ship, and only then realized there is no native push notification feature. The [thread is here](https://www.reddit.com/r/Base44/comments/1trh5da/no_push_notifications_with_base44/), and the replies are a mess of conflicting advice. One person says use a third party. One says it is not possible. One says wrap it in Capacitor and integrate native push, but that requires professional help. One says nothing works, FCM will not wire up and OneSignal does not work. And one says it works fine over Firebase. They are all partly right, in different contexts, which is exactly why this is confusing. Here is the actual picture.

## Why the advice contradicts itself

Base44 exports an App Store build that runs your web app inside a WebView. There is no native binding to Apple Push Notification service or to Firebase Cloud Messaging in that export, which is why there is nothing for a native push SDK to attach to. That is the "nothing works" reality when people try to wire OneSignal's or FCM's native SDK into a raw Base44 build by hand.

The person who says Firebase works is also telling the truth, just about a different thing. Web push, delivered through Firebase or any web push service, does work, but with a hard platform split. On Android it is reasonably reliable. On iOS, web push only works when the site is installed as a Progressive Web App from Safari, added to the home screen, on iOS 16.4 or later. It does not work in a normal Safari tab, and it does not work inside an in-app browser or a wrapped WebView. So a Base44 app shipped to the App Store cannot use web push the way an installed Safari PWA can, because it is not that. This is Apple's restriction, not a Base44 or Firebase bug.

That leaves the honest summary: for Android only, web push can be enough. For a real iOS App Store app, you need native push, and Base44 does not provide the native binding to make that work on its own.

## The Capacitor route, and its real cost

The "wrap it in Capacitor" advice is legitimate. Capacitor gives you a full native iOS and Android project, and you can install a native push plugin and wire it in. That is why the commenter added that it can be done, but requires professional help. The help is not optional flavor. You are taking on a second native project alongside your Base44 app, installing and linking a native push SDK, and debugging the class of failures that come with it: plugin linking errors, native build config, and white screens that only show up in the native build. For someone who built on Base44 specifically to avoid that, it is a long way around.

## The Despia difference: native push is already in the runtime

Despia takes the Base44 app you already built and ships it as a native binary for iOS and Android, with the OneSignal SDK bundled into the runtime. There is no plugin to install, no native project to maintain, and no APNs or FCM wiring to do by hand. Native push is already compiled in and connected to the app lifecycle. The web app stays the source of truth.

Because the SDK is already in the binary, the device registers with OneSignal automatically at launch. You link the signed-in user to their device with one call, on every authenticated load:

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

Check whether push is actually enabled, and send the user to settings if it is not:

```javascript
async function ensurePushEnabled() {
  if (!isDespia) return

  const res = await despia('checkNativePushPermissions://', ['nativePushEnabled'])
  if (!res.nativePushEnabled) {
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

This is APNs on iOS and FCM on Android under the hood, the native path, reached from your existing Base44 JavaScript.

## Setup, start to finish

1.  Create a OneSignal app and choose Native iOS and Native Android as the platforms.
    
2.  Upload your Apple `.p8` Auth Key for iOS, and your Firebase credentials for Android.
    
3.  Copy your OneSignal App ID.
    
4.  In Despia, open App > Settings > Integrations > OneSignal, paste the App ID, then build.
    

The [full OneSignal setup is in the docs](https://setup.despia.com/native-features/onesignal/introduction), including the permission check, deep linking from a notification, and segmentation.

## The one thing people get wrong

The mistake is not a missing register step. Registration is automatic, so anything telling you to hand-roll a `registerpush://` call is wrong and does nothing. The real requirement is calling `setonesignalplayerid://` on every authenticated load. Link the user once at signup and never again, and a reinstall or a new device leaves OneSignal with no external ID to target, so your backend sends land nowhere. Treat it like an identify call: run it whenever you know who the user is.

## The three paths, side by side

|  | Base44 web / Firebase push | Base44 + Capacitor wrap | Base44 + Despia |
| --- | --- | --- | --- |
| Works on Android | Yes | Yes | Yes |
| Works on iOS App Store app | No, needs a Safari home-screen PWA | Yes, once wired | Yes, native |
| Projects to maintain | Web app only | Web app plus a native project | Web app only |
| Native push SDK | None available in the WebView | You install and link it | Built into the runtime |
| Effort | Low, but iOS falls short | High, "professional help" | One JavaScript call |

## When each path is right

If your app is Android only, or you are shipping an installed Safari PWA rather than an App Store app, web push through Firebase can be enough, and you do not need any of this. If you want a full native project to live in and you are comfortable owning it, Capacitor is a legitimate choice. For everyone who built on Base44 to avoid native tooling and still wants native push on both stores, Despia keeps the app you built and gives it the native path in one call.

## Get push working on the app you already built

Keep the Base44 app. Despia ships it to the App Store and Google Play as a native binary, gives it native OneSignal push and 50+ other device features through one JavaScript call, and updates the web layer over the air without a resubmission.

[Read the OneSignal setup in the docs](https://setup.despia.com/native-features/onesignal/introduction) or [start building at despia.com](https://despia.com).