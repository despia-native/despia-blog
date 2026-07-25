---
title: "Web App Google Login (Native Sign-In)"
seoTitle: "Web App Google Login (Native Sign-In)"
seoDescription: "Web app Google login done natively: open a secure browser with the Despia OAuth bridge, handle the callback, and set the session on iOS and Android."
datePublished: 2026-07-11T15:12:03.908Z
cuid: cmrgi6ek200000aizgq8oak32
slug: web-app-google-login-native-sign-in
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/17f2f5f5-ae42-45d6-9d77-af610e3cc4e9.png
tags: html, webview, webapp, google-login

---

* * *

Whatever your web app is built with, plain HTML and JavaScript included, a Google sign-in button that works in the browser fails once the app ships to the stores. Google blocks its OAuth screen in an embedded WebView, and the App Store expects a real browser session. Despia gives you the native `oauth://` bridge, which opens the secure system browser, ASWebAuthenticationSession on iOS and Chrome Custom Tabs on Android, and returns the tokens to your app. One flow covers both platforms, with a normal redirect on web.

## Why the web Google button fails in a mobile app

Google refuses its consent screen inside an embedded WebView, the `disallowed_useragent` error, because an embedded view could read the user's credentials. Apple's guidelines say the same: OAuth has to run in a browser the app cannot inspect. So the plain web button either errors or is rejected at review. The fix is to step out of the WebView for the sign-in. Despia's `oauth://` bridge opens the system browser, the user authenticates, and Despia closes it and returns the tokens. Unlike Apple Sign In, Google uses the same bridge on iOS and Android, so there is no platform branch in your code.

## How the flow works

1.  The user taps Sign in with Google.
    
2.  In Despia, your backend builds a Google OAuth URL that redirects to a small file, `native-callback.html`, and returns it.
    
3.  `despia('oauth://?url=...')` opens that URL in the secure browser.
    
4.  The user signs in. Tokens arrive at `native-callback.html`.
    
5.  That file fires `myapp://oauth/auth?access_token=...`. The `oauth/` prefix tells Despia to close the browser and navigate the WebView to `/auth`.
    
6.  Your `/auth` page reads the token and creates the session.
    

On web the button runs your provider's normal redirect and lands on the same `/auth`.

## Wire the sign-in button

Before any code, register your deeplink scheme in the Despia dashboard under Publish > Deeplink, that is the value `myapp` stands for below. Then install the SDK with `npm install despia-native`, or load it from the CDN, and branch:

```javascript
import despia from 'despia-native'

const isDespia = typeof navigator !== 'undefined' &&
  navigator.userAgent.toLowerCase().includes('despia')

document.getElementById('google-signin').addEventListener('click', async () => {
  if (isDespia) {
    // get the OAuth URL from your backend and open it in the secure browser
    const res = await fetch('/api/auth/google-url', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ deeplink_scheme: 'myapp' }), // Despia > Publish > Deeplink
    })
    const { url } = await res.json()
    despia('oauth://?url=' + encodeURIComponent(url))
  } else {
    // web: normal redirect, supabase-js installs the session itself on return
    await supabase.auth.signInWithOAuth({ provider: 'google', options: { redirectTo: location.origin + '/auth' } })
  }
})
```

Replace `myapp` with your scheme from Despia > Publish > Deeplink, and swap the web call for your provider if you are not on Supabase.

## Generate the OAuth URL on the backend

Your endpoint builds the URL with `native-callback.html` as the redirect. On Supabase the implicit redirect returns the tokens in the URL hash:

```javascript
// server-side handler, builds the Google OAuth URL
const { deeplink_scheme } = await req.json()
const redirectUrl = `https://yourapp.com/native-callback.html?deeplink_scheme=${encodeURIComponent(deeplink_scheme)}`

const url = `${process.env.SUPABASE_URL}/auth/v1/authorize?` + new URLSearchParams({
  provider: 'google',
  redirect_to: redirectUrl,
  scopes: 'openid email profile', // optional, Supabase requests these by default
  flow_type: 'implicit', // implicit is the default without a code_challenge, kept to document intent
})

return Response.json({ url })
```

Why implicit in this example? Client-session setups, Supabase-style, need the OAuth result to land in the browser context that will call `setSession()`. In that setup, the secure browser receives the hash tokens, `native-callback.html` forwards them through Despia's `oauth://` bridge, and `/auth` installs the session in the app. The deeper reason the handoff exists: the OAuth screen runs in a separate secure-browser context, ASWebAuthenticationSession on iOS and Chrome Custom Tabs on Android, whose cookies and storage cannot cross into the WebView by design. A PKCE verifier stored where the flow starts can never meet the code where the session must be installed, so passing the tokens through the deeplink is the bridge between the two worlds. And because the hash carries Supabase's own access and refresh token pair, not a raw Google grant, `setSession()` yields a full session that supabase-js keeps refreshing, users are not signed out when the access token expires.

