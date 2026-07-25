---
title: "Base44 App Store Rejection: Guideline 4.2 and 4.8"
seoTitle: "Base44 App Store Rejection: Guideline 4.2 and 4.8"
seoDescription: "A Base44 App Store rejection usually comes down to Guideline 4.8 and 4.2, not a hidden button. Here is what triggers each one and the full fix."
datePublished: 2026-07-22T13:27:20.162Z
cuid: cmrw4a3bm00000aj39olg48va
slug: base44-app-store-rejection-guideline-4-2-and-4-8
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/ca728310-9e71-413c-8b0d-ff044aed6e04.png
tags: ios-app-development, appstore, base44

---

Your Base44 app builds fine, uploads fine, and Apple rejects it. Again. The rejection cites Guideline 4.8 and points at the Google sign in. So you disabled the Google button, or wired it to redirect somewhere else, and the next rejection came back with the same reason. The problem is not the button. It is what the button is attached to, and hiding it does not move the review. This post explains what actually triggers the rejection and walks the full fix, with the working code and a template you can clone.

## What Apple is actually rejecting

Guideline 4.8 is a rule about login services, not about which buttons are visible. If your app uses a third-party or social login (Google, Facebook, and the rest) to set up the user's primary account, Apple requires you to also offer an equivalent login option that meets three privacy conditions: it limits data collection to the user's name and email, it lets the user keep that email address private, and it does not collect interactions with the app for advertising without consent.

In practice, the only mainstream login that satisfies all three is Sign in with Apple. So when Apple says "you offer a third-party login but no equivalent option," what it means is: you ship Google, and nothing next to it clears that privacy bar.

## Why disabling the Google button changes nothing

The auth in a Base44 app is Google-first at the backend. Hiding the Google button, or pointing it somewhere else, is a front-end change. The OAuth client, the provider config, and the flow behind it are all still wired in, and the reviewer can still reach them. Reviewers test the login, read your metadata and screenshots, and inspect the account setup. They reject on the service the account uses, not on the pixel you moved. 4.8 is an account-architecture rule, not a UI toggle, and you cannot style your way out of it.

## The second rejection hiding behind the first

Even if you clear 4.8, there is a second wall from the same root. Base44's export is a basic web view of your published site, and Apple's Guideline 4.2 rejects apps that add nothing beyond the website. Base44's own documentation is direct that native-only features like push notifications are not supported by the export. So you can pass the login rejection and still trip on 4.2. The store wants a native app and the export is a website in a frame. Both rejections come from the same place.

## The fix, and the Base44 catch nobody warns you about

The equivalent login Apple wants is the native Sign in with Apple sheet, not a web button styled to look like one. That part is standard. The catch is specific to Base44: its built-in auth cannot mint a session for a native sign-in flow. There is no token-exchange endpoint, `User.create` returns 405 even from backend code, and sessions only fall out of Base44's own hosted browser flow. So you cannot flip Apple on in the builder and be done.

What you do instead is run your own auth on Base44's backend. You keep an `Account` entity, you verify the sign-in server-side, and you mint your own JWT. Apple, Google, and email all plug into that one system. It is more involved than a checkbox, which is why the rejection loop feels unwinnable from inside the builder, but it is a solved problem with a fixed shape.

## Start from the template

You do not have to build this from scratch. We keep a store-review-ready Base44 starter that already handles this, and the rest of what Apple and Google reject hybrid apps for:

[github.com/despia-native/base44-native-boilerplate](https://github.com/despia-native/base44-native-boilerplate)

It is a React, Vite, and Tailwind project that runs on Base44 and ships as a native app through Despia. What it already carries maps directly onto the rejection surface:

*   Native Sign in with Apple next to Google, on a custom JWT system (`src/lib/appleAuth.js`, the `appleSignIn` function), which is the 4.8 fix.
    
*   A native-first UI system, safe areas, iOS transitions, sheets, and haptics, so the app does not read as a wrapped website, which is the 4.2 fix.
    
*   Automatic guest accounts, so the app is usable before anyone signs up (`src/lib/deviceAuth.js`), which keeps you clear of the force-registration rejection under 5.1.1.
    
*   In-app account deletion with a two-step confirmation (`docs/ACCOUNT_DELETION.md`), required under 5.1.1(v).
    
*   Deny-all row-level security on every entity, with all data access behind authenticated functions (`docs/DB_SECURITY.md`).
    

Clone it, run `base44 dev`, and pushed changes sync back into the Base44 Builder. The documentation lives in `docs/`, with `docs/JWT_AUTH.md` and `docs/APPLE_SIGN_IN.md` as the two to read first. The rest of this post is the condensed version of what ships in that repo, so you can see the moving parts before you lift them.

## Step 1: an Account entity locked to service-role only

Your users will hold your JWT, not a Base44 session, so they are anonymous to Base44's data layer. Any permissive row-level security would expose this entity, and it holds identity ids. So every field is defined by you and the `rls` block denies everything except your backend functions.

```json
{
  "name": "Account",
  "type": "object",
  "properties": {
    "email":          { "type": "string", "description": "Lowercased, the unique identity" },
    "full_name":      { "type": "string" },
    "password_hash":  { "type": "string", "description": "Empty for Google and Apple accounts" },
    "google_id":      { "type": "string", "description": "Google 'sub', set when linked" },
    "apple_id":       { "type": "string", "description": "Apple 'sub', set when linked" },
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

Never call `base44.entities.Account` from the front end. Every data path goes front end, to a backend function, which verifies your JWT, then reads through `base44.asServiceRole`. That is what keeps authorization in one place.

## Step 2: mint your own session

The session is a hand-rolled HS256 JWT signed with a `JWT_SECRET`, no external dependency. This is the helper every sign-in function ends with, straight from the template.

```javascript
async function signJwt(payload, secret, expiresInSec = 60 * 60 * 24 * 30) {
  const now  = Math.floor(Date.now() / 1000)
  const body = { ...payload, iat: now, exp: now + expiresInSec }
  const enc  = new TextEncoder()
  const data = `${b64url(JSON.stringify({ alg: 'HS256', typ: 'JWT' }))}.${b64url(JSON.stringify(body))}`
  const key  = await crypto.subtle.importKey('raw', enc.encode(secret), { name: 'HMAC', hash: 'SHA-256' }, false, ['sign'])
  const sig  = await crypto.subtle.sign('HMAC', key, enc.encode(data))
  return `${data}.${b64url(sig)}`
}
```

The matching `verifyJwt`, the `b64url` helpers, and PBKDF2 password hashing all ship in the template.

## Step 3: native Sign in with Apple, three platforms, two flows

On iOS and on the web, Apple's own JS SDK in popup mode opens the native Apple sheet, Face ID, Touch ID, or the device passcode, directly inside the app. No redirect, no browser session to manage. Android has no native Apple sheet, so it runs the same login through Despia's `oauth://` bridge in a Chrome Custom Tab, and a backend callback hands the token back through a deeplink. This is `src/lib/appleAuth.js` in full.

```javascript
import despia from 'despia-native'
import { base44 } from '@/api/base44Client'
import { appConfig } from '@/config/app-config'

const ua = navigator.userAgent.toLowerCase()
const isDespia = ua.includes('despia')
const isDespiaAndroid = isDespia && ua.includes('android')

// iOS and web return { idToken, fullName }, Android returns null and finishes on /auth
export async function signInWithApple() {
  if (isDespiaAndroid) {
    // no native Apple sheet on Android, run it in Chrome Custom Tabs
    const res = await base44.functions.invoke('appleAuthUrl', { deeplink_scheme: appConfig.deeplinkScheme })
    despia(`oauth://?url=${encodeURIComponent(res.data.url)}`)
    return null
  }

  window.AppleID.auth.init({
    clientId: appConfig.appleServicesId,       // set in src/config/app-config.js
    scope: 'name email',
    redirectURI: window.location.origin + '/', // must match the registered Return URL
    usePopup: true,                            // redirect mode blanks the screen and gets rejected
  })

  const response = await window.AppleID.auth.signIn()
  // Apple sends the name only on the first sign-in
  const name = response.user?.name
  const fullName = name ? `${name.firstName || ''} ${name.lastName || ''}`.trim() : ''
  return { idToken: response.authorization.id_token, fullName }
}
```

`usePopup: true` is not a preference. Redirect mode blanks the screen inside a WebView, which is its own rejection, so this line is load-bearing.

## Step 4: verify Apple's token on the backend and mint your JWT

Apple's `id_token` reaches your backend from the client, so it must be cryptographically verified, not just decoded. The `appleSignIn` function fetches Apple's public keys, verifies the RS256 signature, then confirms the issuer is Apple, the audience is your Services ID, and the token has not expired. Only then does it resolve the account and mint your session.

```javascript
// base44/functions/appleSignIn, condensed. verifyAppleIdToken (JWKS + RS256) ships in the file.
import { createClientFromRequest } from 'npm:@base44/sdk'

const { apple_id_token, full_name } = await req.json()
if (!apple_id_token) return Response.json({ error: 'apple_id_token is required' }, { status: 400 })

