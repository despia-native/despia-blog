---
title: "Base44 Apple Login (Native Sign-In)"
seoTitle: "Base44 Apple Login (Native Sign-In)"
seoDescription: "Base44 Apple login done natively: the Apple JS SDK shows the native sign-in sheet on iOS, Android rides the OAuth bridge, your own JWT is the session."
datePublished: 2026-07-21T04:45:54.919Z
cuid: cmru67osy00010ahs6qsw8sny
slug: base44-apple-login-native-sign-in
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/20e7e7f8-02eb-421f-b8dc-a73407fab8af.png
tags: oauth, webview, webapp, base44

---

If your Base44 app offers Google login on iOS, Guideline 4.8 obliges you to offer an equivalent privacy-focused login next to it, one that limits data collection and supports private email, and Sign in with Apple is usually the simplest compliant choice, unless one of Guideline 4.8's exceptions applies. Apple's exceptions include proprietary account systems, enterprise and education accounts, and government identity systems. The good news is that Apple is the easy one. Unlike Google, Apple's JS SDK with `usePopup: true` can present the native Apple sign-in sheet, Face ID, Touch ID, or the device passcode from the WebView in Despia, so there is no browser session to manage on iPhone at all. Android has no equivalent, so it rides the same `oauth://` bridge as Google. And because Base44's built-in auth cannot mint a session for either path, Apple plugs into the same custom JWT system you built for Google login: one more backend function, the same `Account` entity.

> **This builds on the custom auth setup.** Everything here assumes the custom JWT system from the Base44 Google login guide: the `Account` entity with deny-all RLS, the `signJwt` helper, and the `JWT_SECRET`. Apple is an addition to that system, not a standalone flow. The full working reference is the template at github.com/despia-native/base44-native-oauth.

## Three platforms, two flows

On iOS, the Apple JS SDK with `usePopup: true` opens the native Apple sign-in sheet, Face ID, Touch ID, or the device passcode directly inside the WebView. The `id_token` lands in your JS callback, no redirect, no bridge, no callback file. On the web the same SDK call opens a normal browser popup and behaves identically. Android is the odd one out: the SDK cannot trigger a native sheet there, so the flow opens Chrome Custom Tabs through `despia('oauth://?url=...')`, Apple redirects to a small bridge file, and a deeplink with the `oauth/` prefix closes the tab and hands the token back to the WebView.

Both paths end the same way: your backend verifies the Apple `id_token` and mints your own JWT.

## Apple Developer Console setup, no .p8 needed

The best part of this setup: **login needs no** `.p8` **key**. This is the OpenID Connect `id_token` flow. The `appleSignIn` function verifies Apple's token against Apple's public keys (`appleid.apple.com/auth/keys`), checks `iss` / `aud` / `exp`, and mints your own HS256 JWT. Apple is only used to prove identity, so there is no Sign in with Apple private key and no client secret to create or rotate. A `.p8` enters the picture later for authenticated Apple calls, including code exchange and token revocation. Plan for it if you ship in-app account deletion: Apple recommends exchanging the authorization code at first sign-in and storing the refresh token so deletion can revoke Apple access automatically, and documents a manual fallback when those tokens were never obtained.

1.  **App ID.** At developer.apple.com, Certificates, Identifiers and Profiles, Identifiers, open your app's App ID (for example `com.yourcompany.yourapp`) and enable the Sign In with Apple capability. This enables the native app side.
    
2.  **Services ID.** Create a Services ID, for example `com.yourcompany.yourapp.webauth`. This exact string is your `clientId`, the public client id the Apple JS SDK uses, like a Google OAuth client id. An App ID does not work as the client id, using one gives `invalid_client`.
    
3.  **Configure the Services ID.** Open it, enable Sign In with Apple, set the Primary App ID to the App ID from step 1, add your domain, and register two Return URLs, comma-delimited, `https://` required, no wildcards:
    
    *   `https://yourapp.com/` for the JS SDK popup on iOS and web.
        
    *   `https://yourapp.com/functions/appleCallback` for the Android Chrome Custom Tabs flow. Backend functions are served on your app's own domain at `/functions/<name>`, which is what makes this registrable. Without a custom domain, register the platform URL instead, `https://app.base44.com/api/apps/<APP_ID>/functions/appleCallback`.
        
    
    Apple does exact string matching, so the trailing slash on the root URL matters and both URLs must be registered verbatim.
    
