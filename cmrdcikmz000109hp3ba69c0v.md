---
title: "Despia Push Notifications: The Complete OneSignal Guide"
seoTitle: "Despia Push Notifications: The Complete OneSignal Guide"
seoDescription: "Everything that breaks when setting up Despia push notifications with OneSignal, the reason behind each error, and the exact fix for every scenario."
datePublished: 2026-07-09T10:10:15.446Z
cuid: cmrdcikmz000109hp3ba69c0v
slug: despia-push-notifications-the-complete-onesignal-guide
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/65911570-e8c3-44ab-9233-b6ce2e1d6eb2.png
tags: mobile-apps, push-notifications, native-apps, fcm, onesignal, apn

---

Push is the single feature most builders get wrong on their first try, and it is almost never because Despia or OneSignal is broken. It is because the setup has half a dozen independent parts that all have to be right, and most people wire them up from AI advice that is out of date, aimed at web push, or invents functions that do not exist. The result is the same three symptoms over and over: no devices in OneSignal, devices with no External ID, or notifications that send successfully from the dashboard and never arrive on the phone.

This guide walks the setup the way it actually works, then goes through every failure mode one at a time with the real cause and the fix. If your push is not working, you can skip to the symptom that matches and you will find it.

## How push actually works in Despia

Get the mental model right and most of the errors explain themselves.

Despia bundles the native OneSignal SDK into your app at build time. When the app launches, Despia registers the device with OneSignal for you. You do not write registration code, you do not install a service worker, and you do not touch APNs or FCM tokens in your app. That part is handled.

Your job is two calls, both from your existing web codebase:

*   Link the logged-in user to their device so you can target them later.
    
*   Optionally, check or request notification permission.
    

Everything else, the actual sending, happens from your backend against OneSignal's REST API. That split is the thing to hold onto: the device and the identity live in the app, the sending lives on your server.

OneSignal moved away from Player IDs toward `external_id`, which is your own user ID from your own database. Despia supports this directly. You pass your user's ID once they are logged in, and OneSignal stores the mapping. When you want to notify that user, your backend targets them by the same ID with `include_external_user_ids`. If you are reading AI answers that talk about Player IDs, subscription IDs, or `getUserId()`, they are describing an older OneSignal and you can ignore them.

## The correct way to link a user

This is the call that associates a device with your user. It has to run on every authenticated load, after you know the user is logged in, not once at signup.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// run on every authenticated load, once you have a real user id
function identifyUser(userId) {
  if (!isDespia || !userId) return
  despia(`setonesignalplayerid://?user_id=${userId}`)
}