const clientId = Deno.env.get('APPLE_SERVICES_ID')
let payload
try {
  // iss === https://appleid.apple.com, aud === your Services ID, exp still valid
  payload = await verifyAppleIdToken(apple_id_token, clientId)
} catch (e) {
  return Response.json({ error: e.message }, { status: 401 })
}

const base44 = createClientFromRequest(req)
// Apple can hand back a private relay address, and no email on later sign-ins,
// so synthesize a stable fallback from the subject when it is missing
const email = (payload.email || `apple-${payload.sub}@apple.local`).toLowerCase().trim()

// match by apple_id first, then email, so Apple links an existing account
let [account] = await base44.asServiceRole.entities.Account.filter({ apple_id: payload.sub })
if (!account) [account] = await base44.asServiceRole.entities.Account.filter({ email })
if (!account) {
  account = await base44.asServiceRole.entities.Account.create({
    email, full_name: full_name || 'Apple User', apple_id: payload.sub,
    email_verified: !!payload.email_verified, role: 'user',
    last_login_at: new Date().toISOString(),
  })
}

const token = await signJwt({ sub: account.id, email: account.email, role: account.role }, Deno.env.get('JWT_SECRET'))
return Response.json({ token })
```

The name arrives only on the very first sign-in, which is why the client passes `full_name` through. The audience check is the one that blocks token substitution, so do not drop it.

## Step 5: Apple Developer Console, and no .p8 needed

The best part of this setup: Sign in with Apple as a login needs no `.p8` key. It is the OpenID Connect `id_token` flow, so your backend verifies Apple's token against Apple's public keys and mints your own session. A `.p8` only enters later if you add in-app account deletion and need to revoke Apple access.

1.  Open your App ID at developer.apple.com and enable the Sign In with Apple capability.
    
2.  Create a Services ID, for example `com.yourcompany.yourapp.appleauth`. This exact string is your `clientId`. An App ID does not work as the client id, using one gives `invalid_client`.
    
3.  Configure the Services ID: enable Sign In with Apple, set the Primary App ID to your App ID, add your domain, and register two Return URLs, `https://` required, exact match: `https://yourapp.com/` for the popup on iOS and web, and `https://yourapp.com/functions/appleCallback` for the Android flow.
    
4.  Set the Services ID in two places, kept identical: `appleServicesId` in `src/config/app-config.js`, which the popup sends to Apple, and the `APPLE_SERVICES_ID` secret in Base44, which `appleSignIn` checks the token against. Also set `APP_BASE_URL` and reuse the `JWT_SECRET` from your auth. No `.p8`, no client secret.
    

One thing that wastes an afternoon: the Base44 editor preview runs on a different origin than your published app, which is not a registered Return URL, so Apple sign in always fails there with `invalid_request`. Test on the published URL.

## Add Google on the same system

Google rides the same `Account` entity and the same JWT. The one difference is that Google blocks its consent screen inside a WebView, so it always runs through Despia's `oauth://` bridge in the system browser on both platforms, and your backend exchanges the code for the identity (`googleAuthUrl` and `googleSignIn`). The template ships both, and `docs/GOOGLE_LOGIN_BASE44_LIMITATIONS.md` explains why Base44's built-in Google login could not be used.

## What passing review looks like

With Apple and Google both minting the same JWT from one `Account` entity, your app offers a compliant equivalent login next to Google, which is exactly what App Review checks for under 4.8. Guest accounts keep the app usable without a forced sign-up, in-app deletion satisfies 5.1.1(v), and the native UI clears the 4.2 wrapped-website concern. Web content still updates over the air, so later changes to your login page ship without another submission.

One thing to add before you open sign-up to the public: the template's auth endpoints ship without per-IP or per-email rate limiting, so put an attempt counter with a cooldown in front of login, register, and password reset. It is noted in the repo, and it is the one gap worth closing before launch.

## When you do not need Sign in with Apple at all

Guideline 4.8 has exemptions. If your app uses only your own email and password account system, with no Google or social login anywhere, Sign in with Apple is not required, and the same holds for education or enterprise apps using an existing institutional account. On Base44 that still runs on the custom auth above, minus the Google and Apple functions. If you do not need social login, drop it and ship email and password alone. Most apps keep Google though, and the moment it stays, native Sign in with Apple is the option that clears the rejection for good.

## Get it on the stores

Clone the [template](https://github.com/despia-native/base44-native-boilerplate), wire in native Sign in with Apple, and ship your Base44 app to iOS and Android from one codebase. Code signing and submission run from the browser, no CLI and no Mac required.

[See the setup docs at setup.despia.com](https://setup.despia.com)