4.  **Secrets.** In the Base44 dashboard set `APPLE_SERVICES_ID` to your Services ID and `APP_BASE_URL` to your app's public URL with no trailing slash. `JWT_SECRET` is already set from your Google auth. That is the whole secret list, no `.p8`, no client secret.
    

## Put the Services ID in two places

This is the single most common mistake, so it gets its own step. Backend secrets never reach the browser, and the Apple JS SDK popup runs in the browser, so the Services ID has to live in two spots, kept identical:

The snippets below show the complete authentication flow, but intentionally leave application-specific database, session, routing, and error-handling functions for you to connect to your existing stack.

```javascript
// src/config/app-config.js  (frontend, read by the SDK popup)
export const appConfig = {
  deeplinkScheme: 'myapp',
  appleServicesId: 'com.yourcompany.yourapp.webauth', // same value as the APPLE_SERVICES_ID secret
}
```

The `APPLE_SERVICES_ID` secret is what `appleSignIn` checks the token's audience against on the backend; `appConfig.appleServicesId` is what the popup sends to Apple on the frontend. A Services ID is a public client id, visible in the popup URL anyway, so putting it in the frontend is safe. If the frontend value is wrong you get `invalid_client` in the Apple popup; if the secret does not match it you get a "Token not issued for this app" 401 after an otherwise successful popup.

One Base44 gotcha worth knowing before you test: the editor preview runs on a different origin than your published app, and that origin is not a registered Return URL, so Apple sign-in always fails there with `invalid_request: Invalid web redirect url`. Test on the published URL.

## Enable it in Despia

On iOS, enable the Sign In with Apple capability for your app in the Despia dashboard so it matches the App ID. On Android there is nothing Apple-specific, the flow rides the existing `oauth://` bridge and your deeplink scheme.

## Load the SDK and detect the platform

Add Apple's script before your app script. It requires HTTPS and will not load on plain localhost. The HTTPS rule has one Despia-specific consequence: the Local Server offline mode serves your app over HTTP, so Apple sign-in cannot initialize there. If your app uses it, open the Despia Editor and switch Offline Support from Native to PWA, then add a service worker that caches your assets. That keeps offline support through the service worker while giving Apple's SDK the secure HTTPS origin it requires.

```html
<script src="https://appleid.cdn-apple.com/appleauth/static/jsapi/appleid/1/en_US/appleid.auth.js"></script>
```

Derive all three constants from one user agent read:

```javascript
const ua = navigator.userAgent.toLowerCase()
const isDespia = ua.includes('despia')
const isDespiaIOS = isDespia && (ua.includes('iphone') || ua.includes('ipad'))
const isDespiaAndroid = isDespia && ua.includes('android')
const isWeb = !isDespia
```

## One sign-in function for all three platforms

iOS and web share the SDK popup. Android asks your backend for the OAuth URL and opens the bridge. `usePopup: true` is not a preference: the redirect mode blanks the screen inside a WebView and gets apps rejected.