identifyUser(currentUser.id)
```

The value you pass becomes the `external_id` in OneSignal. That is the same string you target from your backend. Two rules decide whether this works:

*   It runs every time a logged-in user opens the app, not only the first time. Devices change, sessions change, and OneSignal needs the link refreshed.
    
*   It runs after login is confirmed and `userId` is a real value. Firing it on a public page or mid-redirect passes nothing, and nothing links.
    

Calling this a single time at onboarding is the most common reason External IDs never appear. If you take one thing from this post, take that.

## The correct permission flow

Despia can request permission automatically at launch, or you can control the prompt yourself.

If Automatic Prompt is on, the OS asks on first launch and you do nothing. If it is off, you trigger the prompt at the point in your flow that makes sense:

```javascript
// only when Automatic Prompt is off in the Despia dashboard
if (isDespia) despia('registerpush://')
```

For most apps, turning the automatic prompt off and asking after a short priming screen converts far better than a cold system prompt on first open. Explain why you send notifications, then request. Once a user denies on iOS, you cannot ask again in-app, you can only send them to Settings, so the first ask matters.

You can check the current state at any time and route a denied user to their settings:

```javascript
async function checkPush() {
  if (!isDespia) return
  const result = await despia('checkNativePushPermissions://', ['nativePushEnabled'])
  if (!result.nativePushEnabled) {
    despia('settingsapp://')
  }
}
```

## Sending from your backend

Notifications are sent server-side with your REST API key, never from the client. Target a specific user with the same ID you linked.

```javascript
await fetch('https://onesignal.com/api/v1/notifications', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Basic ${process.env.ONESIGNAL_REST_API_KEY}`,
  },
  body: JSON.stringify({
    app_id: process.env.ONESIGNAL_APP_ID,
    include_external_user_ids: [userId],
    headings: { en: 'Your order shipped' },
    contents: { en: 'Tap to track it' },
  }),
})
```

Keep the REST API key on the server. If it is ever in client code, it is compromised.

## Now the failures, one at a time

Open OneSignal, go to Audience then Subscriptions, and look at the device list. What you see there tells you which class of problem you have, and the classes have completely different fixes.

### Symptom: no devices appear in OneSignal at all

The SDK never registered the device. There is nothing for an External ID to attach to, so no amount of linking code will help until this is fixed. Work through these in order.

**You added the App ID in Despia but did not rebuild.** OneSignal is compiled into the binary, so it cannot ship over the air. Adding the App ID to the dashboard does nothing on its own. You have to trigger a fresh build after adding it.

**You rebuilt but did not bump the version, or you are testing the old build.** This one catches almost everyone. In Despia, raise the version under App then Settings then Versioning, for example from 1.0.0 to 1.0.1, before you rebuild. Then install that new build on your device and test on that one. If your phone still has the old version installed, you are testing a binary that never had OneSignal in it, no matter what the dashboard says.

**On iOS, the Push Notifications capability is not enabled on your bundle ID.** Despia bundles the SDK and provisions the notification service extension, but enabling the Push Notifications capability on your core bundle ID is done by you in Apple Developer Console. APNs rejects registration without it even when your key and OneSignal config are correct. Go to developer.apple.com, Identifiers, open your core bundle ID, tick Push Notifications under Capabilities, save, then rebuild.

**On Android, Firebase is not wired up.** Android push runs through Firebase Cloud Messaging. If the FCM credentials are wrong or missing, the device never gets a token and never appears in OneSignal. Confirm there is an Android app in your Firebase project using the exact same package name as your Despia app, and that the Server Key and Sender ID are pasted into OneSignal under Settings, Push and In-App, Google Android.

**The OneSignal App ID in Despia is wrong.** Copy it fresh from OneSignal, Settings, Keys and IDs, paste it into Despia, and rebuild. One wrong character means the device registers against nothing.

**You set the OneSignal app up as Web Push.** Despia apps are native, so the OneSignal platform has to be Native iOS and Native Android, not the Web Push option, even though your app code is web. If you followed a web push tutorial and created a service worker, none of that applies here.

### Symptom: devices appear, but none have an External ID

The SDK is working. The link between device and user is the only thing missing, and it is always one of three causes.

**The link only runs once.** It fires at signup and never again. It needs to run on every authenticated load. See the linking section above.

**The link runs before login.** If you call it on a public route or during a redirect, there is no user ID yet, so nothing links. Only call it after login is confirmed.

**The user ID is empty when it runs.** Log the value right before the call. If it prints `undefined` or a blank string, that is the bug, and you need to call the function later in your flow, once the user object is populated.

```javascript
function identifyUser(userId) {
  if (!isDespia || !userId) return
  console.log('linking push for user:', userId) // confirm this is a real id
  despia(`setonesignalplayerid://?user_id=${userId}`)
}
```

### Symptom: OneSignal says it sent, but nothing arrives on the phone

The device is registered and targeted, but delivery fails at the OS layer.

**The user never granted permission.** Check with `checkNativePushPermissions://` and route them to settings if denied. A user who dismissed or denied the prompt receives nothing.

**On iOS, the Push Notifications capability is still missing.** Same cause as the no-devices case. OneSignal will show the send as successful because it handed the payload to APNs, but APNs drops it. Enable the capability, rebuild, reinstall.

**You are testing on a simulator.** iOS simulators do not receive real push. Test on a physical device.

