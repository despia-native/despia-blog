---
title: "Base44 Google Login (Native Sign-In)"
seoTitle: "Base44 Google Login (Native Sign-In)"
seoDescription: "Base44 Google login done natively: Base44 auth cannot mint the token, so run your own JWT auth, open the Despia OAuth bridge, and sign in on mobile."
datePublished: 2026-07-11T13:18:49.353Z
cuid: cmrge4ru500000akv8pw3h78x
slug: base44-google-login-native-sign-in
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/0ad13e9b-1d10-49a1-91cf-7de5f9131c20.png
tags: webview, google-login, base44

---

Base44 builds the app, but two things block Google sign-in on mobile. A Google button that works in the browser fails inside a WebView, because Google refuses OAuth in embedded views and the App Store expects a real browser session. And Base44's built-in auth cannot mint the session token the native flow needs to hand back to your app. The fix for both has the same shape: run your own auth on Base44's backend, and use Despia's native `oauth://` bridge to open the secure system browser, ASWebAuthenticationSession on iOS and Chrome Custom Tabs on Android. One flow covers both platforms, with a normal redirect on web.

> **Heads up, this is an advanced setup.** You are replacing Base44's built-in auth with your own: backend functions, your own JWTs, and an `Account` entity you manage. It is the right path for native Google login on Base44, but it is more involved than a typical Base44 feature, so budget the time and lean on the template rather than typing it all from scratch. If you only ship on the web, Base44's built-in Google login is simpler and perfectly fine, you only need this because native apps do.

## Why the web Google button fails in a mobile app

Google blocks its consent screen inside an embedded WebView, the `disallowed_useragent` error, because an embedded view could read the user's credentials, and Apple's guidelines agree. So the plain button either errors or is rejected at review. Despia's `oauth://` bridge steps out of the WebView: it opens the system browser, the user authenticates, and Despia closes it and returns the result. Unlike Apple Sign In, Google uses the same bridge on iOS and Android, so there is no platform split in your sign-in code.

## Why Base44's built-in Google auth cannot do this

Base44 gives you one Google entry point, `base44.auth.loginWithProvider('google')`, and it is a sealed hosted flow: a full-page redirect to Base44's own broker, using Base44's Google client on Base44's domain, ending by writing a session into the browser that ran it. Every layer a native app needs to reach is locked:

*   **The redirect URI is Base44's, not yours.** Native Google sign-in has to run in the system browser, and the only way back into the app is a custom-scheme deep link. Base44 always redirects to its own callback and finishes in Safari or Chrome, a different cookie jar from your WebView. That session cannot cross into the app.
    
*   **There is no token-exchange endpoint.** Even after you run Google yourself and hold a verified credential, Base44 has no API to trade it for a session. Firebase has `signInWithCredential`, Supabase has `signInWithIdToken`, Auth0 has token exchange. Base44 has none.
    
*   **You cannot create users in code.** `base44.entities.User.create()` returns 405, even from service-role code. The only path is an emailed invite through Base44's hosted signup, so a first-time Google sign-in cannot produce a platform user.
    
*   **There is no session-minting API.** Even for a user who exists, nothing issues a session for them server-side. Sessions only fall out of the hosted flow.
    
*   **Sessions are browser-storage bound.** A native app wants the credential in secure storage so it survives WebView purges and Face ID re-entry. Base44 sessions cannot be exported there.
    

| What a native login needs | Base44 API | Status |
| --- | --- | --- |
| Custom OAuth redirect (deep link) | none | locked to platform callback |
| Exchange a Google credential for a session | none | no endpoint |
| Create a user in code | `User.create` | 405, invite-only |
| Mint a session for a known user | none | no API |
| Credential in secure storage | none | browser storage only |
| Privileged DB access behind checks | `asServiceRole` | open, this is the way in |

The one capability Base44 leaves open is service-role backend functions, and that is exactly where you rebuild auth. You run Google's authorization-code flow yourself, keep users in your own entity, and mint your own JWT, all still on Base44's backend. Leaving the built-in auth is all or nothing: once Google login is custom, email and password are yours too, and you can skip Google entirely and ship email and password alone. We keep a full working template of this at github.com/despia-native/base44-native-boilerplate, worth a read for the exact functions, the client glue, and the security rules.