If you own the backend, or your provider supports a clean PKCE exchange in the app, prefer authorization-code with PKCE instead of this client-token handoff. Supabase supports that with `exchangeCodeForSession()`, and custom backends should generally use code plus PKCE rather than passing long-lived tokens through the URL. The Despia side is identical either way: the same `oauth://` open, the same callback file, the same `oauth/` deeplink home.

## Add public/native-callback.html

This small page runs inside the secure browser. It reads the tokens from the URL hash and fires the deeplink that closes the browser and passes them to your app. It is a plain HTML file either way, which is the point: nothing between the redirect and your read can touch the hash.

```html
<!-- public/native-callback.html -->
<!DOCTYPE html>
<html lang="en">
<head><meta charset="UTF-8" /><title>Completing sign in...</title></head>
<body>
<p>Completing sign in...</p>
<script>
  (function () {
    var params = new URLSearchParams(location.search)
    var scheme = params.get('deeplink_scheme')
    if (!scheme) return

    var hash = new URLSearchParams(location.hash.slice(1))
    var accessToken = hash.get('access_token')
    var refreshToken = hash.get('refresh_token') || ''
    var error = hash.get('error') || params.get('error')

    // oauth/ tells Despia to close the browser and navigate the WebView to /auth
    location.href = accessToken
      ? scheme + '://oauth/auth?access_token=' + encodeURIComponent(accessToken) + '&refresh_token=' + encodeURIComponent(refreshToken)
      : scheme + '://oauth/auth?error=' + encodeURIComponent(error || 'no_access_token')
  })()
</script>
</body>
</html>
```

The `oauth/` prefix is not optional. `myapp://oauth/auth` closes the session and opens `/auth`. `myapp://auth` without it does nothing, and the user is stuck in the browser.

## Read the token on /auth

Both flows end here: the native deeplink puts the token in the query string, the web redirect puts it in the hash. Read both on load, and again on `popstate` so a deeplink into an already-open page still runs:

```html
<p id="status">Signing you in...</p>

<script>
  function handleAuth() {
    var p = new URLSearchParams(location.search)
    var hash = new URLSearchParams(location.hash.slice(1))
    var accessToken = p.get('access_token') || hash.get('access_token')
    var refreshToken = p.get('refresh_token') || hash.get('refresh_token') || ''
    var error = p.get('error') || hash.get('error')

    if (error) { document.getElementById('status').textContent = 'Sign in failed: ' + error; return }
    if (!accessToken) return

    // setSession is your provider's call, e.g. supabase.auth.setSession()
    setSession({ access_token: accessToken, refresh_token: refreshToken })
      .then(function () { location.href = '/' })
  }

  handleAuth()
  window.addEventListener('popstate', handleAuth)   // deeplink changed the URL in place
  window.addEventListener('hashchange', handleAuth) // hash-only change, popstate does not fire
</script>
```

Running only on load misses a deeplink that arrives while `/auth` is already open. The `popstate` and `hashchange` listeners are what make it fire on the in-place URL change Despia performs, since a hash-only change does not raise `popstate`.

## Set up Google in the Cloud Console

1.  In console.cloud.google.com, open APIs and Services, then Credentials, and create an OAuth 2.0 Client ID of type Web application.
    
2.  Add your app domain under Authorized JavaScript origins.
    
3.  Under Authorized redirect URIs, add your provider's callback. For Supabase that is `https://YOUR-PROJECT.supabase.co/auth/v1/callback`; for a custom backend, your backend callback URL.
    
4.  Copy the Client ID and Client Secret into your provider's Google settings.
    
5.  Set your deeplink scheme from Despia > Publish > Deeplink and replace `myapp` in the code.
    

## The four things that cause almost every failure

The `oauth/` prefix must be in the deeplink, or Despia never closes the browser. `native-callback.html` must be a real static file served at that path. `/auth` must read tokens from both the query string and the hash, since native and web deliver them differently. And `/auth` must re-run its token logic on a URL change, which is why the handler runs on load, on `popstate`, and on `hashchange`.

## Ship it with native sign-in built in

Take your web app to iOS and Android with Google sign-in built for review, from the single codebase you already ship. Code signing and submission run from the browser, no Mac and no CLI.

[See the setup docs at setup.despia.com](https://setup.despia.com/native-features/oauth/google)