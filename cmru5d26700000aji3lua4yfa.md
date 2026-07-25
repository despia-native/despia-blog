---
title: "Vue.js Apple Login (Native Sign-In)"
seoTitle: "Vue.js Apple Login (Native Sign-In)"
seoDescription: "Vue.js Apple login done natively: the Apple JS SDK shows the native sign-in sheet on iOS, Android rides the OAuth bridge, one handler covers all."
datePublished: 2026-07-21T04:22:05.915Z
cuid: cmru5d26700000aji3lua4yfa
slug: vue-js-apple-login-native-sign-in
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/666dc77d-c4d9-4c46-a206-c8932950431d.png
tags: vuejs, oauth, webview, webapp, pwa, vue-js

---

Ship a Vue app to iOS with Google or any social login and Guideline 4.8 obliges you to offer an equivalent privacy-focused login next to it, one that limits data collection and supports private email, and Sign in with Apple is usually the simplest compliant choice, unless one of Guideline 4.8's exceptions applies. Apple's exceptions include proprietary account systems, enterprise and education accounts, and government identity systems. The upside: Apple is the easy provider. Its JS SDK with `usePopup: true` can present the native Apple sign-in sheet from the WebView in Despia, so iPhone needs no browser session, no bridge, and no callback file. Android has no equivalent and rides Despia's `oauth://` bridge, reusing the exact plumbing from Google login if you have it.

## Three platforms, two flows

On iOS, `AppleID.auth.signIn()` with `usePopup: true` opens the native sheet in place and resolves with the `id_token` in your JS callback. On the web, the identical call opens a browser popup. On Android, your backend builds an OAuth URL, `despia('oauth://?url=...')` opens Chrome Custom Tabs, and the session returns through the `oauth/` deeplink like every other bridge flow.

## Apple Developer Console setup

1.  Under Certificates, Identifiers and Profiles, open your App ID and enable Sign In with Apple.
    
2.  Create a Services ID, for example `com.yourcompany.yourapp.webauth`. That is the `clientId` your code uses. The Services ID drives web and JS Sign in with Apple, while the App ID capability from step 1 enables the native app side.
    
3.  Configure the Services ID: enable Sign In with Apple, set the Primary App ID, add your domain, and register the Return URLs, comma-delimited, `https://` required, no wildcards, exact string match with the trailing slash included. Which URLs depends on your Android variant, and mixing them up is the number one rejection cause, so keep the three different URLs in this setup straight. Apple's registered Return URL is where Apple itself returns. Your `redirect_to` is where Supabase forwards afterwards, allowlisted in Supabase, not Apple. The deeplink is how the result re-enters the app, and Apple never sees it.
    
    *   Always register `https://yourapp.com/` for the JS SDK popup on iOS and web.
        
    *   Direct Apple variant: also register your callback endpoint, `https://yourapp.com/api/auth/apple-callback`, since that is the `redirect_uri` your Android URL sends.
        
    *   Supabase-brokered variant: also register Supabase's callback, `https://YOUR-PROJECT.supabase.co/auth/v1/callback`, because Apple returns to Supabase, not to you. Your `native-callback.html` is never registered with Apple in this variant, it only goes in Supabase's redirect allowlist as the `redirect_to` target.
        
4.  The `.p8` question splits by variant, so here it is as two explicit branches. Direct Apple `id_token` verification, the popup and the direct Android leg in this guide: no `.p8` and no client secret, the token is verified against Apple's public keys, and that alone is sufficient to cryptographically authenticate the Apple login and create your own application session. Supabase-brokered Apple OAuth: a `.p8` signing key is required, Supabase uses it to generate the client secret, which expires and must be regenerated at least every six months. One production planning note for the direct branch: authentication does not need the authorization code, but revocation-ready account deletion does. Apple recommends exchanging the code at first sign-in, which requires a `.p8` client secret, and storing the refresh token so your in-app account deletion can call Apple's revoke endpoint automatically; without stored tokens Apple documents a manual fallback, delete the account data and direct the user to revoke access themselves. So automatic deletion-time revocation requires adding the `.p8`\-backed code exchange during initial account creation, even though authenticating the `id_token` itself never depends on it.
    