**Low power mode or a force-quit app on Android.** Some Android OEMs aggressively kill background delivery for battery. This is device behaviour, not a Despia or OneSignal problem, and it affects native apps generally.

## What AI advice gets wrong about Despia push

Most broken setups trace back to a small set of confident, wrong answers that AI assistants give because they are describing raw OneSignal web integration, not Despia. Here is the correction for each.

**"Call** `OneSignal.login(externalId)` **after the user signs in."** Despia does not expose the raw OneSignal SDK methods. There is no `OneSignal.login`, `setExternalUserId`, or `getDeviceState` to call. The equivalent, and the only correct command, is `setonesignalplayerid://?user_id=`. Under the hood it sets the `external_id`, which is exactly what those methods would have done.

**"Check** `window.despia` **to detect the native app."** That is not the detection method and it will be undefined. The canonical check is the user agent:

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')
```

**"Register a service worker and add** `firebase-messaging-sw.js`**."** That is web push. It does nothing inside a native app. Despia handles native registration for you, and the service worker file is dead weight.

**"Store the OneSignal Player ID in a** `device_push_tokens` **table."** Player IDs are the deprecated model. You do not need to store device tokens at all. You target users by your own ID through `external_id`. If a boilerplate handed you a push tokens table, it is scaffolding from a different stack, not something Despia needs.

**"Schedule reminders locally and cancel them when the user edits the appointment."** There is a simple on-device option for a one-off timed nudge, but it cannot cancel, update, or list anything after the fact. For any reminder a user can edit or delete, you need OneSignal, which is covered next. Do not build editable reminders on a fire-once scheduler.

## Safe reminders: the architecture that does not misfire

Reminder apps hit a specific trap. A user sets a reminder, then edits or deletes the underlying appointment, homework item, or medication, and a stale reminder fires anyway because nothing could reach in and cancel it. If your app deals with anything where a wrong or stale notification matters, and medication is the obvious case, this is not a polish problem, it is a correctness problem.

The reliable pattern is backend-driven scheduling through OneSignal, which gives you full control over the schedule: create, cancel, replace, list.

```javascript
// schedule: store the returned id against the reminder row
const res = await fetch('https://onesignal.com/api/v1/notifications', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Basic ${process.env.ONESIGNAL_REST_API_KEY}`,
  },
  body: JSON.stringify({
    app_id: process.env.ONESIGNAL_APP_ID,
    include_external_user_ids: [userId],
    headings: { en: 'Medication reminder' },
    contents: { en: 'Time for your evening dose' },
    send_after: '2026-07-10T18:00:00Z',
  }),
})

const { id } = await res.json()
```

```javascript
// on edit or delete: cancel the exact notification tied to this reminder
await fetch(`https://onesignal.com/api/v1/notifications/${notificationId}?app_id=${appId}`, {
  method: 'DELETE',
  headers: { 'Authorization': `Basic ${process.env.ONESIGNAL_REST_API_KEY}` },
})
```

Because the OneSignal notification id lives on the reminder row, editing the item cancels and reschedules the exact notification, and deleting the item cancels it. No stale reminder can survive a change. This is the setup to ship for any real reminder feature.

## Routing to the right screen on tap

Once notifications arrive, most builders want a tap to open a specific screen. Send a `path` in the notification's `data` object and Despia updates the URL for you on tap, no reload.

```javascript
body: JSON.stringify({
  app_id: process.env.ONESIGNAL_APP_ID,
  include_external_user_ids: [userId],
  headings: { en: 'Your order shipped' },
  contents: { en: 'Tap to track it' },
  data: { path: '/account/orders/4567?tab=tracking' },
})
```

Most single-page routers pick up the change automatically. If yours does not, assign a global handler outside the `isDespia` gate and navigate yourself:

```javascript
window.onNotificationEvent = function (payload) {
  if (payload.path) router.navigate(payload.path)
}
```

## FAQ

**I added the OneSignal App ID in Despia. Why is nothing happening?** Adding the App ID does nothing until you rebuild, because OneSignal is compiled into the binary. Bump the version under App, Settings, Versioning, rebuild, and install the new build on your device. Testing the old build is the most common reason people think the App ID did not take.

**No users show up in OneSignal at all. What is wrong?** The SDK never registered. On iOS this is usually the missing Push Notifications capability on your bundle ID, or an old build. On Android it is usually Firebase not being wired up. Confirm you rebuilt after adding the App ID, that the capability is enabled on Apple's side, and that the OneSignal platform is set to Native, not Web Push.

**Users appear in OneSignal but have no External ID. Why?** You are not calling `setonesignalplayerid://?user_id=` correctly. It has to run on every authenticated load, after login, with a real user ID. Calling it once at signup, or before the user is logged in, leaves the External ID blank. Log the ID right before the call to confirm it is real.

