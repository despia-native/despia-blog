---
title: "Lovable Push Notifications, Done Natively"
seoTitle: "Lovable Push Notifications, Done Natively"
seoDescription: "Lovable ships a web app, not native push. Here is the full OneSignal, APNs, and FCM setup to add iOS and Android push to a Lovable app with Despia."
datePublished: 2026-07-25T08:41:21.777Z
cuid: cms04dvtp00000akr2qdf8fn8
slug: lovable-push-notifications-done-natively
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/a440ba74-0ffc-47f4-9049-f610dd36a534.png
tags: android-app-development, webapp, pwa, ios-app-development, onesignal, lovable

---

Lovable and Supabase get you a working product fast: a React app with real auth, a database, and backend functions. What they do not get you is a push notification landing on a phone that has the app closed, which is usually the entire reason to have a mobile app. Ship your Lovable app with Despia and that layer is there: real APNs and FCM push, targeted at the same Supabase users you already have, sent from a Supabase Edge Function.

## Why Lovable cannot send push on its own

Lovable builds a web app. Its native option is a Capacitor export, and that is a real native project, but it comes with a tax. You need a Mac, Xcode, and signing certificates to build it, the export ships with no billing and no push pipeline, and Capacitor serves your app offline from the `file://` protocol. That last one is the quiet problem: `file://` breaks Supabase auth, OAuth redirects, and Sign in with Apple, because those flows expect an http origin. So the moment you go native the Capacitor way, the auth you built on Supabase starts failing in the exact place users notice.

A progressive web app is the other route, and it is worse for push. iOS only delivers web push to a PWA the user added to the home screen by hand, and no web page receives anything once the app is fully closed. Native push is a different mechanism. It runs over APNs on iOS and FCM on Android, wakes the device, and reaches the lock screen whether the app is open or not. That is what Despia adds, without the `file://` problem, because it loads your live Lovable URL through remote hydration and keeps Supabase auth working exactly as it does on the web.

## What native push needs

Three things have to line up. APNs and FCM have to accept your app and issue a device token. Something has to map that token to a user you can name. And a backend has to call the push service to send. Despia compiles the OneSignal SDK into the binary and registers the device at launch, so the token side is handled. The user you target is your Supabase user, and the backend that sends is a Supabase Edge Function. You already own both.

## The setup, in order

This is mostly one-time credential wiring across OneSignal, Apple, Firebase, and Despia. Take it in order.

### 1\. Create the OneSignal app

Sign up at onesignal.com and create an app. The free plan covers mobile push, so it does not gate development. When it asks for platforms, choose Native iOS and Native Android, not Web Push. Despia produces a native binary even though your code is web, so the native platforms are correct.

### 2\. iOS: the APNs key and bundle id capabilities

1.  In Apple Developer, under Certificates, Identifiers and Profiles, Keys, create a key with the Apple Push Notifications service (APNs) enabled. Download the `.p8` once, since Apple only allows one download, and note its Key ID and your Team ID.
    
2.  Under Identifiers, open your core bundle id (for example `com.acme.myapp`, the value in your Despia project settings) and enable the Push Notifications capability. Apple rejects registration without it.
    
3.  Register a second bundle id for the notification service extension: your core id plus `.OneSignalNotificationServiceExtension`, so `com.acme.myapp.OneSignalNotificationServiceExtension`. Enable Associated Domains and Push Notifications on it. Despia provisions this target at build time, so the name has to match exactly.
    
4.  Optional, for rich push with images and buttons or delivery metrics: create an App Group `group.com.acme.myapp.onesignal` and add it to both bundle ids.
    
5.  In OneSignal, under Settings, Push and In-App, Apple iOS, upload the `.p8`, enter the Key ID, Team ID, and your iOS bundle id, and save.
    

### 3\. Android: the Firebase Service Account key (FCM V1)

Google retired the legacy FCM Server Key, so the current path is a Firebase Service Account JSON. The old Server Key and Sender ID fields no longer authorize sends.

1.  At console.firebase.google.com, create or pick a project and add an Android app with the same package name as your Despia app.
    
2.  In Project settings, Cloud Messaging, confirm the Firebase Cloud Messaging API (V1) is enabled.
    
3.  In Project settings, Service accounts, click Generate new private key. This downloads a JSON file. Keep it secure.
    
4.  In OneSignal, under Settings, Push and In-App, Google Android (FCM), upload that Service Account JSON and save.
    

