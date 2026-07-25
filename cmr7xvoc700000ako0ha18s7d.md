---
title: "Base44 Push Notifications with OneSignal (Native)"
seoTitle: "Base44 Push Notifications with OneSignal (Native)"
seoDescription: "Base44 push notifications, done natively: full OneSignal, Apple, and Firebase setup with Despia, then link users and send from your backend."
datePublished: 2026-07-05T15:21:41.647Z
cuid: cmr7xvoc700000ako0ha18s7d
slug: base44-push-notifications-with-onesignal-native
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/777e0ec5-2751-4a15-ae0b-5af66ab9284a.png
tags: webview, push-notifications, onesignal, base44

---

Base44 builds the app, but a web app cannot deliver a push to a phone that is closed, and on iOS web push barely works at all. Despia includes the native OneSignal SDK in your iOS and Android binaries, so your Base44 app sends real notifications through Apple and Google, delivered even when the app is not running. This is the whole setup: OneSignal, Apple, Firebase, and Despia, then linking users and sending.

## Why web push does not cut it on a mobile app

Web push has two hard limits on a real mobile app. On iOS it only fires for a PWA the user manually added to the home screen, it did not exist before iOS 16.4, and it is unreliable even now. And a web page receives nothing once the browser or app is fully closed, which is exactly the moment a notification earns its keep. Native push goes through APNs on iOS and FCM on Android, wakes the device, and lands on the lock screen whether or not your app is open. Despia compiles OneSignal's native SDK into the binary, so you get that delivery while your Base44 app stays the single codebase.

## The full setup: OneSignal, Apple, Firebase, Despia

Most of this is one-time credential wiring. Do it in order, each step feeds the next.

### Create the OneSignal app

1.  Sign up at onesignal.com and create an app. It is free up to 10,000 subscribers, so it does not gate early work.
    
2.  When it asks for platforms, pick Native iOS and Native Android. Do not choose Web Push, even though your Base44 code is web. Despia ships a native binary, so it needs the native platforms.
    

### iOS: Apple Push Key and bundle IDs

3.  In Apple Developer, go to Certificates, Identifiers and Profiles, then Keys, and create a key with Apple Push Notifications service (APNs) enabled. Download the `.p8` (you get one download), and note its Key ID plus your Team ID from the top right of the account.
    
4.  Under Identifiers, open your core bundle id (for example `com.despia.myapp`) and enable the Push Notifications capability. Apple rejects registration without it, even with a valid key.
    
5.  Create a second bundle id for OneSignal's notification service extension: your core id with `.OneSignalNotificationServiceExtension` appended, so `com.despia.myapp.OneSignalNotificationServiceExtension`. Enable Associated Domains and Push Notifications on it. The name must be exact, Despia provisions this target at build time.
    
6.  Optional, for delivery metrics or rich push with images and buttons: create an App Group `group.com.despia.myapp.onesignal` and add it to both bundle ids. Basic push works without it.
    
7.  Back in OneSignal, open Settings, Push and In-App, Apple iOS. Upload the `.p8`, paste the Key ID, Team ID, and iOS bundle id, and save. OneSignal validates with Apple immediately.
    

### Android: Firebase

8.  Create a project at console.firebase.google.com and add an Android app with the same package name as your Despia app. In Project settings, Cloud Messaging, grab the Server Key and Sender ID. If the legacy Server Key is hidden, enable the Cloud Messaging API in Google Cloud first.
    
9.  In OneSignal, open Settings, Push and In-App, Google Android, paste the Server Key and Sender ID, and save.
    

### Keys and Despia

10.  In OneSignal, open Settings, Keys and IDs. Copy the OneSignal App ID for the client, and the REST API Key for your backend. The REST API Key stays on your server, never in client code.
     
11.  In the Despia Editor, open App, Settings, Integrations, OneSignal, add your OneSignal App ID in the iOS and Android fields (the same value in most setups), and save. Then trigger a fresh build. The OneSignal SDK and the notification service extension compile into the binary and are signed against the bundle ids you registered, so this cannot ship over the air.
     