## The data model

One entity. `Account` is your user table, matched on `email`, with `google_id` set for Google users and `password_hash` for password users. Base44 adds `id`, `created_date`, and `updated_date`, so you define the rest. The `rls` block at the end matters as much as the fields, more on it below.

```json
{
  "name": "Account",
  "type": "object",
  "properties": {
    "email":          { "type": "string", "description": "Lowercased, trimmed, the unique identity" },
    "full_name":      { "type": "string" },
    "password_hash":  { "type": "string", "description": "PBKDF2 salt:hash hex, empty for Google-only accounts" },
    "google_id":      { "type": "string", "description": "Google 'sub', set when linked to Google" },
    "avatar_url":     { "type": "string" },
    "email_verified": { "type": "boolean", "default": false },
    "role":           { "type": "string", "enum": ["user", "admin"], "default": "user" },
    "last_login_at":  { "type": "string", "format": "date-time" }
  },
  "required": ["email"],
  "rls": {
    "create": { "user_condition": { "role": "__service_only__" } },
    "read":   false,
    "update": { "user_condition": { "role": "__service_only__" } },
    "delete": false
  }
}
```

## Lock the database, or this is not secure

This is the part that makes the whole setup safe, so do not skip it. Because your users hold your JWT rather than a platform session, they are anonymous to Base44's data layer. Any permissive row-level security would expose the entity to the whole internet, and `Account` holds password hashes and identity ids. So every entity gets the deny-all `rls` block above: `read` and `delete` are hard platform denials, and `create` and `update` are gated on a role no user can ever have, which denies them for everyone. Backend functions bypass RLS through `base44.asServiceRole`, and that is the single sanctioned data path:

frontend, to a backend function, which verifies your JWT, then touches the database through `asServiceRole`.

Never call `base44.entities.X` from the frontend, for any entity, and give every future entity the same deny-all block from its first version. With that rule in place, authorization lives in exactly one place, your functions, which verify the JWT and enforce roles in code.

## Backend helpers

Your JWT is a hand-rolled HS256 token, no external dependency, signed with a `JWT_SECRET` env var. Passwords use PBKDF2 through Web Crypto, stored as `salt:hash` hex.

```javascript
function b64url(input) {
  const str = typeof input === 'string' ? input : String.fromCharCode(...new Uint8Array(input))
  return btoa(str).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '')
}
function fromB64url(str) {
  str = str.replace(/-/g, '+').replace(/_/g, '/')
  while (str.length % 4) str += '='
  return atob(str)
}

async function signJwt(payload, secret, ttl = 60 * 60 * 24 * 30) {
  const now  = Math.floor(Date.now() / 1000)
  const data = `${b64url(JSON.stringify({ alg: 'HS256', typ: 'JWT' }))}.${b64url(JSON.stringify({ ...payload, iat: now, exp: now + ttl }))}`
  const key  = await crypto.subtle.importKey('raw', new TextEncoder().encode(secret), { name: 'HMAC', hash: 'SHA-256' }, false, ['sign'])
  const sig  = await crypto.subtle.sign('HMAC', key, new TextEncoder().encode(data))
  return `${data}.${b64url(sig)}`
}

async function verifyJwt(token, secret) {
  const [h, b, s] = (token || '').split('.')
  if (!s) return null
  const key = await crypto.subtle.importKey('raw', new TextEncoder().encode(secret), { name: 'HMAC', hash: 'SHA-256' }, false, ['verify'])
  const sig = Uint8Array.from(fromB64url(s), c => c.charCodeAt(0))
  if (!(await crypto.subtle.verify('HMAC', key, sig, new TextEncoder().encode(`${h}.${b}`)))) return null
  const payload = JSON.parse(fromB64url(b))
  return payload.exp && payload.exp < Math.floor(Date.now() / 1000) ? null : payload
}
```