5.  If Supabase is your auth provider, add your Services ID to the Apple provider's authorized client IDs under Authentication, Providers, Apple, so `signInWithIdToken` accepts tokens issued for it. The `id_token` method needs no client secret. (The client-secret JWT is only for Supabase's server-side OAuth redirect flow, and it expires within six months, another reason to prefer the `id_token` flow here.)
    

## Load the SDK and detect the platform

Apple's script goes before your app bundle in `index.html`, and it only initializes over HTTPS, not plain localhost. The HTTPS rule has one Despia-specific consequence: the Local Server offline mode serves your app over HTTP, so Apple sign-in cannot initialize there. If your app uses it, open the Despia Editor and switch Offline Support from Native to PWA, then add a service worker that caches your assets. That keeps offline support through the service worker while giving Apple's SDK the secure HTTPS origin it requires.

The snippets below show the complete authentication flow, but intentionally leave application-specific database, session, routing, and error-handling functions for you to connect to your existing stack.

```html
<script src="https://appleid.cdn-apple.com/appleauth/static/jsapi/appleid/1/en_US/appleid.auth.js"></script>
```

Install the SDK with `npm install despia-native` and derive the three constants once:

```javascript
const ua = navigator.userAgent.toLowerCase()
const isDespia = ua.includes('despia')
const isDespiaIOS = isDespia && (ua.includes('iphone') || ua.includes('ipad'))
const isDespiaAndroid = isDespia && ua.includes('android')
const isWeb = !isDespia
```

## One method for all three platforms

iOS and web share the SDK popup and hand the `id_token` to your backend to verify. Android fetches an OAuth URL and opens the bridge. `usePopup: true` is mandatory: the redirect variant blanks the WebView and fails App Review.

```javascript
import despia from 'despia-native'

function toBase64Url(bytes) {
  let binary = ''
  for (const byte of bytes) binary += String.fromCharCode(byte)
  return btoa(binary).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '')
}

async function signInWithApple() {
  // iOS + Web: Apple's JS SDK popup (native sheet on iOS, browser popup on web)
  if (isDespiaIOS || isWeb) {
    // the backend issues the nonce and remembers it, so the token is bound
    // to a sign-in the server itself started
    const { nonce } = await fetch('/api/auth/apple-nonce', { method: 'POST' }).then(r => r.json())
    window.AppleID.auth.init({
      clientId: 'com.yourcompany.yourapp.webauth', // Your Services ID
      scope: 'name email',
      redirectURI: window.location.origin + '/',   // Must match Apple config
      usePopup: true,                              // Required inside a WebView
      nonce,
    })

    let response
    try {
      response = await window.AppleID.auth.signIn()
    } catch (error) {
      const code = error?.error || error?.code || error?.message
      // cancelling is not a failure, swallow it instead of rejecting
      if (code === 'user_cancelled_authorize' || code === 'popup_closed_by_user') return
      throw error
    }
    const idToken = response.authorization.id_token
    // Apple only sends the name on the very first sign-in
    const name = response.user?.name
    const fullName = name ? `${name.firstName || ''} ${name.lastName || ''}`.trim() : ''

    // Send the id_token to your backend, with the nonce it must match
    await exchangeAppleToken(idToken, fullName, nonce)
  }

  // Android, direct variant: no native sheet exists, continue via Chrome Custom
  // Tabs + deeplink. If Supabase brokers your auth, use the shorter branch below instead
  if (isDespiaAndroid) {
    // PKCE-style verifier: another app squatting the custom scheme could grab
    // the handoff code first, but without this verifier the code is useless.
    // RFC 7636's recommended construction, 32 random bytes as unpadded base64url
    const verifierBytes = crypto.getRandomValues(new Uint8Array(32))
    const verifier = toBase64Url(verifierBytes) // 43 chars, 256 bits
    // WebView sessionStorage: invisible to the Chrome Custom Tab, and that is
    // fine, only /auth back in this same WebView ever needs it
    sessionStorage.setItem('apple_verifier', verifier)
    const digest = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(verifier))
    const challenge = toBase64Url(new Uint8Array(digest))

    // the deeplink scheme itself stays a server-side constant, only the challenge goes up
    const res = await fetch('/api/auth/apple-url', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ code_challenge: challenge }),
    })
    if (!res.ok) throw new Error('could not start Apple sign-in')
    const { url } = await res.json()
    despia(`oauth://?url=${encodeURIComponent(url)}`)
    // sign-in resumes on /auth when the deeplink lands
  }
}
```

With a generic backend, `exchangeAppleToken` posts the `id_token` to your own endpoint, which verifies it against Apple's public keys and returns your session. Verification is non-negotiable, the token arrives from the client, so the endpoint must prove Apple issued it for your app before trusting a single claim. On Node, `jose` does the whole job, signature against Apple's JWKS, expiry, issuer, and audience:

```javascript
// backend: verify the Apple id_token before creating any session
import { createRemoteJWKSet, jwtVerify } from 'jose'

