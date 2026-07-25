---
title: "Web App Push Notifications with OneSignal (Native)"
seoTitle: "Web App Push Notifications with OneSignal (Native)"
seoDescription: "Web app push notifications, done natively: full OneSignal, Apple, and Firebase setup with Despia, then link users and send from any backend."
datePublished: 2026-07-05T15:36:39.779Z
cuid: cmr7yexc800000ajlc2dq48q0
slug: web-app-push-notifications-with-onesignal-native
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/727e8af7-a630-4f2a-8d2b-d9d1fce2e860.png
tags: website, webview, webapp, push-notifications, onesignal

---

* * *

Whatever your web app is built with, plain HTML and JavaScript included, a web page cannot deliver a push to a closed phone, and on iOS web push barely works. Despia ships your web app as a native iOS and Android binary with the OneSignal SDK compiled in, so you get real notifications through Apple and Google, delivered even when the app is not running. This is the full path: OneSignal, Apple, Firebase, and Despia, then linking users and sending.

## Why web push does not cut it on mobile

Browser push has two dead ends on a real mobile app. iOS only delivers it to a PWA the user added to the home screen by hand, it did not exist before iOS 16.4, and it stays unreliable. And once the app is fully closed, a web page receives nothing, which is exactly when a notification earns its place. Native push runs over APNs on iOS and FCM on Android, wakes the device, and reaches the lock screen whether the app is open or not. Despia compiles OneSignal's native SDK into the binary, so any web app gets native delivery without changing stacks.

## The full setup: OneSignal, Apple, Firebase, Despia

Mostly one-time credential wiring. Take it in order.

### Create the OneSignal app

1.  Sign up at onesignal.com and create an app. Free up to 10,000 subscribers, so it does not gate development.
    
2.  Choose Native iOS and Native Android for platforms, not Web Push. Despia produces a native binary even though your code is web.
    

### iOS: Apple Push Key and bundle IDs

3.  In Apple Developer, under Certificates, Identifiers and Profiles, Keys, create a key with Apple Push Notifications service (APNs) enabled. Download the `.p8` once, and note its Key ID and your Team ID.
    
4.  Under Identifiers, open your core bundle id (for example `com.despia.myapp`) and enable Push Notifications. Apple rejects registration without it.
    
5.  Register the notification service extension bundle id: core id plus `.OneSignalNotificationServiceExtension`, so `com.despia.myapp.OneSignalNotificationServiceExtension`, and enable Associated Domains and Push Notifications. The name is exact, Despia provisions this target at build time.
    
6.  Optional, for metrics or rich push with images and buttons: create App Group `group.com.despia.myapp.onesignal` and add it to both bundle ids. Basic push works without it.
    
7.  In OneSignal, Settings, Push and In-App, Apple iOS: upload the `.p8`, enter Key ID, Team ID, and iOS bundle id, and save.
    

### Android: Firebase

8.  Create a Firebase project at console.firebase.google.com, add an Android app with the same package name as your Despia app, and read the Server Key and Sender ID from Project settings, Cloud Messaging. Enable the Cloud Messaging API in Google Cloud first if the legacy key is hidden.
    
9.  In OneSignal, Settings, Push and In-App, Google Android: paste the Server Key and Sender ID and save.
    

### Keys and Despia

10.  In OneSignal, Settings, Keys and IDs: copy the OneSignal App ID for the client and the REST API Key for the backend. The REST API Key stays server-side, never in page scripts.
     
11.  In the Despia Editor, App, Settings, Integrations, OneSignal: add the OneSignal App ID in the iOS and Android fields (the same value in most setups), save, and trigger a fresh build. The OneSignal SDK and the notification service extension compile into the binary and are signed against your registered bundle ids, so this cannot ship over the air.
     

Until the rebuild, OneSignal stays inactive with the App ID saved: `setonesignalplayerid://` resolves silently, and backend sends return success from the API but never reach the device. If push stops after a settings change, rebuild first.

## Link the device to your user

Despia registers the device with OneSignal at launch. Your job is to tell OneSignal which user this device belongs to, on every authenticated load, so you can target that user by your own id later.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// after login, link the device to the user
function linkPush(userId) {
  if (!isDespia) return
  despia(`setonesignalplayerid://?user_id=${userId}`)
}
```

The `user_id` becomes the user's `external_id` in OneSignal, and it is exactly what you target with `include_external_user_ids` when sending. Keep it stable and identical on both ends, or the send reaches no one.

## Ask for permission without nagging

Check whether push is enabled, and send the user to their settings instead of blocking them if it is off.

```javascript
async function promptIfPushOff() {
  if (!isDespia) return
  const { nativePushEnabled } = await despia('checkNativePushPermissions://', ['nativePushEnabled'])
  if (!nativePushEnabled) despia('settingsapp://')  // opens this app's system settings
}
```

## Send from your backend

Sends run server-side so the REST API key stays private. Target the same id you linked.

```javascript
// server-side, REST API key never reaches the client
async function sendPush(userId, title, message) {
  await fetch('https://onesignal.com/api/v1/notifications', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Basic ${process.env.ONESIGNAL_REST_API_KEY}`,
    },
    body: JSON.stringify({
      app_id: process.env.ONESIGNAL_APP_ID,
      include_external_user_ids: [userId],
      headings: { en: title },
      contents: { en: message },
    }),
  })
}
```

## Open the right screen when they tap

A tap should land the user somewhere specific. OneSignal sends a `data` object with each notification, and Despia reads three fields on tap. `path` updates the URL through the History API and fires a `popstate` with no reload, which is what you want almost always. `metadata` is any JSON for restoring state. `url` is the legacy full-reload option, only when a reload is genuinely needed.

Send the deep link from the same backend call:

```javascript
// server-side send with a deep link
async function sendPush(userId, title, message, path, metadata) {
  await fetch('https://onesignal.com/api/v1/notifications', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Basic ${process.env.ONESIGNAL_REST_API_KEY}`,
    },
    body: JSON.stringify({
      app_id: process.env.ONESIGNAL_APP_ID,
      include_external_user_ids: [userId],
      headings: { en: title },
      contents: { en: message },
      data: { path, metadata },
    }),
  })
}
```

If your app reacts to `popstate`, through a small router or a listener you add, `path` navigates on its own. A plain page with no such listener will not move on a `pushState` alone, so navigate from `payload.path` in `window.onNotificationEvent`, assigned outside the `isDespia` gate. The same callback restores any `metadata` you sent:

```javascript
window.onNotificationEvent = (payload) => {
  if (payload.path) window.location.assign(payload.path)
  if (payload.metadata) {
    // object from the REST API, string from the OneSignal dashboard
    const meta = typeof payload.metadata === 'string' ? JSON.parse(payload.metadata) : payload.metadata
    restoreState(meta)
  }
}
```

If you do listen to `popstate`, drop the `path` line so you do not navigate twice.

Taps resolve from any state: foreground changes the URL immediately, background applies on resume, and a cold start buffers the payload until the page loads.

## Get it on the stores

Take your web app to iOS and Android with native push already wired in. Code signing and submission run from the browser, no Mac and no CLI.

[See the setup docs at setup.despia.com](https://setup.despia.com/native-features/onesignal/introduction)