```javascript
const toHex = (buf) => [...new Uint8Array(buf)].map(x => x.toString(16).padStart(2, '0')).join('')

async function pbkdf2(password, salt) {
  const km = await crypto.subtle.importKey('raw', new TextEncoder().encode(password), 'PBKDF2', false, ['deriveBits'])
  return crypto.subtle.deriveBits({ name: 'PBKDF2', salt, iterations: 100000, hash: 'SHA-256' }, km, 256)
}
async function hashPassword(password) {
  const salt = crypto.getRandomValues(new Uint8Array(16))
  return `${toHex(salt)}:${toHex(await pbkdf2(password, salt))}`
}
// constant-time compare so a wrong password does not leak timing
function timingSafeEqual(a, b) {
  if (a.length !== b.length) return false
  let diff = 0
  for (let i = 0; i < a.length; i++) diff |= a.charCodeAt(i) ^ b.charCodeAt(i)
  return diff === 0
}
async function verifyPassword(password, stored) {
  const [saltHex, hashHex] = (stored || '').split(':')
  if (!hashHex) return false
  const salt = Uint8Array.from(saltHex.match(/.{2}/g).map(h => parseInt(h, 16)))
  return timingSafeEqual(toHex(await pbkdf2(password, salt)), hashHex)
}
```

## Wire the Google button

Install the SDK with `npm install despia-native`. Both native and web fetch the same OAuth URL, Despia just opens it differently.

```javascript
import despia from 'despia-native'
import { base44 } from '@/api/base44Client'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

async function signInWithGoogle() {
  const { url } = (await base44.functions.invoke('googleAuthUrl', { deeplink_scheme: 'myapp' })).data

  if (isDespia) {
    despia(`oauth://?url=${encodeURIComponent(url)}`) // open in the secure browser
  } else {
    window.location.href = url                         // web: full-page redirect
  }
}
```

Replace `myapp` with your scheme from Despia > Publish > Deeplink.

## Function one: googleAuthUrl

Builds the Google URL using the authorization-code flow. Google returns a single-use code to `native-callback.html`, and the deeplink scheme rides along in `state` so the redirect URI stays clean.

```javascript
// Base44 function: googleAuthUrl
const { deeplink_scheme } = await req.json()

// explicit APP_BASE_URL secret, else auto-detect from the calling app's origin
const APP_BASE_URL = Deno.env.get('APP_BASE_URL') || req.headers.get('origin')
// must exactly match what is registered in Google Cloud Console, no query params
const redirectUri = `${APP_BASE_URL}/native-callback.html`

const url = 'https://accounts.google.com/o/oauth2/v2/auth?' + new URLSearchParams({
  client_id:     Deno.env.get('GOOGLE_CLIENT_ID'),
  redirect_uri:  redirectUri,
  response_type: 'code',
  scope:         'openid email profile',
  state:         deeplink_scheme, // so the callback knows which scheme to fire
})

return Response.json({ url })
```

The code that comes back is worthless on its own: it is single-use and can only be exchanged with the client secret, which never leaves your backend. That is why this flow is the defensible one, the browser and the deeplink only ever carry a dead-end credential.

One simplification to know about: OAuth `state` normally doubles as a random CSRF value, and here it carries the deeplink scheme instead. The exposure is limited because the code is useless without your server-held secret, but if you want the stricter version, generate a random `state`, store `{ state, deeplink_scheme }` server-side for a few minutes, and look it up during the exchange instead of trusting the value in the URL.

## Add public/native-callback.html

This tiny page runs inside the secure browser. It receives `?code=...&state=<scheme>` from Google and fires the deeplink that closes the browser and reopens your app with the code. On web, where there is no scheme in `state`, it continues to `/auth` on the same origin instead. Keep it as a plain HTML file in `public/`, not a React component.

```html
<!-- public/native-callback.html -->
<!DOCTYPE html>
<html lang="en">
<head><meta charset="UTF-8" /><title>Signing you in...</title></head>
<body>
<p>Signing you in...</p>
<p><a id="continue" style="display:none">Continue to app</a></p>
<script>
  var q      = new URLSearchParams(location.search)
  var code   = q.get('code') || ''
  var error  = q.get('error') || ''
  var scheme = q.get('state') || ''

  // keep the code RAW: Google codes are URL-safe, and percent-encoding here
  // gets double-encoded by some native layers, so Google rejects the exchange
  var query = code ? 'code=' + code : 'error=' + encodeURIComponent(error)

  if (scheme && /^[a-z][a-z0-9+.-]*$/i.test(scheme)) {
    // native: oauth/ tells Despia to close the tab and reopen the app at /auth
    var deeplink = scheme + '://oauth/auth?' + query
    location.href = deeplink
    // custom schemes can require a user gesture, so offer a tappable fallback
    var btn = document.getElementById('continue')
    btn.href = deeplink
    btn.style.display = 'inline-block'
  } else {
    // web: no scheme in state, continue to /auth on the same origin
    location.replace('/auth?' + query)
  }