Until you rebuild, OneSignal stays inactive even with the App ID saved. The `setonesignalplayerid://` call resolves silently, and backend sends return success from the API but never reach the device. If notifications stop after a settings change, rebuild before anything else.

## Link the device to your user

Despia registers the device with OneSignal at launch. Your job is to tell OneSignal which user this device belongs to, on every authenticated load, so you can target that user later by your own id.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// link this device to your user on every authenticated load
function linkPush(userId) {
  if (!isDespia) return
  despia(`setonesignalplayerid://?user_id=${userId}`)
}
```

The `user_id` you pass here becomes the user's `external_id` in OneSignal, and it is the exact value you target with `include_external_user_ids` when you send. Keep it stable, and keep it identical on both ends, or the send reaches no one.

## Ask for permission without nagging

Check whether push is actually enabled and, if not, send the user to their settings instead of blocking them.

```javascript
async function promptIfPushOff() {
  if (!isDespia) return
  const { nativePushEnabled } = await despia('checkNativePushPermissions://', ['nativePushEnabled'])
  if (!nativePushEnabled) despia('settingsapp://')  // opens this app's system settings
}
```

A quiet banner on load when push is off converts better than a hard gate.

## Send from a Base44 function

Sends run server-side so the REST API key stays private, and Base44 can run backend functions, which is where this belongs. Target the same id you linked.

```javascript
// server-side in a Base44 function, REST API key never ships to the client
async function sendPush(userId, title, message) {
  await fetch('https://onesignal.com/api/v1/notifications', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Basic ${ONESIGNAL_REST_API_KEY}`,
    },
    body: JSON.stringify({
      app_id: ONESIGNAL_APP_ID,
      include_external_user_ids: [userId],
      headings: { en: title },
      contents: { en: message },
    }),
  })
}
```

## Open the right screen when they tap

A notification is only useful if the tap lands the user somewhere specific. OneSignal carries a `data` object with each notification, and Despia reads three fields from it on tap. `path` updates the URL through the History API and fires a `popstate`, with no reload, which is what you want almost always. `metadata` is any JSON you attach to restore state. `url` is the legacy option that forces a full WebView reload, for when a reload is genuinely needed.

Send the deep link as part of the same backend call:

```javascript
// server-side in a Base44 function, send with a deep link
async function sendPush(userId, title, message, path, metadata) {
  await fetch('https://onesignal.com/api/v1/notifications', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Basic ${ONESIGNAL_REST_API_KEY}`,
    },
    body: JSON.stringify({
      app_id: ONESIGNAL_APP_ID,
      include_external_user_ids: [userId],
      headings: { en: title },
      contents: { en: message },
      data: { path, metadata },
    }),
  })
}

// deep link a user to their shipped order
sendPush(userId, 'Your order shipped', 'Tap to track it', '/orders/4567?tab=tracking', { orderId: 4567 })
```

That is all `path` needs. Despia updates the URL and fires `popstate`, and your router navigates on its own, so there is no client code for the common case. Add `window.onNotificationEvent` only when you send `metadata` and want to restore state, assigned outside the `isDespia` gate:

```javascript
window.onNotificationEvent = (payload) => {
  if (!payload.metadata) return
  // object from the REST API, string from the OneSignal dashboard
  const meta = typeof payload.metadata === 'string' ? JSON.parse(payload.metadata) : payload.metadata
  restoreState(meta)
}
```

The one exception is a router that ignores the synthetic `popstate`. If that is yours, navigate from `payload.path` in the same handler instead, but do not also lean on the automatic change, or you navigate twice.

The tap resolves from any app state: foreground changes the path immediately, background applies it once the WebView is shown, and a cold start buffers the payload until the page loads. It fires once per tap.

## Get it on the stores

Take your Base44 app to iOS and Android with native push already wired in. Code signing and submission run from the browser, no Mac and no CLI.

[See the setup docs at setup.despia.com](https://setup.despia.com/native-features/onesignal/introduction)