const appleJwks = createRemoteJWKSet(new URL('https://appleid.apple.com/auth/keys'))

async function verifyAppleIdToken(idToken) {
  const { payload } = await jwtVerify(idToken, appleJwks, {
    issuer: 'https://appleid.apple.com',
    audience: 'com.yourcompany.yourapp.webauth', // your Services ID, blocks token substitution
    algorithms: ['RS256'], // pin the alg, which also keeps the SHA-256 c_hash check consistent
  })
  return payload // payload.sub is the stable user id, payload.email when granted
}

// Apple's name arrives as raw browser data, it is NOT inside the signed token,
// so validate and sanitize it before it touches storage
function sanitizeAppleName(value) {
  if (typeof value !== 'string') return ''
  return value
    .normalize('NFC')
    .replace(/[\u0000-\u001F\u007F]/g, '')
    .trim()
    .slice(0, 120)
}
```

`jwtVerify` checks the RS256 signature and `exp` for you, the options pin `iss` and `aud`. The popup's nonce comes from the server, a tiny endpoint issues it and remembers it in the same pending store the Android leg uses:

```javascript
// node:crypto, not the browser global: createHash and timingSafeEqual below
// exist only here, and without them c_hash and verifier checks throw
import crypto from 'node:crypto'
// npm install cookie-parser, without it req.cookies is undefined and the
// nonce check below rejects every popup sign-in
import cookieParser from 'cookie-parser'
app.use(cookieParser())

// shared by every backend endpoint in this guide, declare it once, above them all
const pending = new Map() // key -> { ...transaction fields, createdAt }
const TTL = 5 * 60 * 1000

