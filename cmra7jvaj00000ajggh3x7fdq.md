---
title: "Push Notification Deep Linking in a WebView App"
seoTitle: "Push Notification Deep Linking in a WebView App"
seoDescription: "Push notification deep linking that opens the right screen in a WebView app, using data.path and one JavaScript event. No native rebuild needed."
datePublished: 2026-07-07T05:27:59.304Z
cuid: cmra7jvaj00000ajggh3x7fdq
slug: push-notification-deep-linking-in-a-webview-app
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/f9a55987-8c1b-4a87-9253-0535d1c92d91.png
tags: android-app-development, ios, android, push-notifications, ios-app-development, onesignal, deeplinking

---

A user taps a push that says their order shipped. The app opens on the home screen. The notification knew which order, the payload carried the route, and none of it reached the screen the user wanted. This is the most common push bug in a WebView app, and it is almost always one field sitting in the wrong place.

## Where the routing actually lives

Most push providers give you a top-level `url` field. In OneSignal it exists for web push, and it is the first thing people reach for. Inside a native WebView app it is the wrong field, because the runtime does not read routing from the provider's launch URL. It reads it from the notification's `data` object.

Put your route in `data` and the app can act on it. Leave it at the top level and the app opens at your start URL every time, which is exactly the home-screen symptom people report. Nothing is broken. The payload is just being read from a place the runtime never looks.

```js
// Backend: OneSignal Create Notification REST API
// The route goes inside data, not at the top level
await fetch('https://onesignal.com/api/v1/notifications', {
    method: 'POST',
    headers: {
        'Content-Type':  'application/json',
        'Authorization': `Basic ${process.env.ONESIGNAL_REST_API_KEY}`,
    },
    body: JSON.stringify({
        app_id:                    process.env.ONESIGNAL_APP_ID,
        include_external_user_ids: [userId],
        headings:                  { en: 'Your order shipped' },
        contents:                  { en: 'Tap to track it' },
        data: {
            path: '/account/orders/4567?tab=tracking',
        },
    }),
})
```

## path versus url

The `data` object gives you two routing fields, and the difference matters.

`path` is a route like `/account/orders/4567`. Despia applies it through the History API and fires `popstate`. No reload. Most SPA routers already listen for `popstate`, so the navigation happens on its own with nothing extra on the web side. This is the field you want.

`url` is the legacy path. It takes a full URL or relative path and forces a full WebView reload before the app is usable again. Reach for it only when you actually need to reload, for example when the target lives outside your SPA. For in-app routes it is slower and worse.

| Field | How it navigates | Reload | Use it for |
| --- | --- | --- | --- |
| `path` | History API, fires popstate | No | In-app SPA routes |
| `url` | Forces WebView reload | Yes | Legacy or cross-origin |

## Reading the payload yourself

Sending `data.path` is enough for a router that reacts to `popstate`. If yours does not, or you want to carry extra state along, read the open event directly. Despia calls `window.onNotificationEvent` on every tap, after it has applied the URL change.

```js
import despia from 'despia-native'

// Assigned globally so the native runtime can always find it,
// even before the app has mounted anything.
window.onNotificationEvent = function (payload) {
    // payload.path     -> internal route from data.path
    // payload.url      -> legacy full-reload target from data.url
    // payload.metadata -> your data.metadata object, if any
    if (payload.path) {
        router.navigate(payload.path)
    }

    if (payload.metadata) {
        restoreState(payload.metadata) // reopen a modal, preload the record, etc.
    }
}

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

if (isDespia) {
    despia(`setonesignalplayerid://?user_id=${userId}`)
}
```

Two things matter about where this lives. Assign it at the top level, not inside the `isDespia` gate, because the runtime is the one calling it. And assign it somewhere global, not inside a component that can unmount, or the handler disappears the moment that component leaves the screen and taps stop routing. Your `despia()` calls stay inside the gate.

On React Router, register it from a component that stays mounted for the app's lifetime, and clean it up on unmount:

```jsx
useEffect(() => {
    window.onNotificationEvent = (payload) => {
        if (payload.path) navigate(payload.path)
        if (payload.metadata) restoreState(payload.metadata)
    }

    return () => {
        delete window.onNotificationEvent
    }
}, [navigate])
```

## Foreground, background, cold start

The same event fires in every app state. Only the timing changes, and that timing is what trips people up when they test one state and assume the rest are broken.

| App state when tapped | When the route applies |
| --- | --- |
| Foreground (already loaded) | Immediately. The URL changes with no reload. |
| Background (resumed) | When the WebView is presented, once the page is ready. |
| Killed / cold start | The native side buffers the payload and applies it once the page finishes loading. |

Cold start is the case people fear, because the JavaScript is not running when the tap happens. The runtime holds the payload and replays it after your app boots, so a killed app routes correctly without any special handling. The event delivers once per tap and then clears.

## Carrying state, not just a route

A route tells the app where to go. Sometimes you also want to hand it context, like which record the notification was about or a scroll position to restore. Send `data.metadata` as a JSON object and it arrives on the same payload.

```js
// Backend: OneSignal Create Notification REST API
await fetch('https://onesignal.com/api/v1/notifications', {
    method: 'POST',
    headers: {
        'Content-Type':  'application/json',
        'Authorization': `Basic ${process.env.ONESIGNAL_REST_API_KEY}`,
    },
    body: JSON.stringify({
        app_id:                    process.env.ONESIGNAL_APP_ID,
        include_external_user_ids: [userId],
        headings:                  { en: 'New reply' },
        contents:                  { en: 'Tap to read' },
        data: {
            path:     '/threads/812',
            metadata: { highlightMessageId: 4471 },
        },
    }),
})
```

One note worth knowing before it costs you a debugging hour: `metadata` sent through the REST API `data` field arrives as a real object. The same value set through the OneSignal dashboard's additional-data fields arrives as a string and needs `JSON.parse`. Handle both if your sends come from two places.

## The two failure modes

Almost every "my push opens the home screen" report is one of these. First, the route is at the top-level `url` instead of inside `data`, so the runtime never sees it. Second, the route is in `data.path` correctly but the SPA router does not react to a synthetic `popstate`, so nothing navigates. The first is a one-line move into `data`. The second is fixed by wiring `window.onNotificationEvent` and calling your router yourself.

Neither needs a native rebuild. This is all payload shape and one callback, running in the web codebase you already have.

## Add this to your app

Every native capability in this post is one JavaScript call away, inside your existing web codebase. No Xcode, no native project to maintain.

[Read the full reference at setup.despia.com](https://setup.despia.com/native-features/onesignal/reference)