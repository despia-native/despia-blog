---
title: "Fix Sign in with Apple Rejection in a WebView App"
seoTitle: "Fix Sign in with Apple Rejection in a WebView App"
seoDescription: "Got a Sign in with Apple rejection on your WebView or wrapper app? Here is what Guideline 4.8 requires and how to ship native login that passes."
datePublished: 2026-06-30T23:08:21.031Z
cuid: cmr19cjh200000ako5owld1vt
slug: fix-sign-in-with-apple-rejection-in-a-webview-app
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/7f638cdc-4aca-42b1-9df5-d9f36f6bdbda.jpg
tags: webview, appst

---

Your login works on the web. It works on your phone. You submit, and Apple bounces it: missing Sign in with Apple, or a blank screen during authentication, or a flow that "does not appear to function." All three are login rejections, and they hit wrapper and WebView apps the hardest, because the failure is baked into how a WebView handles auth in the first place. Here is what Apple is actually checking and how to clear it in a Despia app without rebuilding your auth from scratch.

## The two reasons Apple rejects your login

The first is Guideline 4.8. If your app offers any third-party social login, Google, Facebook, X, anything, you must also offer Sign in with Apple, and it has to be equally prominent. Not buried under a "more options" link, not smaller than the Google button. The rule is narrow and worth memorising:

*   Third-party login present, Sign in with Apple required.
    
*   No third-party login, not required.
    
*   Email and password only, not required.
    

So if your login screen has a Google button and nothing from Apple, that is an automatic rejection, every time, no exceptions for "but it is a web app."

The second rejection is more confusing because the app works for you. A web OAuth flow redirects the browser to the provider's domain and back. Inside a WebView, that redirect has nowhere to land. The reviewer taps Sign in with Google, the screen goes white while the redirect hangs, and to Apple a white screen during auth is a broken app. You see "blank screen," "unresponsive during sign in," or a minimum functionality rejection. It is not your account or your build, it is the redirect model running inside a native shell.

Despia fixes both. The catch is they need different mechanisms.

## What Despia is doing here

Despia ships your existing web app as a real native binary and keeps one codebase. For auth specifically, it gives the web layer two native paths the browser does not have: Apple's native sign-in sheet, and real native OAuth through the system browser instead of an in-WebView redirect. On iOS that is `ASWebAuthenticationSession`, on Android it is Chrome Custom Tabs. Both return control to your app through a deeplink, which is the piece a plain WebView redirect cannot do.

That is the whole reason the reviewer sees a working flow instead of a white screen.

## Detect the runtime first

Every branch below keys off whether the code is running inside the native app. The canonical check is the user agent, never a made-up global:

```javascript
const ua = navigator.userAgent.toLowerCase()
const isDespia = ua.includes('despia')
const isDespiaIOS = isDespia && (ua.includes('iphone') || ua.includes('ipad'))
const isDespiaAndroid = isDespia && ua.includes('android')
```

## Sign in with Apple is two flows, not one

This is the part people get wrong, including a first pass at this post. Apple's JavaScript SDK can summon the native sign-in sheet on iOS and in the browser on web, but it cannot do it on Android. So Sign in with Apple is two different implementations gated by platform, and skipping the Android half means an Android user taps the button and nothing happens.

On iOS and web, load Apple JS in your `index.html` before your app script, then call it with `usePopup: true`. That one flag is the whole anti-rejection move. With the popup, the `id_token` comes straight back in the JS callback, no redirect. Without it, the SDK falls back to a redirect to apple.com, the WebView shows a blank white page while that hangs, and that white page is the App Store rejection.

```javascript
// iOS and web. usePopup is what prevents the blank-screen rejection.
async function signInWithApple() {
  window.AppleID.auth.init({
    clientId: 'com.yourcompany.yourapp.webauth', // your Services ID
    scope: 'name email',
    redirectURI: 'https://yourapp.com/',         // must match the Services ID origin exactly
    usePopup: true,
  })
  const res = await window.AppleID.auth.signIn()
  await exchangeAppleToken(res.authorization.id_token)
}
```

On Android, Apple JS has no native sheet to open, so the flow goes through the system browser instead. You ask your backend for an Apple OAuth URL and hand it to Despia, which opens Chrome Custom Tabs, lets Apple redirect to a small bridge page, and deeplinks back into the app:

```javascript
import despia from 'despia-native'

async function signInWithAppleAndroid() {
  const { url } = await fetch('/api/auth/apple-url', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ deeplink_scheme: 'myapp' }),
  }).then(r => r.json())

  despia(`oauth://?url=${encodeURIComponent(url)}`)
}
```

The bridge page (a plain `native-callback.html`, not a React route, because the router can strip the token out of the hash) reads the token and redirects to a deeplink with one non-obvious requirement: the `oauth/` prefix. `myapp://oauth/auth` closes the tab and lands the WebView on `/auth`. Drop the prefix and the tab just sits there. The full bridge and the `/auth` token handling are in the docs linked below.

Branch the button by platform and you have covered all three contexts:

```javascript
async function handleAppleSignIn() {
  if (isDespiaAndroid) return signInWithAppleAndroid()
  return signInWithApple() // iOS native sheet and web popup both run here
}
```

This satisfies 4.8 and clears the blank-screen rejection at the same time, as long as `usePopup: true` is set on iOS and the Android bridge actually exists.

## Google and the rest: native OAuth, not in-WebView

For other providers, open the OAuth flow in the system browser and return through a deeplink. The native call only makes sense inside the runtime, so it sits behind the `isDespia` gate, with the standard web flow as the fallback:

```javascript
async function signInWithGoogle() {
  if (isDespia) {
    // generate the OAuth URL server side, pointed at a /native-callback route
    const { data } = await supabase.functions.invoke('auth-start', {
      body: { provider: 'google', deeplink_scheme: 'myapp' }
    })
    // opens ASWebAuthenticationSession (iOS) / Chrome Custom Tabs (Android),
    // then deeplinks back into the app to close the session
    despia(`oauth://?url=${encodeURIComponent(data.url)}`)
  } else {
    // web: standard provider OAuth
    await supabase.auth.signInWithOAuth({ provider: 'google' })
  }
}
```

The reason for the edge function is the redirect target. A normal web login redirects to `/auth`, which a WebView cannot resume. The native flow redirects to a `/native-callback` route instead, and that page fires the deeplink that hands control back to the app. Without that indirection, the session opens and never closes, which is the hang the reviewer sees.

## Kill the white flash on the callback page

Even with native OAuth, the callback page renders for a moment while its JavaScript runs. If that moment is a blank white page, a reviewer on a slow connection can still catch it. Give the callback page a static HTML loader with inline CSS, so a spinner that matches your app theme paints before any script executes. Show a spinner before you open the OAuth session too, not after. The fix is boring and it is what clears "white flash on login."

## The login rejections that ride along

Two more things get flagged in the same review pass as login, so handle them before you resubmit.

An access gate is not allowed. If tapping login lands the user on a "request access" or "your account is not authorized" page, Apple rejects it. The store requires instant access for any user, with no approval step and no hidden features. Remove the gate, or if you genuinely need gated access, provide a fully provisioned demo account in the App Review notes and make sure that account has full access.

And if your backend uses an approval flow, give the reviewer working credentials in App Store Connect regardless. A reviewer who cannot get past your login will reject for it, and they will not email you to ask.

## Get it on the stores

Take the app you already built, add Sign in with Apple and native OAuth through your one web codebase, and ship a login that passes review on iOS and Android. No Xcode, no second project, code signing and submission run from the browser.

[See the full Apple sign-in setup in the docs](https://setup.despia.com/native-features/oauth/apple)