---
title: "Turn a Base44 Web App into a Mobile App"
seoTitle: "Turn a Base44 Web App into a Mobile App"
seoDescription: "Convert your Base44 web app into a native mobile app and publish it to the App Store and Google Play, straight from the codebase you already built."
datePublished: 2026-07-23T10:13:12.379Z
cuid: cmrxcsai500000aj83yk876qo
slug: turn-a-base44-web-app-into-a-mobile-app
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/3b3975e6-28d5-49d6-93b8-053791462a62.png
tags: webview, webapp, pwa, base44

---

Base44 builds the web app. Despia converts it into a native mobile app you publish to the App Store and Google Play, from the same codebase, with native billing, push, and 50+ device features. This walks the full path: add the package, detect the runtime, call your first native feature, and ship.

## Add the package through the Builder chat

Base44 does not have a terminal. You add npm packages by asking the Builder AI, which installs and wires them into your project. Open the Builder chat and send:

```plaintext
Install the npm package despia-native.
```

Approve the request when Base44 prompts you, then publish the app so the change takes effect. One note specific to Base44: once a package is installed it stays in the project, so there is nothing to undo if you change your mind, an unused package does not call any native feature by itself.

After it installs, import the default export wherever you call native features:

```javascript
import despia from 'despia-native'
```

Installing `despia-native` adds the JavaScript bridge. Features that need native SDKs, such as OneSignal, RevenueCat, or Stripe, still need the matching Despia integration enabled and a fresh native build.

## Detect the Despia runtime

Native calls only do something inside the Despia app. In a browser preview they are no-ops, which is what you want, but it means you should gate every call so the same code runs cleanly on the web version of your Base44 app.

For Despia apps, use the user agent check:

```javascript
const ua = navigator.userAgent.toLowerCase()
const isDespia = ua.includes('despia')
const isDespiaIOS = isDespia && (ua.includes('iphone') || ua.includes('ipad'))
const isDespiaAndroid = isDespia && ua.includes('android')
```

Put this in one shared helper and import it. Base44 regenerates component code as you prompt, so keep detection in a stable file the AI is not rewriting on every turn.

## Call your first native feature

The three call shapes cover everything, and the features worth shipping first are push, purchases, and biometric storage.

Fire-and-forget, no return value. Register the signed-in user for push on every authenticated load:

```javascript
if (isDespia) despia(`setonesignalplayerid://?user_id=${userId}`)
```

Promise, for calls that read device state. Check entitlements straight from the native store, no backend:

```javascript
async function checkEntitlements() {
    const data   = await despia('getpurchasehistory://', ['restoredData'])
    const active = (data.restoredData ?? []).filter(p => p.isActive)
    if (active.some(p => p.entitlementId === 'premium')) unlockPremium()
}
```

Callback, for native events. Assign `window.onXxx` outside the gate so the runtime can reach it, then re-check on every purchase:

```javascript
window.onRevenueCatPurchase = checkEntitlements
checkEntitlements()
```

Storage Vault is the other one to reach for early. It survives reinstall and can lock a value behind Face ID:

```javascript
await despia('setvault://?key=sessionToken&value=abc123&locked=true')

// reading a locked key triggers the biometric prompt
const { sessionToken } = await despia('readvault://?key=sessionToken', ['sessionToken'])
```

## The two things Base44 builders miss

First, changes do not go live until you hit Publish. If a native call is not firing on device, confirm you published after the last edit.

Second, get the payment rail right. Digital goods, subscriptions, premium tiers, credits, ad removal, have to go through Apple and Google billing. Stripe for digital content gets you rejected. Despia ships RevenueCat for exactly this, launched from your web code:

```javascript
if (isDespia) {
    despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)
}
```

Physical goods and real-world services are the opposite case. Apple and Google take no cut and require Stripe there, and using In-App Purchase for a shipped product or a booking gets you rejected too. If your Base44 app sells merch, delivery, or appointments, route those through Stripe:

```javascript
async function checkoutCart() {
    const res = await fetch('/api/create-payment-intent', { method: 'POST' })
    const { publishable_key, client_secret } = await res.json()
    if (isDespia) {
        despia(`stripe://payment?publishable_key=${publishable_key}&payment_intent_client_secret=${client_secret}`)
    }
}
```

## Let Base44 write correct Despia code

The single biggest thing you can do for code accuracy is connect the Despia MCP. It feeds the Builder AI the real API surface so it writes correct scheme names instead of guessing. Add it to your account once, under your profile at Settings, MCP connections, Add custom MCP. Name it Despia, set the URL to `https://setup.despia.com/mcp`, leave authentication as Not required, then Test and add. This needs a Builder plan or higher.

