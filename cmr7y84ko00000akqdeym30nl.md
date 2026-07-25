---
title: "React Push Notifications with OneSignal (Native)"
seoTitle: "React Push Notifications with OneSignal (Native)"
seoDescription: "React push notifications, done natively: full OneSignal, Apple, and Firebase setup with Despia, then a binding hook and backend sends."
datePublished: 2026-07-05T15:31:22.608Z
cuid: cmr7y84ko00000akqdeym30nl
slug: react-push-notifications-with-onesignal-native
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/2aff2973-4c76-426e-b59b-1aa735927707.png
tags: react, reactjs, webview, push-notifications, onesignal

---

Your React app ships to iOS and Android as a native binary through Despia, and Despia compiles the native OneSignal SDK into it, so you get real push through Apple and Google, delivered even when the app is closed. A React web app on its own cannot do that, and on iOS web push barely works. This guide is the full path: OneSignal, Apple, Firebase, and Despia, then a binding hook and backend sends.

## Why web push is not enough on mobile

Web push breaks down in two places on a real mobile app. iOS only fires it for a PWA the user added to the home screen, it landed in iOS 16.4, and it is still unreliable. And nothing arrives once the app is fully closed, which is when a notification actually counts. Native push travels over APNs on iOS and FCM on Android, wakes the device, and appears on the lock screen whether the app is open or not. Despia builds OneSignal's native SDK into the binary, so your React app gets native delivery from one codebase.

## The full setup: OneSignal, Apple, Firebase, Despia

Mostly one-time credential wiring. Go in order.

### Create the OneSignal app

1.  Make an app at onesignal.com. Free to 10,000 subscribers, so setup does not block development.
    
2.  Pick Native iOS and Native Android for platforms, not Web Push. Despia ships a native binary even though your React code is web.
    

### iOS: Apple Push Key and bundle IDs

3.  In Apple Developer, under Certificates, Identifiers and Profiles, Keys, create a key with Apple Push Notifications service (APNs) enabled. Download the `.p8` once, and note its Key ID and your Team ID.
    
4.  Under Identifiers, open your core bundle id (for example `com.despia.myapp`) and enable Push Notifications. Apple rejects registration otherwise.
    
5.  Register the notification service extension bundle id: core id plus `.OneSignalNotificationServiceExtension`, so `com.despia.myapp.OneSignalNotificationServiceExtension`, and turn on Associated Domains and Push Notifications. The name is exact, Despia provisions this target at build time.
    
6.  Optional, for metrics or rich push: create App Group `group.com.despia.myapp.onesignal` and add it to both bundle ids. Basic push works without it.
    
7.  In OneSignal, Settings, Push and In-App, Apple iOS: upload the `.p8`, enter Key ID, Team ID, and iOS bundle id, and save.
    

### Android: Firebase

8.  Create a Firebase project at console.firebase.google.com, add an Android app with the same package name as your Despia app, and read the Server Key and Sender ID from Project settings, Cloud Messaging. Enable the Cloud Messaging API in Google Cloud first if the legacy key is hidden.
    
9.  In OneSignal, Settings, Push and In-App, Google Android: paste the Server Key and Sender ID and save.
    

### Keys and Despia

10.  In OneSignal, Settings, Keys and IDs: copy the OneSignal App ID for the client and the REST API Key for the backend. The REST API Key stays server-side.
     
11.  In the Despia Editor, App, Settings, Integrations, OneSignal: add the OneSignal App ID in the iOS and Android fields (the same value in most setups), save, and trigger a fresh build. The OneSignal SDK and the notification service extension compile into the binary and are signed against your registered bundle ids, so this cannot ship over the air.
     

Until the rebuild, OneSignal stays inactive with the App ID saved: `setonesignalplayerid://` resolves silently, and backend sends return success from the API yet never reach the device. If push stops after a settings change, rebuild first.

## A hook that binds the device to the user

Despia registers the device with OneSignal on launch. You tell OneSignal which user this device belongs to, on every authenticated session, so you can target them by your own id. Wrap it in a hook.

```jsx
import { useEffect } from 'react'
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

export function usePushBinding(userId) {
  useEffect(() => {
    if (!isDespia || !userId) return
    despia(`setonesignalplayerid://?user_id=${userId}`)
  }, [userId])
}
```

Call it once high in the tree, `usePushBinding(user?.id)`. The id becomes the user's `external_id` in OneSignal, and it is exactly what you target with `include_external_user_ids` when sending. Keep it stable and identical on both ends, or the send goes to no one.

## Ask for permission without nagging

Check the real permission state and, if push is off, open settings rather than blocking the UI.

```jsx
async function promptIfPushOff() {
  if (!isDespia) return
  const { nativePushEnabled } = await despia('checkNativePushPermissions://', ['nativePushEnabled'])
  if (!nativePushEnabled) despia('settingsapp://')  // opens this app's system settings
}
```

## Send from your API route

Keep the REST API key server-side. Target the same id you bound the device to.

```javascript
// e.g. app/api/push/route.js, REST API key stays out of the bundle
export async function POST(req) {
  const { userId, title, message } = await req.json()
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
  return new Response('ok')
}
```

## Route to the right screen on tap

A tap should land the user somewhere specific. OneSignal sends a `data` object with each notification, and Despia reads three fields on tap. `path` updates the URL through the History API and fires `popstate` with no reload, the default you want. `metadata` is any JSON for restoring state. `url` is the legacy full-reload path, for when a reload is genuinely needed.

Send the deep link from the same API route:

```javascript
// app/api/push/route.js, send with a deep link
export async function POST(req) {
  const { userId, title, message, path, metadata } = await req.json()
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
  return new Response('ok')
}
```

React Router reacts to the `popstate` Despia fires, so `path` navigates on its own with no client code. Add `window.onNotificationEvent` only to act on `metadata`, assigned in an effect outside the `isDespia` gate:

```jsx
useEffect(() => {
  window.onNotificationEvent = (payload) => {
    if (!payload.metadata) return
    // object from the REST API, string from the OneSignal dashboard
    const meta = typeof payload.metadata === 'string' ? JSON.parse(payload.metadata) : payload.metadata
    restoreState(meta)
  }
}, [])
```

If a custom router ignores the synthetic `popstate`, navigate from `payload.path` in the same handler with `useNavigate` instead, but do not also rely on the automatic change, or you navigate twice.

Taps resolve from any state: foreground routes immediately, background on resume, and a cold start buffers the payload until the page loads.

## Get it on the stores

Take your React app to iOS and Android with native push already wired in. Code signing and submission run from the browser, no Mac and no CLI.

[See the setup docs at setup.despia.com](https://setup.despia.com/native-features/onesignal/introduction)