```javascript
// src/lib/appleAuth.js
import despia from 'despia-native'
import { base44 } from '@/api/base44Client'
import { appConfig } from '@/config/app-config'

const ua = navigator.userAgent.toLowerCase()
const isDespia = ua.includes('despia')
const isDespiaAndroid = isDespia && ua.includes('android')

// iOS + web return { idToken, fullName }; Android returns null and resumes on /auth
export async function signInWithApple() {
  if (isDespiaAndroid) {
    // no native sheet on Android, run Apple in Chrome Custom Tabs via the bridge
    const res = await base44.functions.invoke('appleAuthUrl', { deeplink_scheme: appConfig.deeplinkScheme })
    despia(`oauth://?url=${encodeURIComponent(res.data.url)}`)
    return null
  }

  if (!window.AppleID?.auth) throw new Error('Apple sign-in is unavailable')

  window.AppleID.auth.init({
    clientId: appConfig.appleServicesId,        // the frontend copy of the Services ID
    scope: 'name email',
    redirectURI: window.location.origin + '/',  // must match a registered Return URL exactly
    usePopup: true,                             // required, redirect mode blanks the screen and gets rejected
  })

  const response = await window.AppleID.auth.signIn()
  // Apple only sends the name on the very first sign-in
  const name = response.user?.name
  const fullName = name ? `${name.firstName || ''} ${name.lastName || ''}`.trim() : ''
  return { idToken: response.authorization.id_token, fullName }
}
```

The caller exchanges the returned `id_token` for your JWT, then stores it the same way the Google flow does:

```javascript
async function completeAppleSignIn() {
  const result = await signInWithApple()
  if (!result) return // Android, /auth finishes the job
  const res = await base44.functions.invoke('appleSignIn', { apple_id_token: result.idToken, full_name: result.fullName })
  setToken(res.data.token) // same setToken + vault as the Google flow
}
```

In your caller, catch and swallow `user_cancelled_authorize`, the cancellation error Apple documents, and you may also handle `popup_closed_by_user` defensively for older integrations. Either means the person changed their mind. One known wrinkle: iOS 17.1 briefly showed an HTML form instead of the native sign-in sheet inside WKWebView, fixed in 17.2, and sign-in still worked throughout.

## Function one: appleAuthUrl

`appleAuthUrl` builds the URL against Apple directly, with `response_mode: 'form_post'`, which Apple requires the moment the `name` or `email` scope is present, so Apple POSTs the result to a backend endpoint rather than a static file, and the deeplink scheme rides in `state` so the registered redirect URI stays clean.

```javascript
// Base44 function: appleAuthUrl
const { deeplink_scheme } = await req.json()
const clientId = Deno.env.get('APPLE_SERVICES_ID')

// Apple hard-requires form_post when the name/email scopes are requested, and a
// POST body needs a server route, so the appleCallback function below is the
// registered Return URL. Functions are served on the app's own domain at
// /functions/<name>, which is what APP_BASE_URL hosts.
const APP_BASE_URL = Deno.env.get('APP_BASE_URL')
const redirectUri = APP_BASE_URL
  ? `${APP_BASE_URL}/functions/appleCallback`
  : `https://app.base44.com/api/apps/${Deno.env.get('BASE44_APP_ID')}/functions/appleCallback`

const url = 'https://appleid.apple.com/auth/authorize?' + new URLSearchParams({
  client_id: clientId,
  redirect_uri: redirectUri, // must exactly match the registered Return URL
  response_type: 'code id_token',
  scope: 'name email',
  response_mode: 'form_post',
  state: deeplink_scheme, // travels via state so the redirect URI stays clean
})

return Response.json({ url })
```

One Apple rule shapes this function. Apple hard-requires `response_mode: 'form_post'` the moment you request the `name` or `email` scope, rejecting anything else with `invalid_request: response_mode must be form_post`. A static HTML file cannot read a POST body, so the Return URL is not `native-callback.html` here, it is a backend function, `appleCallback`, and that is what keeps the name and email scopes on the Android leg instead of dropping them.

## Function two: appleCallback, the form\_post bridge

This endpoint is Apple's Return URL for Android. It reads Apple's form POST, `id_token`, `state` carrying the deeplink scheme, and a `user` JSON field that Apple sends only on the very first authorization, which is where the name comes from on Android. It validates the scheme against a strict pattern, then answers with a small HTML bridge page, a spinner, and a tappable Continue button for schemes that need a user gesture, whose script fires the deeplink:

```javascript
// Base44 function: appleCallback, trimmed to the essence, full file in the template
const data = {}
if (req.method === 'POST') {
  const fd = await req.formData().catch(() => null)
  if (fd) for (const [k, v] of fd.entries()) data[k] = String(v)
}
const q = new URL(req.url).searchParams
const idToken = data.id_token || q.get('id_token') || ''
const error = data.error || q.get('error') || ''
const scheme = data.state || q.get('state') || ''

// Apple sends the name ONLY on the very first authorization, as JSON in `user`
let fullName = ''
try {
  const u = JSON.parse(data.user || '')
  if (u?.name) fullName = `${u.name.firstName || ''} ${u.name.lastName || ''}`.trim()
} catch { /* no user payload, normal after the first sign-in */ }

if (!/^[a-z][a-z0-9+.-]*$/i.test(scheme)) {
  return new Response(invalidPageHtml, { status: 400, headers: { 'Content-Type': 'text/html; charset=utf-8' } })
}

