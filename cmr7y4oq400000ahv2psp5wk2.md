---
title: "Lovable Push Notifications with OneSignal (Native)"
seoTitle: "Lovable Push Notifications with OneSignal (Native)"
seoDescription: "Lovable push notifications, done natively: full OneSignal, Apple, and Firebase setup with Despia, then link users and send from an edge function."
datePublished: 2026-07-05T15:28:42.050Z
cuid: cmr7y4oq400000ahv2psp5wk2
slug: lovable-push-notifications-with-onesignal-native
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/86a0ae85-85ff-41df-8af0-f097f84f5c10.png
tags: webview, push-notifications, onesignal, lovable

---

Lovable builds the app, and it may scaffold a Capacitor push plugin, but if you are shipping with Despia you do not need it. Despia compiles the native OneSignal SDK into your iOS and Android binaries, so your Lovable app sends real notifications through Apple and Google, delivered even when the app is closed. Here is the whole setup: OneSignal, Apple, Firebase, and Despia, then linking users and sending from your Supabase or Lovable Cloud backend.

## Why web push falls short on a mobile app

A web page has two dead ends on a real mobile app. iOS web push only fires for a PWA the user added to the home screen by hand, it arrived only in iOS 16.4, and it stays flaky. And once the browser or app is fully closed, a web page receives nothing, which is precisely when a notification matters. Native push runs over APNs on iOS and FCM on Android, wakes the device, and shows on the lock screen regardless of whether your app is open. Despia builds OneSignal's native SDK into the binary, so you get native delivery and your Lovable app stays one codebase.

## The full setup: OneSignal, Apple, Firebase, Despia

This part is mostly one-time credential wiring. Work through it in order.

### Create the OneSignal app

1.  Create an app at onesignal.com. The free tier covers 10,000 subscribers, so it will not block your first release.
    
2.  Choose Native iOS and Native Android when prompted for platforms. Skip Web Push, even though your Lovable code is web, because Despia produces a native binary.
    

### iOS: Apple Push Key and bundle IDs

3.  In Apple Developer, under Certificates, Identifiers and Profiles, Keys, generate a key with Apple Push Notifications service (APNs) enabled. Save the `.p8` (single download only), and record its Key ID and your Team ID.
    
4.  Open your core bundle id under Identifiers (for example `com.despia.myapp`) and switch on the Push Notifications capability. Apple refuses registration without it.
    
5.  Register a second bundle id for the notification service extension, your core id plus `.OneSignalNotificationServiceExtension`, giving `com.despia.myapp.OneSignalNotificationServiceExtension`, and enable Associated Domains and Push Notifications on it. Keep the name exact, since Despia provisions this target during the build.
    
6.  Optional, for metrics or rich push with images and buttons: create an App Group `group.com.despia.myapp.onesignal` and attach it to both bundle ids. Standard push does not need it.
    
7.  In OneSignal, go to Settings, Push and In-App, Apple iOS, upload the `.p8`, enter the Key ID, Team ID, and iOS bundle id, and save. OneSignal checks the credentials against Apple on the spot.
    

### Android: Firebase

8.  Create a Firebase project at console.firebase.google.com and add an Android app under the same package name as your Despia app. In Project settings, Cloud Messaging, copy the Server Key and Sender ID. Enable the Cloud Messaging API in Google Cloud first if the legacy key is hidden.
    
9.  In OneSignal, under Settings, Push and In-App, Google Android, paste the Server Key and Sender ID and save.
    

### Keys and Despia

10.  In OneSignal, open Settings, Keys and IDs. Take the OneSignal App ID for the client and the REST API Key for the backend. The REST API Key belongs on the server only.
     
11.  In the Despia Editor, open App, Settings, Integrations, OneSignal, add your OneSignal App ID in the iOS and Android fields (the same value in most setups), and save. Then run a fresh build. The OneSignal SDK and the notification service extension are compiled into the binary and signed against the bundle ids you registered, so an over-the-air update cannot activate them.
     

