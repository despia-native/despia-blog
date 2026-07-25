---
title: "PayPal in a WebView App: iOS and Android"
seoTitle: "PayPal in a WebView App: iOS and Android"
seoDescription: "PayPal blocks WebViews. Here is how to open its approval page in a compliant browser session from a Despia app."
datePublished: 2026-07-04T23:24:11.544Z
cuid: cmr6zobk300000ahu11pge87f
slug: paypal-in-a-webview-app-ios-and-android
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/69278797-9a8a-4e03-b65f-c61ceb08b27e.jpg
tags: paypal, webview

---

PayPal refuses to run its login and approval page inside a WebView. So does Google's OAuth consent screen, and so does Stripe in its stricter modes. The security reason is the same in every case: an embedded WebView can read the page, so providers won't let their auth flow load in one. The naive fix, opening the URL in a new tab, leaves you with a browser surface JavaScript can't close, a Safari sheet stuck over your app on iOS. Here is the flow that opens the page in a compliant browser context on both iOS and Android and hands control back to your app cleanly, from one codebase.

## Why the page won't load in the first place

Providers like PayPal detect embedded WebViews and block their OAuth-style flows. Sometimes the page refuses outright, sometimes the login just fails silently after the user types their password. Either way the fix is not on your side of the page, it is where the page runs. The approval screen has to open in a real browser surface. On iOS that means an `ASWebAuthenticationSession`, on Android the Custom Tabs based auth session. The requirement is identical on both platforms, and so is the code you write.

## The trap: opening it in a new tab

The obvious move is to open the approval URL in a new tab and let the system take it from there:

```javascript
// looks right, dead ends on the way back
window.open(approvalUrl, '_blank')
```

That does open the page in a browser surface, and the user can approve the payment. The problem is the return trip. When PayPal redirects back to your app through a custom scheme, the app receives the callback and reloads underneath, but the browser surface stays on top, because nothing in the JavaScript layer can dismiss it. On iOS you are left staring at a Safari sheet over your own app, with an "Open in App?" prompt on the way through. There is no JS-callable way out of that sheet.

## The flow that closes itself

The runtime has a purpose-built call for this, and it is the same call on both platforms. `oauth://?url=` opens the target URL in a compliant browser session, an `ASWebAuthenticationSession` on iOS and the Custom Tabs based auth session on Android. Both satisfy the no-WebView requirement, and both auto-dismiss the moment your callback scheme fires. That last part is the whole point: the browser surface closes itself on the way back, so you never touch it.

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

function startPayPalCheckout(approvalUrl) {
  if (isDespia) {
    despia('oauth://?url=' + encodeURIComponent(approvalUrl))
  } else {
    // plain web, just navigate
    window.location.href = approvalUrl
  }
}
```

Encode the URL. That is the entire opening half, and it does not branch per platform.

## The callback, and where it lands

Set PayPal's return URL to your app's deeplink scheme with an `oauth/` path. The `oauth` host is a reserved marker. The runtime strips it and grafts the remaining path and query onto your app's own configured URL, then loads that inside the WebView. Same behaviour on iOS and Android. The mapping is direct:

| Provider redirects to | App loads |
| --- | --- |
| `myapp://oauth/complete?data=abc` | `https://yourapp.com/complete?data=abc` |

Two things worth knowing before you wire it up:

1.  The base is always your app's configured URL. You cannot point the callback at an arbitrary domain, it resolves against the origin your app already loads.
    
2.  The scheme is preset from your app name and is not editable. You will find it under Despia > Publish > Deeplink, and it is shared across both platforms. It is the same scheme your existing deep links already use, so if those work, this lands on the route you expect.
    

The return route itself can be thin. It reads whatever you passed in `data`, confirms the order server-side, and moves the user on:

```javascript
// /complete
const params = new URLSearchParams(window.location.search)
const token = params.get('data')
// verify with your backend, then continue the flow
```

## The parts that bite

*   The callback host has to be exactly `oauth`. If it is anything else the strip does not happen and the path will not map onto your domain.
    
*   Do not mix the two approaches. If you open with `_blank` and expect the callback to close the surface, it will not. Open with `oauth://?url=` so the session owns its own dismissal.
    
*   On plain web the same function just redirects, so one checkout handler covers web, iOS, and Android. The `isDespia` gate is the only branch you need.
    

## Where this fits

This is not a PayPal-specific trick, and it is not a per-platform one. Any provider that blocks embedded WebViews, PayPal, Google, Stripe Checkout, Plaid, goes through the same two steps on both stores: open the provider URL with `oauth://?url=`, and set the provider's return URL to `{yourscheme}://oauth/{path}`. The session opens in a compliant browser surface and closes itself when the scheme fires. From your code it is one call out and one route back, and the same code ships to iOS and Android.

## Add this to your app

Every native capability in this post is one JavaScript call away, inside your existing web codebase. No Xcode, no native project to maintain.

[Read the full reference at setup.despia.com](https://setup.despia.com/native-features/oauth/introduction)