const params = new URLSearchParams()
if (idToken) {
  params.set('id_token', idToken)
  if (fullName) params.set('full_name', fullName)
} else {
  params.set('error', error || 'no_token')
}
const deeplink = `${scheme}://oauth/auth?${params.toString()}`

// bridgePageHtml fires location.href = deeplink, shows a spinner, and reveals
// a tappable Continue button after 800ms for schemes that need a user gesture
return new Response(bridgePageHtml(deeplink), { headers: { 'Content-Type': 'text/html; charset=utf-8' } })
```

Two design points worth naming. The endpoint is intentionally public, Apple calls it, not your app, and it only relays: no account is touched here, `appleSignIn` still cryptographically verifies the `id_token` (JWKS signature, `iss`, `aud`, `exp`) before anything is minted. And a direct browser visit with no Apple POST gets a styled "nothing to see here" page rather than an error dump. With this function in place, `public/native-callback.html` goes back to being Google-only.

## The callback layer, three cooperating pieces

One context rule keeps this layer honest: the Chrome Custom Tab and the WebView are separate browsers with separate `sessionStorage`. Everything that crosses between them travels in the URL, the bridge page fires the deeplink from the tab, and the boot capture and `/auth` stash run in the WebView after the deeplink lands. Never try to read the WebView's `sessionStorage` from the callback or bridge page, it is a different browser and the value is not there.

The Android bridge leg does not hand tokens back through one file. Each provider has its return point, Google the static `native-callback.html`, Apple the `appleCallback` function above, and then two client pieces catch the deeplink. Miss any one and the token silently vanishes.

### 1\. public/native-callback.html, Google's return page

Google redirects the in-app browser here with `?code=...&state=<scheme>` in the query. Apple no longer touches this file, its Return URL is the `appleCallback` function. This static page forwards the raw code through the `oauth/` deeplink and offers a tappable fallback in case the custom scheme needs a user gesture. It must be a real static file at `public/native-callback.html`, not a route your framework renders.

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Signing you in…</title>
</head>
<body style="margin:0;background:#0d0c10;color:#f4f2ee;font-family:-apple-system,sans-serif;text-align:center">
  <p style="margin-top:42vh;font-size:15px">Signing you in…</p>
  <p><a id="continue" style="display:none;color:#ff5c1f;font-size:16px;font-weight:600;text-decoration:none">Continue to app</a></p>
  <script>
    // Google OAuth callback — authorization code flow.
    // The single-use code arrives as ?code=...&state=<deeplink scheme>.
    var q = new URLSearchParams(location.search);
    var code = q.get('code') || '';
    var error = q.get('error') || '';
    var scheme = q.get('state') || '';

    // IMPORTANT: keep the code RAW in the forwarded query. Google codes only
    // contain URL-safe chars (alphanumerics, '/', '-', '_') and '/' is legal in
    // a query value. Percent-encoding it here breaks native handoff: some
    // native layers re-encode '%' sequences ("4%2F0A…" → "4%252F0A…"), so the
    // app sends a double-encoded code and Google rejects it as malformed.
    var query = code ? 'code=' + code : 'error=' + encodeURIComponent(error || 'no_token');

    if (scheme && /^[a-z][a-z0-9+.-]*$/i.test(scheme)) {
      // Native: hand the token back to the app via its deep link (Despia path
      // oauth/auth). NO same-origin fallback here — this page runs inside the
      // OAuth in-app browser, and loading /auth in it would exchange (burn)
      // the single-use code before the real app gets to use it.
      var deeplink = scheme + '://oauth/auth?' + query;
      location.href = deeplink;
      // Custom schemes can require a user gesture — offer a tappable fallback.
      var btn = document.getElementById('continue');
      btn.href = deeplink;
      btn.style.display = 'inline-block';
    } else {
      // Web (no scheme in state): same origin — continue to the in-app /auth page.
      location.replace('/auth?' + query);
    }
  </script>
</body>
</html>
```

The `oauth/` prefix is not optional: `myapp://oauth/auth` closes the in-app browser and reopens your app at `/auth`, while `myapp://auth` without it does nothing and strands the user in the browser.

### 2\. deeplinkToken.js, boot-time capture