</script>
</body>
</html>
```

Two details here earn their place. The code is forwarded raw, because Google codes contain `/` and some native layers re-encode `%` sequences, turning a valid code into a double-encoded one Google rejects. And there is no same-origin fallback on the native path, since loading `/auth` inside the OAuth browser would burn the single-use code before the real app gets it.

The `oauth/` prefix is not optional. `myapp://oauth/auth` closes the session and reopens the app at `/auth`. `myapp://auth` without it does nothing, and the user is stuck in the browser.

## Handle the code on /auth

When Despia reopens the app, the code arrives as a history URL change, not a page load, and it can land on any path. So read it from the URL, keep watching until it shows, then exchange it for your JWT.

```jsx
import { useEffect, useRef } from 'react'
import { useNavigate } from 'react-router-dom'
import * as customAuth from '@/lib/customAuth'

function Auth() {
  const navigate = useNavigate()
  const done = useRef(false)

  useEffect(() => {
    const tryExtract = () => {
      if (done.current) return
      const q = new URLSearchParams(location.search)
      const code = q.get('code')
      if (!code) return
      done.current = true

      // exchange the single-use Google code for OUR JWT
      customAuth.loginWithGoogleCode(code)
        .then(() => navigate('/'))
        .catch(() => navigate('/login'))
    }

    tryExtract() // may already be in the URL on mount
    // in the WebView the deeplink changes the URL without a reload, so keep observing
    window.addEventListener('popstate', tryExtract)
    window.addEventListener('hashchange', tryExtract)
    const poll = setInterval(tryExtract, 300)
    return () => {
      window.removeEventListener('popstate', tryExtract)
      window.removeEventListener('hashchange', tryExtract)
      clearInterval(poll)
    }
  }, [navigate])

  return <p>Signing you in...</p>
}
```

`loginWithGoogleCode` invokes the `googleSignIn` function below and stores the JWT it returns.

One more edge worth guarding: static hosts can collapse a deep-linked path like `/oauth/auth?code=...` down to `/` before React mounts, and then a protected-root guard bounces the user to `/login` before the code is ever read. The fix lives in `main.jsx`: before React mounts, check the URL for a code, stash it in `sessionStorage`, and rewrite the URL to `/auth`, a public route. `Auth` then consumes the stashed code as well as reading it live.

## Function two: googleSignIn

Exchanges the code server-side, where the client secret lives, reads the verified identity from Google's `id_token`, upserts the `Account`, and returns your own JWT. This is where the session is minted.