// POST /api/auth/apple-nonce, issues a single-use nonce bound to THIS browser session
app.post('/api/auth/apple-nonce', (req, res) => {
  const nonce = crypto.randomUUID()
  const txnId = crypto.randomUUID()
  pending.set(nonce, { txnId, createdAt: Date.now() })
  res.cookie('apple_txn', txnId, { httpOnly: true, secure: true, sameSite: 'lax', maxAge: TTL })
  res.set('Cache-Control', 'no-store') // never cache credentials material
  res.json({ nonce })
})
```

With the helper in place, here is the complete endpoint: verify, consume the server-issued nonce, find or create the user, and mint your own short-lived session. Nothing is trusted before verification, and lookups go by `sub`, never email, since Apple emails can be private relay addresses that users can rotate or withhold.

```javascript
// POST /api/auth/apple, the endpoint exchangeAppleToken calls
app.post('/api/auth/apple', express.json(), async (req, res) => {
  const { id_token, full_name, nonce } = req.body

  let claims
  try {
    claims = await verifyAppleIdToken(id_token)
  } catch {
    return res.status(401).json({ error: 'invalid apple token' })
  }
  // the nonce must be one this server issued, unconsumed, inside the token, AND
  // requested by this same browser session, so a stolen token plus its visible
  // nonce cannot be redeemed from anywhere else. Same rule as the handoff:
  // verify first, consume only on success, so a failed guess cannot burn it
  const issued = nonce && pending.get(nonce)
  if (!issued || Date.now() - issued.createdAt > TTL) {
    pending.delete(nonce)
    return res.status(401).json({ error: 'nonce mismatch' })
  }
  if (claims.nonce !== nonce || issued.txnId !== req.cookies.apple_txn) {
    return res.status(401).json({ error: 'nonce mismatch' })
  }
  pending.delete(nonce) // consume only after all checks pass

  let user = await db.users.findByAppleSub(claims.sub)
  if (!user) {
    user = await db.users.create({
      apple_sub: claims.sub,
      // Apple can withhold the email entirely; store null rather than
      // manufacturing one, claims.sub is the real identity
      email: typeof claims.email === 'string' && claims.email ? claims.email : null,
      name: sanitizeAppleName(full_name),                        // raw browser data, sanitize before storing
      // Apple sends email_verified as a boolean OR the string 'true'
      email_verified: claims.email_verified === true || claims.email_verified === 'true',
    })
  }

  const token = await signSession({ sub: user.id }) // your own short-lived session token
  // mirror the set options, clearCookie only matches with the same path and domain
  res.clearCookie('apple_txn', { httpOnly: true, secure: true, sameSite: 'lax' })
  res.set('Cache-Control', 'no-store') // the response carries the session token
  res.json({ token })
})
```

On the client, `exchangeAppleToken` is the thin caller:

```javascript
async function exchangeAppleToken(idToken, fullName = '', nonce = '') {
  const res = await fetch('/api/auth/apple', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ id_token: idToken, full_name: fullName, nonce }), // name only exists on the first sign-in
  })
  if (!res.ok) throw new Error('apple sign-in failed')
  const { token } = await res.json()
  saveSession(token) // store your session, then your app treats the user as signed in
}
```

Supabase is a common shortcut here, since it verifies the Apple `id_token` for you. If that is your backend, generate a nonce, pass it to `AppleID.auth.init`, and hand the same nonce to `signInWithIdToken`, which is what Supabase's own docs recommend:

```javascript
const nonce = crypto.randomUUID()
window.AppleID.auth.init({
  clientId: 'com.yourcompany.yourapp.webauth',
  scope: 'name email',
  redirectURI: window.location.origin + '/',
  usePopup: true,
  nonce,
})
const response = await window.AppleID.auth.signIn()
await supabase.auth.signInWithIdToken({ provider: 'apple', token: response.authorization.id_token, nonce })
```

Firebase's `signInWithCredential` fills the same slot. The rest of this guide stays backend-agnostic.

Catch and swallow `user_cancelled_authorize`, the cancellation error Apple documents, and you may also handle `popup_closed_by_user` defensively for older integrations. Either means the person dismissed the sheet. Historical note: iOS 17.1 briefly rendered an HTML form instead of the native sheet inside WKWebView, fixed in 17.2, with sign-in working throughout.

## The Android backend handler

The Android endpoint builds an Apple authorize URL with your `apple-callback` API route as the redirect, a random single-use `state` binding the round-trip, and a server-generated `nonce` that Apple echoes inside the signed token. Shown as Express, one runtime for every endpoint in this guide, adapt the route wiring to your framework:

```javascript
// one Express backend serves all five auth endpoints in this guide, adapt the
// route wiring if your framework differs. pending and TTL are the same ones
// declared with the nonce endpoint above, not a second store

// POST /api/auth/apple-url
app.post('/api/auth/apple-url', express.json(), (req, res) => {
  // never accept the scheme from the client, it is your app's own server constant
  const { code_challenge } = req.body || {}
  // an S256 challenge is 32 bytes as unpadded base64url, exactly 43 chars
  if (typeof code_challenge !== 'string' || !/^[A-Za-z0-9_-]{43}$/.test(code_challenge)) {
    return res.status(400).json({ error: 'invalid code challenge' })
  }
  const state = crypto.randomUUID() // random, single-use, binds the round-trip
  const nonce = crypto.randomUUID() // Apple echoes this inside the signed id_token
  pending.set(state, { nonce, code_challenge, createdAt: Date.now() })
  res.set('Cache-Control', 'no-store') // the URL carries live transaction material

  const url = 'https://appleid.apple.com/auth/authorize?' + new URLSearchParams({
    client_id: 'com.yourcompany.yourapp.webauth', // your Services ID
    redirect_uri: 'https://yourapp.com/api/auth/apple-callback', // the form_post endpoint below, registered with Apple
    response_type: 'code id_token',
    response_mode: 'form_post', // required with name/email scopes, Apple POSTs the result
    scope: 'name email',
    state,
    nonce,
  })

  res.json({ url })
})
```

Apple POSTs the result to the `apple-callback` endpoint below, which verifies it, creates your session, parks that session behind a verifier-bound one-time handoff code, and deeplinks only the code into the app. `/auth` redeems it over HTTPS.

If your backend is Supabase, its implicit flow is the simplest setup for a WebView-based mobile app, and it is simple for a structural reason. The sign-in crosses several contexts, the WebView, the secure browser, and back, and implicit means the session survives every hop with zero server code on your side: Supabase brokers Apple, mints its own access and refresh token pair, and delivers both straight through the hash and the deeplink, so `/auth` installs a full self-refreshing session with one `setSession()` call. No callback endpoint, no exchange, no pending store, the static `native-callback.html` is the entire return path. Point the broker at it with your scheme in the `redirect_to`, URL-encoded and allowlisted in Supabase, `redirect_to=https://yourapp.com/native-callback.html?deeplink_scheme=yourcompany`, and do not lean on Supabase's `state` for the scheme, Supabase owns that parameter for its own transaction. This is Supabase's documented mobile deep-linking pattern and the right default for most apps; the trade is that the token pair rides the custom scheme, so if your threat model demands the strictest bar, the verifier-bound direct flow above is the upgrade path.