When Despia reopens the app, the deeplink lands as a history URL change on any path, often `/`, and the WebView does not reload. If your app has a protected-root guard, it can bounce to `/login` before React reads the token. So capture it before React mounts, in `main.jsx`, stash it in `sessionStorage`, and strip it from the visible URL. This helper reads the query and the hash, so it catches Google's `code` and Apple's `id_token` alike.

This is the OAuth subset of the real helper, which also reads a `token`/`access_token` used by the password-reset deep link. For Apple and Google you need `code`, `id_token`, and `error`:

```javascript
const KEY = 'pending_oauth'

// Read token/code/id_token/error from the current URL, query or hash.
export function readFromUrl() {
  const query = new URLSearchParams(window.location.search.replace(/^\?/, ''))
  const hash = new URLSearchParams(window.location.hash.replace(/^#/, ''))
  const code = query.get('code') || hash.get('code')          // Google auth-code flow
  const idToken = query.get('id_token') || hash.get('id_token') // Apple
  const fullName = query.get('full_name') || hash.get('full_name') // Apple first-auth name (Android leg)
  const error = query.get('error') || hash.get('error')
  return { code: code || null, idToken: idToken || null, fullName: fullName || null, error: error || null }
}

// Call once at startup, before React mounts. Stash any incoming token and
// clean the URL so it is not left lying around.
export function captureIncomingToken() {
  const { code, idToken, fullName, error } = readFromUrl()
  if (code || idToken || error) {
    try { sessionStorage.setItem(KEY, JSON.stringify({ code, idToken, fullName, error })) } catch { /* ignore */ }
    try { window.history.replaceState(null, '', window.location.pathname) } catch { /* ignore */ }
  }
}

// Consume the stash once, then clear it.
export function consumePendingToken() {
  try {
    const raw = sessionStorage.getItem(KEY)
    if (!raw) return { code: null, idToken: null, fullName: null, error: null }
    sessionStorage.removeItem(KEY)
    return JSON.parse(raw)
  } catch {
    return { code: null, idToken: null, fullName: null, error: null }
  }
}

// True if a token is waiting, in the stash or still in the live URL. Used by
// main.jsx to decide whether to normalize the route to /auth before mounting.
export function hasPendingToken() {
  if (sessionStorage.getItem(KEY)) return true
  const { code, idToken, error } = readFromUrl()
  return !!(code || idToken || error)
}
```

Wire it at the very top of `main.jsx`, before the React root renders:

```javascript
import { captureIncomingToken, hasPendingToken } from '@/lib/deeplinkToken'

// if a token landed on any path, stash it and normalize to /auth before mounting
if (hasPendingToken()) {
  captureIncomingToken()
  window.history.replaceState(null, '', '/auth')
}
```

This is the essence of the wiring, trimmed for the article. The template's real `main.jsx` does a little more: it also captures the token on non-`/auth` paths and treats `/oauth/auth` itself as an auth route, so copy the full version from the repo rather than this excerpt when you build.

### 3\. /auth, consume the stash or the live URL

`/auth` reads whatever `deeplinkToken.js` stashed at boot, and if nothing was stashed it reads the live URL, because on the native leg the token can arrive as a later history change while `/auth` is already mounted. So it also watches `popstate`, `hashchange`, and a short poll, and gives up after a timeout. A `code` routes to your Google exchange, an `id_token` routes to `appleSignIn`.