```javascript
// Base44 function: googleSignIn, exchanges the code and returns OUR JWT
import { createClientFromRequest } from 'npm:@base44/sdk'

let { google_code } = await req.json()
if (!google_code) return Response.json({ error: 'google_code is required' }, { status: 400 })
// some native layers re-encode the deeplink, real Google codes never contain '%'
if (google_code.includes('%')) {
  try { google_code = decodeURIComponent(google_code) } catch { /* keep as-is */ }
}

const APP_BASE_URL = Deno.env.get('APP_BASE_URL') || req.headers.get('origin')
const tokenRes = await fetch('https://oauth2.googleapis.com/token', {
  method:  'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: new URLSearchParams({
    code:          google_code,
    client_id:     Deno.env.get('GOOGLE_CLIENT_ID'),
    client_secret: Deno.env.get('GOOGLE_CLIENT_SECRET'),
    redirect_uri:  `${APP_BASE_URL}/native-callback.html`, // must match googleAuthUrl exactly
    grant_type:    'authorization_code',
  }),
})
const tokenData = await tokenRes.json()
if (!tokenRes.ok || !tokenData.id_token) {
  return Response.json({ error: tokenData.error_description || 'Google code exchange failed' }, { status: 401 })
}

// id_token came from Google's token endpoint in this server-side exchange,
// so validate the claims that matter before trusting it
const info = JSON.parse(atob(tokenData.id_token.split('.')[1].replace(/-/g, '+').replace(/_/g, '/')))
if (!info.email) return Response.json({ error: 'Google account has no email' }, { status: 401 })
if (!['accounts.google.com', 'https://accounts.google.com'].includes(info.iss)) {
  return Response.json({ error: 'Unexpected token issuer' }, { status: 401 })
}
if (info.exp && info.exp < Math.floor(Date.now() / 1000)) {
  return Response.json({ error: 'Token expired' }, { status: 401 })
}
// the id_token must be issued FOR THIS APP, blocks token substitution
if (info.aud !== Deno.env.get('GOOGLE_CLIENT_ID')) {
  return Response.json({ error: 'Token not issued for this app' }, { status: 401 })
}

const email  = info.email.toLowerCase().trim()
const base44 = createClientFromRequest(req)

const [existing] = await base44.asServiceRole.entities.Account.filter({ email })
let account = existing
if (!account) {
  account = await base44.asServiceRole.entities.Account.create({
    email,
    full_name:      info.name || email.split('@')[0],
    google_id:      info.sub || '',
    avatar_url:     info.picture || '',
    email_verified: true,
    role:           'user',
    last_login_at:  new Date().toISOString(),
  })
} else {
  // keep the Google link and verified state fresh on returning logins
  await base44.asServiceRole.entities.Account.update(account.id, {
    google_id:      account.google_id || info.sub || '',
    avatar_url:     account.avatar_url || info.picture || '',
    email_verified: true,
    last_login_at:  new Date().toISOString(),
  })
}

const token = await signJwt({ sub: account.id, email: account.email, role: account.role }, Deno.env.get('JWT_SECRET'))
return Response.json({ token, account: { id: account.id, email: account.email, full_name: account.full_name, role: account.role } })
```

## Store the session in the Despia Storage Vault

Keep the JWT in `localStorage` for immediate use, and mirror it into Despia's native Storage Vault, which is backed by the platform's secure storage, so the session survives WebView storage purges, app updates, and reinstalls. That is what lets you offer Face ID re-entry.

```javascript
import despia from 'despia-native'

const VAULT_KEY = 'app_session_token'
const isNative  = () => navigator.userAgent.toLowerCase().includes('despia')

export function setToken(token) {
  localStorage.setItem('app_auth_token', token)
  if (isNative()) despia(`setvault://?key=${VAULT_KEY}&value=${encodeURIComponent(token)}&locked=false`)
}

// on startup, restore a session the vault kept across reinstalls
export async function restoreToken() {
  const local = localStorage.getItem('app_auth_token')
  if (local || !isNative()) return local
  const data  = await despia(`readvault://?key=${VAULT_KEY}`, [VAULT_KEY])
  const token = data?.[VAULT_KEY] ? decodeURIComponent(data[VAULT_KEY]) : null
  if (token) localStorage.setItem('app_auth_token', token)
  return token
}
```

## Verify the session on protected calls

Every function that needs a user reads the token, verifies it, and loads the account through `asServiceRole`, since the deny-all RLS makes that the only path to data. Base44's `createClientFromRequest` only recognises platform tokens, so you verify your own.

```javascript
// Base44 function: guard a protected endpoint
const token   = req.headers.get('x-app-token') || (await req.json().catch(() => ({}))).token
const payload = await verifyJwt(token, Deno.env.get('JWT_SECRET'))
if (!payload) return Response.json({ error: 'unauthorized' }, { status: 401 })

