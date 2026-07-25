---
title: "Vue.js Push Notifications with OneSignal (Native)"
seoTitle: "Vue.js Push Notifications with OneSignal (Native)"
seoDescription: "Vue.js push notifications, done natively: full OneSignal, Apple, and Firebase setup with Despia, then a composable and backend sends."
datePublished: 2026-07-05T15:33:33.989Z
cuid: cmr7yaxzi00000akrge7f4atz
slug: vue-js-push-notifications-with-onesignal-native
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/1ecc0dfd-eabb-4839-b564-d81f64d8cd90.png
tags: vuejs, webview, vue, push-notifications, onesignal

---

Your Vue app ships to iOS and Android as a native binary through Despia, and Despia compiles the native OneSignal SDK into it, so you get real push through Apple and Google, delivered even when the app is closed. A Vue web app on its own cannot, and iOS web push barely works. This is the full path: OneSignal, Apple, Firebase, and Despia, then a binding composable and backend sends.

## Why web push is not enough on mobile

Web push runs into two walls on a real mobile app. iOS only fires it for a PWA the user added to the home screen, it arrived in iOS 16.4, and it stays unreliable. And nothing lands once the app is fully closed, which is the exact moment a notification matters. Native push moves over APNs on iOS and FCM on Android, wakes the device, and shows on the lock screen whether the app is open or not. Despia builds OneSignal's native SDK into the binary, so your Vue app gets native delivery from a single codebase.

## The full setup: OneSignal, Apple, Firebase, Despia

Mostly one-time credential wiring. Follow it in order.

### Create the OneSignal app

1.  Create an app at onesignal.com. Free up to 10,000 subscribers, so it does not gate development.
    
2.  Select Native iOS and Native Android for platforms, not Web Push. Despia produces a native binary even though your Vue code is web.
    

### iOS: Apple Push Key and bundle IDs

3.  In Apple Developer, under Certificates, Identifiers and Profiles, Keys, create a key with Apple Push Notifications service (APNs) enabled. Download the `.p8` once, and note its Key ID and your Team ID.
    
4.  Under Identifiers, open your core bundle id (for example `com.despia.myapp`) and enable Push Notifications. Registration fails without it.
    
5.  Register the notification service extension bundle id: core id plus `.OneSignalNotificationServiceExtension`, giving `com.despia.myapp.OneSignalNotificationServiceExtension`, and enable Associated Domains and Push Notifications. The name is exact, Despia provisions this target at build time.
    
6.  Optional, for metrics or rich push: create App Group `group.com.despia.myapp.onesignal` and attach it to both bundle ids. Basic push does not need it.
    
7.  In OneSignal, Settings, Push and In-App, Apple iOS: upload the `.p8`, enter Key ID, Team ID, and iOS bundle id, and save.
    

### Android: Firebase

8.  Create a Firebase project at console.firebase.google.com, add an Android app under the same package name as your Despia app, and read the Server Key and Sender ID from Project settings, Cloud Messaging. Enable the Cloud Messaging API in Google Cloud first if the legacy key is hidden.
    
9.  In OneSignal, Settings, Push and In-App, Google Android: paste the Server Key and Sender ID and save.
    

### Keys and Despia

10.  In OneSignal, Settings, Keys and IDs: copy the OneSignal App ID for the client and the REST API Key for the backend. The REST API Key stays server-side.
     
11.  In the Despia Editor, App, Settings, Integrations, OneSignal: add the OneSignal App ID in the iOS and Android fields (the same value in most setups), save, and trigger a fresh build. The OneSignal SDK and the notification service extension compile into the binary and are signed against your registered bundle ids, so this cannot arrive over the air.
     

Until the rebuild, OneSignal stays inactive with the App ID saved: `setonesignalplayerid://` resolves silently, and backend sends return success from the API but never reach the device. If push stops after a settings change, rebuild first.

## A composable that binds the device to the user

Despia registers the device with OneSignal on launch. You tell OneSignal which user owns the device, on every authenticated session, so you can target them by your own id. Wrap it in a composable.

```javascript
import { watch } from 'vue'
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

export function usePushBinding(userId) {
  // userId is a ref, re-bind whenever it resolves or changes
  watch(userId, (id) => {
    if (!isDespia || !id) return
    despia(`setonesignalplayerid://?user_id=${id}`)
  }, { immediate: true })
}
```

Call it once in your root setup, `usePushBinding(userIdRef)`. The id becomes the user's `external_id` in OneSignal, and it is exactly what you target with `include_external_user_ids` when sending. Keep it stable and identical on both ends, or the send reaches no one.

## Ask for permission without nagging

Check the real permission state, and open settings instead of blocking the UI if push is off.

```javascript
async function promptIfPushOff() {
  if (!isDespia) return
  const { nativePushEnabled } = await despia('checkNativePushPermissions://', ['nativePushEnabled'])
  if (!nativePushEnabled) despia('settingsapp://')  // opens this app's system settings
}
```

## Send from your backend

Keep the REST API key on the server. Target the same id you bound the device to.

```javascript
// server-side handler, REST API key never reaches the client
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

## Route to the right screen on tap

A tap should land the user somewhere specific. OneSignal sends a `data` object with each notification, and Despia reads three fields on tap. `path` updates the URL through the History API and fires `popstate` with no reload, the default. `metadata` is any JSON for restoring state. `url` is the legacy full-reload option, only when a reload is genuinely needed.

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

Vue Router reacts to the `popstate` Despia fires, so `path` navigates on its own with no client code. Add `window.onNotificationEvent` only to restore state from `metadata`, assigned in `onMounted` outside the `isDespia` gate:

```javascript
import { onMounted } from 'vue'

onMounted(() => {
  window.onNotificationEvent = (payload) => {
    if (!payload.metadata) return
    // object from the REST API, string from the OneSignal dashboard
    const meta = typeof payload.metadata === 'string' ? JSON.parse(payload.metadata) : payload.metadata
    restoreState(meta)
  }
})
```

If a custom router ignores the synthetic `popstate`, navigate from `payload.path` in the same handler with `useRouter` instead, but never both, or you navigate twice.

Taps resolve from any state: foreground routes at once, background on resume, and a cold start buffers the payload until the page is ready.

## Get it on the stores

Take your Vue app to iOS and Android with native push already wired in. Code signing and submission run from the browser, no Mac and no CLI.

[See the setup docs at setup.despia.com](https://setup.despia.com/native-features/onesignal/introduction)