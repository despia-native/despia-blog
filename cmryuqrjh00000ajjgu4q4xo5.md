---
title: "Base44 Push Notifications, Done Natively"
seoTitle: "Base44 Push Notifications, Done Natively"
seoDescription: "Base44 cannot send native push on its own. Here is the full APNs, FCM, and OneSignal setup to add iOS and Android push to a Base44 app with Despia."
datePublished: 2026-07-24T11:23:40.420Z
cuid: cmryuqrjh00000ajjgu4q4xo5
slug: base44-push-notifications-done-natively
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/40d99201-1f29-4cf4-a0e4-5cf968b7d5fa.png
tags: push-notifications, onesignal, base44

---

Base44 builds the web app, but it cannot send a push notification to a phone that has the app closed. It is one of the most requested features on their feedback board, and Base44's own guidance says plainly that push is not currently supported. The reason is not a Base44 oversight, it is that push lives below the web layer, in the part of the app a browser never touches. Ship your Base44 app with Despia and that layer is there: real APNs and FCM push, targeted at a specific user, from the same codebase you already built.

## Why Base44 cannot send push on its own

Base44 is good at the thing it does. You describe an app, it generates a React and Tailwind web app with a database, auth, and backend functions, and it runs in a browser. The catch is that notifications live below the browser. Web push, the browser version, needs a Service Worker registered at the root of your domain, and Base44 gives you no root-level static file access: no `public/` directory and no way to serve `/sw.js`. A service worker can only be loaded through a `blob:` URL or a backend function, which is unreliable and out of scope for real background push, and Base44's own guidance says push is not currently supported. Native push is a separate mechanism again, and Base44 has no native layer to receive an APNs or FCM device token in the first place.

Even where web push does register, it does nothing useful on a real phone. iOS only delivers it to a PWA the user added to the home screen by hand, and once any app is fully closed a web page receives nothing, which is exactly when a notification earns its place. Native push is a different mechanism. It runs over APNs on iOS and FCM on Android, wakes the device, and reaches the lock screen whether the app is open or not.

## What native push actually requires

Three things have to line up. Apple's APNs and Google's FCM have to accept your app and hand it a device token. Something has to map that token to a user you can name, so you are sending to a person and not a random device. And your backend has to be able to call the push service to send. Despia compiles the OneSignal native SDK into your binary, so the device registers on launch, and the mapping and sending come down to one client call and one backend call. The rest of this page is the credential wiring that makes those two calls work.

## The setup, in order

This is mostly one-time credential wiring across four places: OneSignal, Apple, Firebase, and Despia. Take it in order, because each step feeds the next.

### 1\. Create the OneSignal app

Sign up at onesignal.com and create an app. The free plan covers mobile push, so it does not gate development while you build. When it asks for platforms, choose Native iOS and Native Android, not Web Push. Despia produces a native binary even though your code is web, so the native platforms are the correct ones.

### 2\. iOS: the APNs key and bundle id capabilities

Apple push needs one auth key and two correctly configured bundle ids.

1.  In Apple Developer, under Certificates, Identifiers and Profiles, Keys, create a key with the Apple Push Notifications service (APNs) enabled. Download the `.p8` once, because Apple only lets you download it a single time, and note its Key ID and your Team ID.
    
2.  Under Identifiers, open your core bundle id (for example `com.despia.myapp`) and enable the Push Notifications capability. Apple rejects device registration without it.
    
3.  Register a second bundle id for the notification service extension: your core id plus `.OneSignalNotificationServiceExtension`, so `com.despia.myapp.OneSignalNotificationServiceExtension`. Enable Associated Domains and Push Notifications on it. The name is exact, and Despia provisions this target at build time, so it has to match.
    
4.  Optional, only if you want delivery metrics or rich push with images and buttons: create an App Group `group.com.despia.myapp.onesignal` and add it to both bundle ids. Basic push works without it.
    
5.  Back in OneSignal, under Settings, Push and In-App, Apple iOS: upload the `.p8`, enter the Key ID, Team ID, and your iOS bundle id, and save.
    

### 3\. Android: the Firebase Service Account key (FCM V1)

Google retired the legacy FCM Server Key, so the current path is a Firebase Service Account JSON. The older Server Key and Sender ID fields no longer authorize sends.

1.  At console.firebase.google.com, create a project (or pick an existing one) and add an Android app with the same package name as your Despia app.
    
2.  In Project settings, Cloud Messaging, confirm the Firebase Cloud Messaging API (V1) is enabled. If it is off, open it in the Google Cloud Console and click Enable, then wait a minute for it to propagate.
    
3.  In Project settings, Service accounts, click Generate new private key and confirm. This downloads a JSON file. Keep it secure, it is a credential. The default service account already carries the `cloudmessaging.messages.create` and `firebase.projects.get` permissions OneSignal needs.
    
4.  In OneSignal, under Settings, Push and In-App, Google Android (FCM): upload that Service Account JSON and save.
    

### 4\. Wire OneSignal into Despia and rebuild

Two OneSignal values connect the app. Under Settings, Keys and IDs, copy the OneSignal App ID for the client and the REST API Key for the backend. The REST API Key is server-side only and never goes in a page script.

In the Despia Editor, under App, Settings, Integrations, OneSignal, toggle the integration on and paste the OneSignal App ID. If you created the optional App Group, paste `group.com.despia.myapp.onesignal` as well. Save, then trigger a fresh build. The OneSignal SDK and the notification service extension compile into the binary and are signed against your registered bundle ids, so this part cannot ship over the air.