**Do I need to write code to register the device or handle tokens?** No. Despia registers the device with OneSignal automatically at launch. You do not manage APNs or FCM tokens, and you do not store device tokens. You only link the user with their ID and send from your backend.

**AI told me to call** `OneSignal.login()`**. It does not work. What do I use?** Despia does not expose the raw OneSignal SDK. Use `setonesignalplayerid://?user_id=` instead. It sets the same `external_id` that `OneSignal.login()` would have.

**Why is** `window.despia` **undefined?** That is not the detection method. Detect the native app with `navigator.userAgent.toLowerCase().includes('despia')`.

**Do I still need Player IDs or a device tokens table?** No. OneSignal moved to `external_id`, which is your own user ID. Target users by that ID with `include_external_user_ids`. You do not need to store OneSignal Player IDs or push tokens anywhere.

**Should I pick Web Push in OneSignal since my app is web code?** No. Despia apps are native, so choose Native iOS and Native Android. The web push path, including service workers, does not apply and will not deliver.

**Notifications work in my test build but not after the app is on the store. Why?** You are almost certainly still testing an older version somewhere, or the store build was made before the App ID or capability was in place. Every push change requires a fresh build with a raised version, installed and tested on device.

**Where do I enable the iOS Push Notifications capability?** In Apple Developer Console, Identifiers, open your core bundle ID, tick Push Notifications under Capabilities, save, then rebuild in Despia. This is not toggled by adding the App ID in Despia.

**How do I set up Firebase for Android push?** Create a Firebase project, add an Android app with the exact same package name as your Despia app, then paste the Server Key and Sender ID into OneSignal under Settings, Push and In-App, Google Android. Rebuild after.

**A user denied notifications. Can I ask again?** Not with a fresh system prompt on iOS. Check the state with `checkNativePushPermissions://` and, if denied, route them to their app settings with `settingsapp://` where they can re-enable manually.

**Can I schedule a reminder without a backend?** For a simple one-off nudge, on-device scheduling exists. For any reminder a user can edit or delete, no, because it cannot be cancelled or updated after it is set. Use backend-driven OneSignal scheduling so you can cancel and replace when the underlying item changes.

**How do I send a critical alert that bypasses Do Not Disturb?** On iOS this needs Apple's Critical Alerts entitlement, which you request from Apple and enable in Despia before rebuilding, then set `ios_critical_alert` to 1 in your payload. On Android, set `priority` to 10 and use a high-importance channel ID. Full details are in the OneSignal reference below.

**Can a notification tap open a specific screen?** Yes. Send a `path` in the notification's `data` object and Despia updates the URL on tap. Most routers navigate automatically, or you can handle it in `window.onNotificationEvent`.

## Add this to your app

Every call in this post runs from your existing web codebase, one JavaScript function away, with the native SDK already bundled and registered for you. No Xcode, no token handling, no native project to maintain. The full OneSignal reference, including critical alerts, segmentation, and the complete payload options, is in the docs.

[Read the full push notifications reference at setup.despia.com](https://setup.despia.com/native-features/onesignal/introduction)