Until that rebuild, OneSignal stays dormant even with the App ID in place. The `setonesignalplayerid://` call resolves silently, and backend sends succeed at the API and never arrive on the device. If push stops after a settings change, rebuild first.

## Link the device to your user

Despia registers the device with OneSignal on launch. You tell OneSignal which user owns the device, on every authenticated render, so you can target that user by your own id later. Lovable generates React, so this lives in an effect.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// bind the device to the logged-in user whenever auth changes
useEffect(() => {
  if (!isDespia || !user) return
  despia(`setonesignalplayerid://?user_id=${user.id}`)
}, [user])
```

The id you pass becomes the user's `external_id` in OneSignal, and it is exactly what you target with `include_external_user_ids` when sending. Keep it stable and identical on both ends, or the notification lands nowhere.

## Ask for permission without nagging

Check whether push is on, and route the user to settings rather than blocking them if it is off.

```javascript
async function promptIfPushOff() {
  if (!isDespia) return
  const { nativePushEnabled } = await despia('checkNativePushPermissions://', ['nativePushEnabled'])
  if (!nativePushEnabled) despia('settingsapp://')  // opens this app's system settings
}
```

## Send from an edge function

Lovable ships a backend, Supabase or Lovable Cloud, so sends slot into an edge function, which keeps the REST API key server-side. Target the same id you linked.

```javascript
// edge function, REST API key stays on the server
import { serve } from 'https://deno.land/std/http/server.ts'

serve(async (req) => {
  const { userId, title, message } = await req.json()
  await fetch('https://onesignal.com/api/v1/notifications', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Basic ${Deno.env.get('ONESIGNAL_REST_API_KEY')}`,
    },
    body: JSON.stringify({
      app_id: Deno.env.get('ONESIGNAL_APP_ID'),
      include_external_user_ids: [userId],
      headings: { en: title },
      contents: { en: message },
    }),
  })
  return new Response('ok')
})
```

## Open the right screen when they tap

A notification earns its tap only if it lands the user somewhere specific. OneSignal ships a `data` object with each notification, and Despia reads three fields on tap. `path` updates the URL through the History API and fires a `popstate` with no reload, the option you want almost always. `metadata` is any JSON for restoring state. `url` is the legacy full-reload option, for when a reload is actually needed.

Attach the deep link to the same edge function send:

```javascript
// edge function, send with a deep link
serve(async (req) => {
  const { userId, title, message, path, metadata } = await req.json()
  await fetch('https://onesignal.com/api/v1/notifications', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Basic ${Deno.env.get('ONESIGNAL_REST_API_KEY')}`,
    },
    body: JSON.stringify({
      app_id: Deno.env.get('ONESIGNAL_APP_ID'),
      include_external_user_ids: [userId],
      headings: { en: title },
      contents: { en: message },
      data: { path, metadata },
    }),
  })
  return new Response('ok')
})
```

React Router reacts to the `popstate` Despia fires, so `path` navigates on its own with no client code. Add `window.onNotificationEvent` only to restore state from `metadata`, assigned outside the `isDespia` gate:

```javascript
useEffect(() => {
  window.onNotificationEvent = (payload) => {
    if (!payload.metadata) return
    // object from the REST API, string from the OneSignal dashboard
    const meta = typeof payload.metadata === 'string' ? JSON.parse(payload.metadata) : payload.metadata
    restoreState(meta)
  }
}, [])
```

If a custom router ignores the synthetic `popstate`, navigate from `payload.path` in the same handler instead of leaning on the automatic change, but never both.

Taps resolve from any state: foreground updates the route at once, background applies on resume, and a cold start buffers the payload until the page is ready.

## Get it on the stores

Take your Lovable app to iOS and Android with native push already in place. Code signing and submission run from the browser, no Mac and no CLI.

[See the setup docs at setup.despia.com](https://setup.despia.com/native-features/onesignal/introduction)