### 4\. Wire OneSignal into Despia and rebuild

Under OneSignal Settings, Keys and IDs, copy the OneSignal App ID for the client and the REST API Key for the backend. The REST API Key is server-side only and belongs in your Supabase secrets, never in page code.

In the Despia Editor, under App, Settings, Integrations, OneSignal, toggle the integration on and paste the OneSignal App ID. If you created the App Group, paste `group.com.acme.myapp.onesignal` as well. Save, then trigger a fresh build.

One failure is worth stating on its own. The OneSignal SDK compiles into the binary, so until you rebuild after saving the App ID, OneSignal stays inactive. The client call resolves silently and backend sends return success but reach no device, with no build error and no crash. If push stops right after a settings change, rebuild before you debug anything else.

## Link the device to your Supabase user

This is where Lovable is simpler than most stacks. You already have a real authenticated user, so there is no separate account layer to build. Read the Supabase user and pass its id to OneSignal on every authenticated load, gated behind the runtime check so it never runs in a normal browser.

```javascript
import despia from 'despia-native'
import { supabase } from '@/integrations/supabase/client'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// Run after auth resolves, on every authenticated load.
async function linkPush() {
  if (!isDespia) return
  const { data: { user } } = await supabase.auth.getUser()
  if (user) despia(`setonesignalplayerid://?user_id=${user.id}`)
}
```

The Supabase `user.id` becomes the user's `external_id` in OneSignal, and it is the same id your Edge Function targets when sending. It is stable across reinstalls and new devices, because it is the account, not the device.

## Send from a Supabase Edge Function

Sends run server-side so the REST API Key stays private, and a Supabase Edge Function is the natural home. Target the same Supabase user id you linked on the client.

```javascript
// supabase/functions/send-push/index.ts
import { serve } from 'https://deno.land/std/http/server.ts'

serve(async (req) => {
  // Verify the caller before sending, so a user cannot push to someone else
  const { userId, title, message, path } = await req.json()

  const res = await fetch('https://onesignal.com/api/v1/notifications', {
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
      data: { path },
    }),
  })

  return new Response(await res.text(), { status: res.status })
})
```

Set the two secrets with `supabase secrets set ONESIGNAL_REST_API_KEY=... ONESIGNAL_APP_ID=...` and deploy. Because targeting is by your own Supabase id, there are no device tokens to store. A common pattern is to call this function from a database trigger or webhook, for example sending a confirmation after a RevenueCat purchase writes to Supabase.

## Open the right screen when they tap

Send the route inside the `data` object as `path`, and Despia applies it through the History API on tap, calling `pushState` and firing `popstate`. Because Lovable builds a React single-page app, React Router reacts to that `popstate` and navigates on its own, with no client code.

The mistake to avoid is the top-level `url` field. OneSignal offers it for web push, and it is the first thing people reach for, but a native Despia app never reads routing from it. Put a route there and the app opens at your start URL every time, which is the home-screen symptom people report as broken push. Nothing is broken, the route is just in a field the runtime does not read. Keep it in `data.path`.

For routers that ignore a synthetic `popstate`, or to restore state from a `metadata` object, define `window.onNotificationEvent`, a global callback assigned outside the `isDespia` gate.

```javascript
// Optional: only if your router ignores popstate, or you send metadata
window.onNotificationEvent = (payload) => {
  if (payload.path) navigate(payload.path)  // your React Router navigate
}
```

## Let Lovable's AI write the calls

Lovable builds by prompt, and left alone it will try to scaffold web push that does not survive the store. Add the Despia MCP as a custom chat connector instead, using the New MCP server entry in Lovable's chat surface, and point it at:

```text
https://setup.despia.com/mcp
```

With that context, a prompt like "register the logged-in Supabase user for push on load and send them a welcome notification from an Edge Function" produces the gated client call and the Edge Function against the real API, rather than a guess.

## What ships, and what you resubmit

Push is a native capability, so it works in the store build and not in a plain browser tab. The OneSignal SDK compiles into the binary, so enabling it is the one step here that needs a rebuild. Everything on the web side of your Lovable app still updates over the air through remote hydration, with no App Store resubmission and with Supabase auth intact, because Despia never touches `file://`. After the wiring is in, sending is one Edge Function call.

## Add this to your app

Every native capability in this post is one JavaScript call away, inside your existing Lovable codebase. No Xcode, no native project to maintain.

[Read the full reference at setup.despia.com](https://setup.despia.com/native-features/onesignal/introduction)