On the client that replaces the whole Android branch, no verifier, no challenge, no endpoint of your own:

```javascript
// Android, Supabase-brokered variant: swap this in for the direct branch above
if (isDespiaAndroid) {
  // the scheme travels inside redirect_to, and this exact URL must be allowlisted
  // in Supabase under Authentication, URL Configuration
  const redirectTo = encodeURIComponent(
    'https://yourapp.com/native-callback.html?deeplink_scheme=yourcompany'
  )
  const url = `${SUPABASE_URL}/auth/v1/authorize?provider=apple&redirect_to=${redirectTo}`
  despia(`oauth://?url=${encodeURIComponent(url)}`)
  return // Supabase returns its token pair to native-callback.html, /auth installs it
}
```

One Apple rule shapes this endpoint, so it deserves a plain statement. Apple hard-requires `response_mode: 'form_post'` the moment you request the `name` or `email` scope, rejecting anything else with `invalid_request`, and a static callback file cannot read a POST body. So the Return URL for this leg is a backend endpoint, and that is what keeps the name and email scopes on Android: Apple POSTs `id_token`, `state`, and, on the very first authorization only, a `user` JSON field carrying the name. The endpoint validates the `state` against the pending store, verifies the token, checks the nonce it saved for this exact round-trip, mints your session, parks it under a one-time handoff code, and answers with a tiny HTML bridge page that deeplinks only that code, so no bearer token, Apple's or yours, ever rides the custom scheme. On Node:

```javascript
const APP_SCHEME = process.env.APP_DEEPLINK_SCHEME // your app name in lowercase, no spaces, exactly as shown in Despia > Publish > Deeplink
if (!/^[a-z][a-z0-9+.-]*$/i.test(APP_SCHEME || '')) throw new Error('APP_DEEPLINK_SCHEME missing or malformed') // fail at startup, not mid-login