```jsx
import { useEffect, useRef, useState } from 'react'
import { useNavigate } from 'react-router-dom'
import * as customAuth from '@/lib/customAuth'
import { readFromUrl, consumePendingToken } from '@/lib/deeplinkToken'

// stashed at boot, or still sitting in the live URL, check both
function extractFromUrl() {
  const stashed = consumePendingToken()
  if (stashed.code || stashed.idToken || stashed.error) return stashed
  return readFromUrl()
}

export default function Auth() {
  const navigate = useNavigate()
  const [status, setStatus] = useState('Signing you in...')
  const handled = useRef(false)

  useEffect(() => {
    const tryExtract = () => {
      if (handled.current) return true
      const { code, idToken, fullName, error } = extractFromUrl()
      if (error) {
        handled.current = true
        setStatus('Sign-in error: ' + error)
        setTimeout(() => navigate('/login'), 3000)
        return true
      }
      if (!code && !idToken) return false
      handled.current = true

      // id_token means Apple, code means Google, both mint YOUR JWT server-side
      const done = idToken ? customAuth.loginWithAppleToken(idToken, fullName || '')
                           : customAuth.loginWithGoogleCode(code)
      done.then(() => navigate('/', { replace: true }))
          .catch((err) => {
            setStatus('Sign-in failed: ' + (err?.message || 'unknown'))
            setTimeout(() => navigate('/login'), 3000)
          })
      return true
    }

    if (tryExtract()) return
    // native WebView: the token arrives via a later history change, keep watching
    window.addEventListener('popstate', tryExtract)
    window.addEventListener('hashchange', tryExtract)
    const poll = setInterval(tryExtract, 300)
    const giveUp = setTimeout(() => {
      if (!handled.current) { setStatus('No sign-in token received.'); setTimeout(() => navigate('/login'), 3000) }
    }, 15000)
    return () => {
      window.removeEventListener('popstate', tryExtract)
      window.removeEventListener('hashchange', tryExtract)
      clearInterval(poll); clearTimeout(giveUp)
    }
  }, [navigate])

  return <p>{status}</p>
}
```

`loginWithAppleToken` and `loginWithGoogleCode` are thin wrappers that invoke your `appleSignIn` and `googleSignIn` functions and store the JWT they return through `setToken`. iOS and web never touch this page, they get the `id_token` straight from the SDK popup and exchange it inline.

## The appleSignIn function mints your JWT

This is where Apple differs most from the Google function, and where it is stricter. Apple's `id_token` reaches your backend from the client, so it must be cryptographically verified, not just decoded. The function fetches Apple's public JWKS, verifies the RS256 signature with Web Crypto, then checks `iss` is `https://appleid.apple.com`, `aud` is your Services ID (blocks token substitution), and `exp` is still valid.

Then it resolves the account: look up by `apple_id` first, fall back to `email` so an Apple sign-in links to an existing account instead of duplicating it, and create if neither matches. Two Apple quirks the function absorbs: the name arrives only on the very first sign-in, from the popup on iOS and web and from `appleCallback` on Android, which is why the client passes `fullName` along, and the email can be a private relay address, so treat it as opaque. If Apple sends no email at all, the function synthesizes a stable placeholder from the `sub` (`apple-<sub>@apple.local`), and it takes `email_verified` from the token claim rather than assuming it. It ends exactly like `googleSignIn`, minting the same HS256 JWT with your `JWT_SECRET`, so every downstream function verifies Apple and Google sessions identically. One optional hardening step for the full OIDC pattern, and note it is not in the template's reference code, so you would be adding it: `AppleID.auth.init` accepts a `nonce`, so you can generate a random one per attempt, pass it in, and have `appleSignIn` check the `nonce` claim in the verified token, which ties each `id_token` to the sign-in that requested it.

Add `apple_id` as a string field on the `Account` entity, keep the deny-all RLS, and the client sends `{ apple_id_token, full_name }` to the function from the popup callback on iOS and web, or from `/auth` on Android, where the deeplink lands with `id_token` in the query and the same watch-until-it-shows extraction you built for Google picks it up. The template carries the complete function, verification included, so lift it rather than retyping the crypto.

The template goes further than a plain sign-in, on the same `id_token` verification: `authLinkAccount` attaches an Apple identity to an anonymous guest account and merges into an existing one if the Apple identity or its verified email already exists, and `authReauth` re-verifies the Apple identity before an account deletion. The Android path sets an `apple_link_mode` flag before the deeplink round-trip so `/auth` links instead of signing in fresh. The full step-by-step, including the Developer Console screenshots and a troubleshooting table, is in `docs/APPLE_SIGN_IN.md` in the template at github.com/despia-native/base44-native-oauth.

## Ship both logins together

With Apple riding the same JWT system, your app offers Google and Apple sign-in from one `Account` entity, one session model, and one verification path, which is exactly the combination iOS review expects to see. Code signing and submission run from the browser, no Mac and no CLI. The full working reference, both auth functions, the shared callback, and the vault, is the template at github.com/despia-native/base44-native-oauth.

[See the setup docs at setup.despia.com](https://setup.despia.com/native-features/oauth/apple)