There is a failure here worth stating on its own, because it costs people a launch. Until you rebuild, OneSignal stays inactive even with the App ID saved. The client call resolves silently, and backend sends return success from the API but never reach a device. There is no error at build time and no crash, just silence. If push stops working right after a settings change, rebuild before you debug anything else.

## Link the device to your user

Despia registers the device with OneSignal at launch. Your job is to tell OneSignal which user this device belongs to, on every authenticated load, so you can target that user by your own id later. Gate it behind the runtime check so it never runs in a normal browser.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// Run after login, on every authenticated load.
// Sets this device's OneSignal external_id to your own user id.
function linkPush(userId) {
  if (!isDespia) return
  despia(`setonesignalplayerid://?user_id=${userId}`)
}
```

The `user_id` becomes the user's `external_id` in OneSignal, and it is exactly what you target when sending. Keep it stable and identical on both ends, or the send reaches no one.

## The one thing push needs: a stable user id

The push call is only as good as the id you pass it. Send `undefined` or a different id each session and you can never reach that user again. So you need one durable id per user that the client and the backend both agree on.

Base44's built-in auth already gives you one. Every signed-in Base44 user has a user id, so you pass that straight to `setonesignalplayerid` on the client and target the same id from a Base44 backend function when you send. The built-in backend is enough for this. You do not need a custom backend or your own account layer just to run push.

A custom account layer earns its place elsewhere, mainly native sign-in. Google blocks OAuth inside a WebView, so native Google or Apple login needs the `oauth://` bridge and an account you control, and that is where you take auth into your own hands. That is a separate feature, not a push requirement. For targeting a notification, the built-in Base44 user id does the job.

The one case where you supply the id yourself is an app with no login at all. There you generate a UUID on first launch, keep it in the Despia Storage Vault, and reuse it as the target id, which the [Base44 loginless setup](https://blog.despia.com/base44-loginless-mobile-app-anonymous-login) covers end to end. That is about not having accounts, not about push needing a custom backend.

## Ask for permission without nagging

Check whether push is on, and route the user to their settings instead of blocking them if it is off.

```javascript
async function promptIfPushOff() {
  if (!isDespia) return
  const { nativePushEnabled } = await despia('checkNativePushPermissions://', ['nativePushEnabled'])
  if (!nativePushEnabled) despia('settingsapp://')  // opens this app's system settings
}
```

## Send from your Base44 backend

Sends run server-side in a Base44 backend function so the REST API Key stays private. Target the same id you linked on the device. Because targeting is by your own user id, there are no device tokens to manage.

```javascript
// Base44 backend function. Sends to one user by the id you mapped on the client.
async function sendPush(userId, title, message, path, metadata) {
  const res = await fetch('https://onesignal.com/api/v1/notifications', {
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
      data: { path, metadata },  // deep link, see below
    }),
  })

  if (!res.ok) throw new Error(`OneSignal ${res.status}`)
  return res.json()
}
```

The `include_external_user_ids` value is the same `userId` you set with `setonesignalplayerid`, which is why the client mapping has to run on every authenticated load. A user who reinstalls or signs in on a new device re-registers under the same id and keeps receiving your sends.

## Open the right screen when they tap

A notification that dumps the user on the home screen wastes the tap, and this is where a Base44 app has it easy. Send a `path` in the notification's `data` object, and Despia applies it through the History API on tap, calling `pushState` and firing `popstate`. Because Base44 builds a React single-page app, its router reacts to that `popstate` and navigates on its own. You write no client code for this.

```javascript
// The backend send from earlier, with a route to open on tap
data: { path: '/account/orders/4567?tab=tracking' }
```

Taps resolve from any app state: a foreground tap changes the screen immediately, a background tap applies on resume, and a cold start buffers the route until the page loads.

You only add a handler for two cases: a router that ignores a synthetic `popstate`, or restoring state from a `metadata` object you sent. Then define `window.onNotificationEvent`, a global callback assigned outside the `isDespia` gate.

```javascript
// Optional: only if your router ignores popstate, or you send metadata
window.onNotificationEvent = (payload) => {
  if (payload.path) router.navigate(payload.path)
  if (payload.metadata) {
    // object from the REST API, string from the OneSignal dashboard
    const meta = typeof payload.metadata === 'string' ? JSON.parse(payload.metadata) : payload.metadata
    restoreState(meta)
  }
}
```

## Let the Base44 agent write the calls for you

Base44 builds by prompt, and it cannot npm-install a native package, so left alone its AI tries to scaffold the web push the platform blocks. Give it the Despia MCP instead and it writes the correct `despia()` calls directly. Paste the MCP URL into Base44 under Settings, Account, MCP connections:

```text
https://setup.despia.com/mcp
```

After that, a prompt like "register this user for push on login and send them a welcome notification from the backend" produces the gated client call and the OneSignal send against the real API, instead of a guess.

## What ships, and what you resubmit

Push is a native capability, so it works in the store build and not in a plain browser tab, which is the point. The OneSignal SDK compiles into the binary, so turning it on is the one thing here that needs a rebuild rather than an over-the-air push. Everything on the web side of your Base44 app still updates over the air through remote hydration, with no App Store resubmission. After the push wiring is in, sending is just a backend call, and you only submit a new binary when you add another genuinely new native capability.

## Add this to your app

Every native capability in this post is one JavaScript call away, inside your existing Base44 codebase. No Xcode, no native project to maintain.

[Read the full reference at setup.despia.com](https://setup.despia.com/native-features/onesignal/introduction)