// POST target registered with Apple as the redirect_uri
app.post('/api/auth/apple-callback', express.urlencoded({ extended: true }), async (req, res) => {
  const { id_token, state, error, user } = req.body // user JSON arrives on the FIRST auth only

  // expiring state: unknown or stale round-trips stop here. Verify first,
  // consume after all checks pass, deleting up front would let anyone who
  // learns a valid state burn the legitimate login with a fake POST
  const txn = pending.get(state)
  if (!txn || Date.now() - txn.createdAt > TTL) {
    pending.delete(state)
    return res.status(400).send('unknown or expired state')
  }

  res.set('Cache-Control', 'no-store') // the bridge page carries a one-time code
  // fire the deeplink automatically, and always render a tappable fallback:
  // some browsers block automatic custom-scheme navigation and would otherwise
  // strand the user on a blank page
  const bridge = (q) => {
    const link = APP_SCHEME + '://oauth/auth?' + q
    return '<!DOCTYPE html><html><body style="font-family:-apple-system,sans-serif;text-align:center">' +
      '<p style="margin-top:42vh">Signing you in&hellip;</p>' +
      '<p><a href="' + link + '">Continue to app</a></p>' +
      '<script>location.href="' + link + '"</script></body></html>'
  }
  if (!id_token) return res.send(bridge('error=' + encodeURIComponent(error || 'no_token')))

  let claims
  try {
    claims = await verifyAppleIdToken(id_token) // same jose helper as the popup path
  } catch {
    return res.send(bridge('error=invalid_token'))
  }
  // this nonce was generated server-side for THIS round-trip, so a token minted
  // for any other sign-in, app, or replay fails right here
  if (claims.nonce !== txn.nonce) return res.send(bridge('error=nonce_mismatch'))

  // hybrid response: the code is always returned and c_hash is required, so
  // treat their absence as an invalid response, fail closed. Validate the
  // binding. This guide then leaves the code unredeemed, the verified id_token
  // already proves identity and redemption needs a .p8 client secret; see the
  // revocation note below before shipping account deletion
  const { code } = req.body
  if (!code || !claims.c_hash) return res.send(bridge('error=invalid_hybrid_response'))
  const digest = crypto.createHash('sha256').update(code).digest()
  const cHash = digest.subarray(0, digest.length / 2).toString('base64url')
  if (cHash !== claims.c_hash) return res.send(bridge('error=c_hash_mismatch'))

  let fullName = ''
  try {
    const u = JSON.parse(user || '')
    if (u?.name) fullName = sanitizeAppleName(`${u.name.firstName || ''} ${u.name.lastName || ''}`)
  } catch { /* normal after the first sign-in */ }

  const account = await findOrCreateByAppleSub(claims, fullName) // same upsert as the popup endpoint
  const token = await signSession({ sub: account.id })           // your own short-lived session

  // never deeplink a bearer token. Park the session under a random one-time
  // handoff code with a short TTL, bound to the app's PKCE challenge,
  // deeplink only the code
  const handoff = crypto.randomUUID()
  pending.set(handoff, { session: token, code_challenge: txn.code_challenge, createdAt: Date.now() })

  // consume the state only now, after the account, session, and handoff all
  // exist, so an infrastructure hiccup between validation and here cannot eat
  // a valid login. In Redis, atomically mark the state claimed right after
  // cryptographic verification and keep the steps below idempotent, so two
  // identical callbacks cannot race through account creation
  pending.delete(state)

  res.send(bridge('handoff_code=' + encodeURIComponent(handoff)))
})