const base44  = createClientFromRequest(req)
const account = await base44.asServiceRole.entities.Account.get(payload.sub)
// account is the signed-in user, proceed
```

## Email and password on the same system

Because you own the stack, email and password are two more functions minting the same JWT, no bridge and no callback, since password login never leaves the app. This is also the whole story if you skip Google.

```javascript
// Base44 function: authRegister
const { email, password, full_name } = await req.json()
if (!email || !password || password.length < 8) return Response.json({ error: 'invalid input' }, { status: 400 })

const clean  = email.toLowerCase().trim()
const base44 = createClientFromRequest(req)
if ((await base44.asServiceRole.entities.Account.filter({ email: clean })).length) {
  return Response.json({ error: 'email already registered' }, { status: 409 })
}

const account = await base44.asServiceRole.entities.Account.create({
  email: clean, full_name: full_name || clean.split('@')[0],
  password_hash: await hashPassword(password), role: 'user', last_login_at: new Date().toISOString(),
})
return Response.json({ token: await signJwt({ sub: account.id, email: clean, role: 'user' }, Deno.env.get('JWT_SECRET')) })
```

```javascript
// Base44 function: authLogin
const { email, password } = await req.json()
const base44 = createClientFromRequest(req)
const [account] = await base44.asServiceRole.entities.Account.filter({ email: email.toLowerCase().trim() })

// one generic error, so you never reveal which emails exist
if (!account || !(await verifyPassword(password, account.password_hash))) {
  return Response.json({ error: 'invalid email or password' }, { status: 401 })
}
return Response.json({ token: await signJwt({ sub: account.id, email: account.email, role: account.role }, Deno.env.get('JWT_SECRET')) })
```

On the client both are plain invokes, and the returned token goes through the same `setToken` as Google, so the vault and verification treat every session identically.

## Set up Google in the Cloud Console

1.  In console.cloud.google.com, open APIs and Services, then Credentials, and create an OAuth 2.0 Client ID of type Web application.
    
2.  Add your app domain under Authorized JavaScript origins.
    
3.  Under Authorized redirect URIs, add your callback file exactly, `https://your-app.base44.app/native-callback.html`.
    
4.  Copy the Client ID and Client Secret into Base44 as `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET`. The secret stays server-side, it is what makes the stolen-code scenario a dead end. Also set `APP_BASE_URL` to your app's public URL and a `JWT_SECRET` for signing sessions. Make that one at least 32 bytes of real randomness, `openssl rand -base64 48` does it, because anyone holding it can forge a session for any user, and rotating it signs everyone out at once.
    
5.  Set your deeplink scheme from Despia > Publish > Deeplink and replace `myapp` in the code.
    

## The things that trip people up

Base44's built-in auth cannot mint the session, so you run your own JWT auth on the `Account` entity, which also gives you password login. Every entity needs the deny-all RLS block, or your data is open to the internet. The `oauth/` prefix must be in the deeplink, or Despia never closes the browser. The redirect URI in Google Cloud must match `native-callback.html` exactly, and the code must travel raw through the deeplink or double-encoding breaks the exchange. And on `/auth` the code arrives as a history URL change with no reload, so watch `popstate`, `hashchange`, and a short poll rather than reading only on mount.

## Ship it with native sign-in built in

One store rule to plan for while you are here: under Guideline 4.8, an iOS app offering Google login must also offer a privacy-focused login option, and in practice that means Sign in with Apple, which App Review looks for by name. It rides the same custom-JWT system, one more auth function and the same `Account` entity.

Take your Base44 app to iOS and Android with Google, Apple, and password sign-in built for review. Code signing and submission run from the browser, no Mac and no CLI. The template at github.com/despia-native/base44-native-boilerplate is the full working reference for everything in this post and the rest of the store-compliance surface around it: Apple Sign-In, anonymous guest accounts, in-app account deletion, password reset, and the deny-all RLS rules, all on the same JWT auth.

[See the setup docs at setup.despia.com](https://setup.despia.com/native-features/oauth/google)