After that, a prompt like "register the signed-in user for push with despia-native" generates the real call instead of an invented one.

If you cannot use MCP, the second-best option is to paste the relevant feature pages from `setup.despia.com` straight into the Builder chat before you prompt. Same source of truth, more manual.

## Start from the boilerplate

Heads up, this is an advanced setup. You are replacing Base44's built-in auth with your own: backend functions, your own JWTs, and an `Account` entity you manage. It is the right path for native Google login on Base44, but it is more involved than a typical Base44 feature, so budget the time and lean on the template rather than typing it all from scratch. If you only ship on the web, Base44's built-in Google login is simpler and perfectly fine, you only need this because native apps do.

The boilerplate is a production-grade, store-review-ready starter, not just an auth snippet. It runs on Base44's backend (serverless functions, entities, email) and already handles the points Apple and Google most often reject hybrid apps for, a strong foundation rather than a guarantee of approval: native Google Sign-In and Sign in with Apple, guest device accounts so the app is usable without forcing signup, in-app account deletion, deny-all row-level security on every entity, push, RevenueCat, and an iOS-style UI. Auth is fully self-owned: an `Account` entity you control and your own HS256 JWTs signed server-side.

```bash
git clone https://github.com/despia-native/base44-native-oauth
cd base44-native-oauth
npm install
npm install -g base44@latest
```

There are three places to edit, nothing else is hardcoded. Set your Despia deep-link scheme in `src/config/app-config.js`. Add your secrets in the Base44 dashboard under Settings, Environment Variables, never in the repo: `JWT_SECRET` (long, random, and stable, since changing it signs everyone out), `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `RESEND_API_KEY` and `RESEND_FROM` for password-reset email, and `APP_BASE_URL`. Push and Sign in with Apple add their own keys (`ONESIGNAL_APP_ID`, `ONESIGNAL_REST_API_KEY`, and `APPLE_SERVICES_ID`) when you turn those features on. Then register the Google redirect URI (`APP_BASE_URL` plus `/native-callback.html`) and your scheme with the `oauth/auth` path in Despia. Run `base44 dev` to work locally and publish from the dashboard.

On security, the template does the right things. Google runs as real native OAuth: the system browser handles consent, a single-use code returns over the deep link, and the backend exchanges it server-side with the `id_token` audience checked against your client id to block token substitution. The `JWT_SECRET` never leaves the backend, passwords are PBKDF2-hashed with per-user salts, every entity is deny-all RLS, and every function verifies the JWT and re-checks the role against the live `Account` record before touching data. The session persists in the Despia Storage Vault, so it survives reinstall. Since you own the stack, the one value you must guard is `JWT_SECRET`: keep it strong, stable, and in Base44 secrets. One gap to close before you open signups to the public: the auth endpoints (login, register, password reset) ship without app-level rate limiting, so add per-IP or per-email throttling before you scale.

One production rule the template bakes in, and one to keep in your own code: never let an awaited `despia()` call block the UI. The bridge has no guaranteed callback, so if the native side does not answer (feature off, older build, web preview), an awaited call can hang for 15 to 30 seconds, which reads as a frozen app and is a rejection risk. The template caps every awaited bridge read at two seconds and lets the work finish in the background, so a dead bridge degrades to a fallback instead of a freeze. Fire-and-forget calls like haptics and paywall launches need none of this.

## Get it on the stores

Take the Base44 app you already built and ship it to iOS and Android without a Mac or Xcode. Code signing and submission run from the browser, and web updates reach users over the air with no store review.

[See the setup docs at setup.despia.com](https://setup.despia.com)