// POST /api/auth/handoff, the app trades the one-time code for the session over HTTPS
app.post('/api/auth/handoff', express.json(), (req, res) => {
  const { code, verifier } = req.body
  if (typeof code !== 'string' || typeof verifier !== 'string') {
    return res.status(400).json({ error: 'invalid request' })
  }
  const txn = pending.get(code)
  if (!txn?.session || Date.now() - txn.createdAt > TTL) {
    pending.delete(code) // expired entries can go
    return res.status(401).json({ error: 'invalid or expired code' })
  }
  // PKCE check: an interceptor may have the code, only the real app has the
  // verifier. Verify BEFORE consuming, deleting on a failed guess would let an
  // interceptor burn the legitimate handoff and lock the real app out
  const challenge = crypto.createHash('sha256').update(verifier, 'ascii').digest('base64url')
  const a = Buffer.from(challenge)
  const b = Buffer.from(txn.code_challenge || '')
  if (a.length !== b.length || !crypto.timingSafeEqual(a, b)) {
    return res.status(401).json({ error: 'verifier mismatch' })
  }
  pending.delete(code) // consume only after successful proof
  res.set('Cache-Control', 'no-store')
  res.json({ token: txn.session })
})
```

In this direct flow, everything that matters is bound server-side: the scheme is your own constant, the `state` is random and single-use, the nonce lives only in the pending store and inside Apple's signed token, and the deeplink carries nothing but a one-time code that expires in minutes, works once, and is useless without the verifier, which never travels through Apple's browser flow or the custom-scheme deeplink and is disclosed only to your own backend over HTTPS at redemption. Custom-scheme URLs can be claimed by other apps, which is exactly why no bearer credential travels through one here, and why the code alone is not enough to redeem. Production hygiene for the pending store: use Redis or a database with native TTL expiry rather than an in-memory map, sweep abandoned entries, rate limit the public `apple-nonce`, `apple-url`, and `handoff` endpoints, since anyone can call them, and in a distributed store claim `state`, nonce, and handoff records atomically, a Redis transaction or Lua script rather than separate get and delete calls, so two racing requests cannot redeem the same record. One naming footnote for precision: the verifier and challenge here are PKCE-style protection for the app handoff, applied by your own backend, Apple's authorization server is not running PKCE in this flow. The popup nonce is bound to the browser session through the HttpOnly `apple_txn` cookie above, so only the session that requested it can redeem it, keep that binding, it is load-bearing. Two footnotes. If you skip the name and email scopes entirely, Apple permits `response_mode: 'fragment'` and a static callback file works, a lighter setup when `sub`\-based identity is enough. And if Supabase brokers your Android flow, none of this applies: Apple form\_posts to Supabase's server callback, and your static `native-callback.html` receives Supabase's token pair in the hash exactly as described.

## The callback layer, three files that work together

The Android bridge leg does not hand tokens back through one file. It is three cooperating pieces, and getting all three right is what makes native sign-in reliable. Miss any one and the session silently vanishes. iOS and web never touch this layer, they exchange the `id_token` inline from the SDK popup. This is only the Android return path, and it crosses two separate browser contexts, so keep the storage model straight. The Chrome Custom Tab and the WebView have completely separate `sessionStorage`. The verifier is written in the WebView before the tab opens, and it is read back in the WebView on `/auth` after the deeplink closes the tab, the same context both times. The tab itself never needs storage access: Apple's screen and the server-rendered bridge page only pass data through the URL, which is exactly why the design survives the context split. Do not try to read `sessionStorage` inside the callback or bridge page, it is a different browser and the value is not there. One edge case follows from this: if Android kills the app process while the tab is open, the WebView's `sessionStorage` dies with it, the verifier is gone, and the handoff redemption fails with the expired error, the user simply signs in again.

With the form\_post bridge above, the direct Apple leg returns through the `apple-callback` endpoint, not this file, and the deeplink carries only the one-time `handoff_code`. The static file below is the return page for the Supabase-brokered variant, where the hash carries the access and refresh token pair, and the boot-capture and `/auth` pieces are shared by both, they just read different field names from the deeplink.

### 1\. public/native-callback.html, the return page

The in-app browser redirects here. Supabase returns its own access and refresh token pair in the URL hash. This static page reads the hash, forwards the tokens through the `oauth/` deeplink, and offers a tappable fallback for schemes that need a user gesture. It must be a real static file at `public/native-callback.html`, not a route your framework renders.

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
    // Supabase returns the session in the URL hash: #access_token=...&refresh_token=...
    // The app scheme is carried explicitly in the allowlisted redirect_to query,
    // never taken from Supabase's state, Supabase owns that parameter.
    var q = new URLSearchParams(location.search);
    var hash = new URLSearchParams(location.hash.slice(1));

    var scheme = q.get('deeplink_scheme') || '';
    var accessToken = hash.get('access_token') || '';
    var refreshToken = hash.get('refresh_token') || '';
    var error = hash.get('error') || q.get('error') || '';

    var query = accessToken
      ? 'access_token=' + encodeURIComponent(accessToken) + '&refresh_token=' + encodeURIComponent(refreshToken)
      : 'error=' + encodeURIComponent(error || 'no_token');

    if (scheme && /^[a-z][a-z0-9+.-]*$/i.test(scheme)) {
      // Native: hand the tokens to the app via its deep link (Despia path oauth/auth).
      var deeplink = scheme + '://oauth/auth?' + query;
      location.href = deeplink;
      // Custom schemes can require a user gesture — offer a tappable fallback.
      var btn = document.getElementById('continue');
      btn.href = deeplink;
      btn.style.display = 'inline-block';
    } else {
      // Web or missing deeplink_scheme: continue to the same-origin /auth page.
      location.replace('/auth?' + query);
    }
  </script>
</body>
</html>
```

The `oauth/` prefix is not optional: `yourcompany://oauth/auth` closes the in-app browser and reopens your app at `/auth`, while `yourcompany://auth` without it does nothing and strands the user in the browser.

### 2\. Boot-time capture, before the app mounts

When Despia reopens the app, the deeplink lands as a history URL change on any path, often `/`, and the WebView does not reload. If a protected-root guard runs first, it can bounce to `/login` before your code reads the tokens. So capture them before the app mounts, stash them, and normalize the URL to `/auth`.

```javascript
// run this before your app renders (main entry file)
const p = new URLSearchParams(location.search)
const h = new URLSearchParams(location.hash.slice(1))
// covers both variants: the one-time handoff code (direct Apple) and the
// Supabase token pair
const hasToken =
  p.get('handoff_code') || h.get('handoff_code') ||
  p.get('access_token') || h.get('access_token') ||
  p.get('error') || h.get('error')
if (hasToken) {
  sessionStorage.setItem('pending_oauth', location.search + location.hash)
  window.history.replaceState(null, '', '/auth')
}
```

### 3\. /auth, install the session

`/auth` reads whatever was stashed at boot, or the live URL if nothing was stashed, because on the native leg the tokens can arrive as a later history change while `/auth` is already mounted. It watches `popstate` and `hashchange` for that, then installs the session with `setSession()`, which supabase-js keeps refreshing.

```javascript
// read WITHOUT deleting: the stash and the verifier survive until the session
// is actually installed, a network blip must not destroy the only copy of the proof
function peek() {
  const stash = sessionStorage.getItem('pending_oauth')
  const src = stash || (location.search + location.hash)
  const p = new URLSearchParams(src.split('#')[0].replace(/^\?/, ''))
  const h = new URLSearchParams((src.split('#')[1] || ''))
  return {
    handoffCode: p.get('handoff_code') || h.get('handoff_code'),   // direct Apple, minted in apple-callback
    accessToken: p.get('access_token') || h.get('access_token'),   // Supabase-brokered variant
    refreshToken: p.get('refresh_token') || h.get('refresh_token') || '',
    error: p.get('error') || h.get('error'),
  }
}

function clearPending() {
  sessionStorage.removeItem('pending_oauth')
  sessionStorage.removeItem('apple_verifier')
}

let completing = false // popstate, hashchange, and the initial call can overlap

async function completeSignIn() {
  if (completing) return
  completing = true
  try {
    const { handoffCode, accessToken, refreshToken, error } = peek()
    if (error) { clearPending(); return goToLogin(error) }

    // direct Apple variant: trade the one-time code plus the secret verifier for the session
    if (handoffCode) {
      const verifier = sessionStorage.getItem('apple_verifier') || ''
      const res = await fetch('/api/auth/handoff', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ code: handoffCode, verifier }),
      })
      if (!res.ok) {
        clearPending() // the code is spent or expired, do not retry it on remount
        return goToLogin('sign-in expired, try again')
      }
      const { token } = await res.json()
      saveSession(token)
      clearPending() // only now, the session is installed
      return goHome()
    }

    // Supabase-brokered variant: install the token pair
    if (accessToken) {
      const { error: sessErr } = await supabase.auth.setSession({
        access_token: accessToken,
        refresh_token: refreshToken,
      })
      if (sessErr) {
        clearPending() // bad tokens, do not retry them on remount
        return goToLogin(sessErr.message)
      }
      clearPending() // only now, the session is installed
      return goHome()
    }
    // nothing present yet: keep watching, the token may still be arriving
  } finally {
    completing = false
  }
}

completeSignIn()
window.addEventListener('popstate', completeSignIn)
window.addEventListener('hashchange', completeSignIn) // hash-only change, popstate does not fire
```

Wrap the three functions above in your framework's page component, `goHome` and `goToLogin` are your router's navigate calls. iOS and web skip all of this, they call `signInWithIdToken` directly from the popup callback.

## The things that trip people up

`usePopup: true` is non-negotiable on iOS, redirect mode shows a blank white screen inside the WebView and fails review. The SDK script tag must load before your bundle, and only over HTTPS, which rules out the HTTP Local Server offline mode, switch to PWA offline support instead. The `redirectURI` passed to `AppleID.auth.init` must match a registered return URL character for character, trailing slash included. Apple returns the user's name only on the very first authorization, so persist it then. And the Android leg still lives or dies by the `oauth/` deeplink prefix. In the Supabase-brokered variant that deeplink is fired by the static `native-callback.html`; in the direct Apple variant it is fired by the HTML your `apple-callback` endpoint returns.

## Ship both logins together

Apple beside Google is the pairing iOS review expects, and here they share one bridge, one callback file, and one `/auth`. Take your Vue app to iOS and Android with both sign-ins built for review, from the codebase you already have. Code signing and submission run from the browser, no Mac and no CLI.

[See the setup docs at setup.despia.com](https://setup.despia.com/native-features/oauth/apple)