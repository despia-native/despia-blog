---
title: "Lovable Apple Login (Native Sign-In)"
seoTitle: "Lovable Apple Login (Native Sign-In)"
seoDescription: "Lovable Apple login done natively: the native Apple sign-in sheet on iOS, the OAuth bridge on Android, and Lovable Cloud holding the session."
datePublished: 2026-07-21T04:39:08.031Z
cuid: cmru5yyuj00010aj793380zqt
slug: lovable-apple-login-native-sign-in
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/715aa694-acf7-4c0d-8cf6-c11d571a04c3.png
tags: apple, reactjs, webview, auth, webapp, lovable, vibe-coding

---

If your Lovable app ships to iOS with Google login, Guideline 4.8 obliges you to offer an equivalent privacy-focused login next to it, one that limits data collection and supports private email, and Sign in with Apple is usually the simplest compliant choice, unless one of Guideline 4.8's exceptions applies. Apple's exceptions include proprietary account systems, enterprise and education accounts, and government identity systems. Here is the pleasant surprise: Apple is easier than Google was. Apple's JS SDK with `usePopup: true` can present the native Apple sign-in sheet from the WebView in Despia, so iPhone needs no browser, no bridge, and no callback file. Android has no equivalent, so it uses the same Despia `oauth://` bridge pattern: Chrome Custom Tabs, a static `native-callback.html`, and a deeplink home. Lovable Cloud is Supabase under the hood, so everything here is plain `supabase-js`. This guide targets the current Lovable stack, TanStack Start. If yours is an older Vite and React Router project, the flow is identical but the file paths and the backend differ, jump to the Legacy Lovable section near the end for that wiring.

## Three platforms, one token

There are two ways in but one credential. On iOS the SDK opens the native sheet in place; on the web the identical call opens a browser popup; both hand you an Apple `id_token` in the JS callback. On Android the SDK cannot open a sheet, so Chrome Custom Tabs runs the Apple screen and returns the `id_token` through the deeplink. Whichever platform, the app ends with the same Apple `id_token` and calls `supabase.auth.signInWithIdToken({ provider: 'apple' })`. One code path, no second flow to reason about.

## Apple Developer Console setup

1.  Under Certificates, Identifiers and Profiles, open your App ID and enable Sign In with Apple.
    
2.  Create a Services ID, for example `com.yourcompany.yourapp.webauth`. That value is the `clientId` in the code below. The Services ID drives web and JS Sign in with Apple, while the App ID capability from step 1 enables the native app side.
    
3.  Configure the Services ID: enable Sign In with Apple, set the Primary App ID, add your app's domain, and register the Return URLs, comma-delimited, `https://` required, no wildcards, exact string match with the trailing slash included. Which URLs depends on your Android variant, and mixing them up is the number one rejection cause. Apple's registered Return URL is where Apple itself returns; your `redirect_to` is where Supabase forwards afterwards, allowlisted in Lovable Cloud, Auth, URL Configuration, not with Apple; and the deeplink is how the result re-enters the app, Apple never sees it.
    
    *   Always register `https://yourapp.com/` for the JS SDK popup on iOS and web.
        
    *   Direct Apple variant: also register your callback route, `https://yourapp.com/api/public/apple-callback`, since that is the `redirect_uri` your server route sends. On the legacy stack it is the edge function URL instead.
        
    *   Supabase-brokered variant: also register Supabase's callback, `https://YOUR-PROJECT.supabase.co/auth/v1/callback`, because Apple returns to Supabase, not to you. Your `native-callback.html` is never registered with Apple in this variant, it only goes in the redirect allowlist as the `redirect_to` target.
        
4.  No `.p8` key. This flow verifies Apple's `id_token` directly, so login needs no Sign in with Apple private key. (A `.p8` is only needed for the server-side OAuth redirect flow or token revocation, not for the `id_token` method used here.)
    
5.  In Lovable Cloud, open Users, Auth Providers, Apple, and add your Services ID to the authorized client IDs, so `signInWithIdToken` accepts tokens issued for it. Because this is the `id_token` flow, that is all it needs, no `.p8` and no client secret to generate or rotate.
    

## Load the SDK and detect the platform

Add Apple's script before your app script, in `index.html`. It requires HTTPS, so it will not initialize on plain localhost. The HTTPS rule has one Despia-specific consequence: the Local Server offline mode serves your app over HTTP, so Apple sign-in cannot initialize there. If your app uses it, open the Despia Editor and switch Offline Support from Native to PWA, then add a service worker that caches your assets. That keeps offline support through the service worker while giving Apple's SDK the secure HTTPS origin it requires.

The snippets below show the complete authentication flow, but intentionally leave application-specific database, session, routing, and error-handling functions for you to connect to your existing stack.

```html
<script src="https://appleid.cdn-apple.com/appleauth/static/jsapi/appleid/1/en_US/appleid.auth.js"></script>
```

Install the SDK with `bun add despia-native`, then derive the three platform constants once:

```javascript
const ua = navigator.userAgent.toLowerCase()
const isDespia = ua.includes('despia')
const isDespiaIOS = isDespia && (ua.includes('iphone') || ua.includes('ipad'))
const isDespiaAndroid = isDespia && ua.includes('android')
const isWeb = !isDespia
```

## One sign-in handler for all three platforms

iOS and web get the `id_token` straight from the SDK popup and hand it to Supabase. Android fetches an Apple OAuth URL from a server route and opens the bridge; the `id_token` comes back later on `/auth`. `usePopup: true` is required in the Despia WebView: redirect mode blanks the WebView there and gets App Store rejections. On a plain web browser redirect mode works, but keeping `usePopup: true` for both is simplest.

```javascript
import despia from 'despia-native'
import { supabase } from '@/integrations/supabase/client'

async function signInWithApple() {
  // iOS + Web: Apple's JS SDK popup (native sheet on iOS, browser popup on web)
  if (isDespiaIOS || isWeb) {
    const nonce = crypto.randomUUID() // bind this sign-in to the token Supabase receives
    window.AppleID.auth.init({
      clientId: 'com.yourcompany.yourapp.webauth', // your Services ID
      scope: 'name email',
      redirectURI: window.location.origin + '/',   // must match the registered return URL exactly
      usePopup: true,                              // required, redirect mode blanks the WebView
      nonce,
    })

    const response = await window.AppleID.auth.signIn()
    const { error } = await supabase.auth.signInWithIdToken({
      provider: 'apple',
      token: response.authorization.id_token,
      nonce, // same nonce Supabase verifies against the token
    })
    if (error) throw error
    return
  }

  // Android: no native sheet exists, run the Apple screen in Chrome Custom Tabs
  if (isDespiaAndroid) {
    const res = await fetch('/api/public/apple-url', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ deeplink_scheme: 'myapp' }), // Despia > Publish > Deeplink
    })
    const { url } = await res.json()
    despia(`oauth://?url=${encodeURIComponent(url)}`)
    // sign-in resumes on /auth when the deeplink lands with the id_token
  }
}
```

Apple only returns the user's name and email on the very first sign-in, so if you store a display name, read `response.user` from that first response and persist it then, later logins omit it. In your click handler, catch and swallow `user_cancelled_authorize`, the cancellation error Apple documents, and you may also handle `popup_closed_by_user` defensively for older integrations. Either means the person dismissed the sheet. One historical wrinkle: iOS 17.1 briefly rendered an HTML form instead of the native sheet inside WKWebView, fixed in 17.2, and sign-in kept working either way.

## The Android URL: a TanStack server route

Lovable's current default is TanStack Start, so app-internal logic can live in a server route. (A Supabase Edge Function works too, since Lovable Cloud is Supabase underneath, and that is the shape the Legacy section uses.) This route builds the Apple authorize URL directly, so Android ends on the same `signInWithIdToken` exchange as iOS, one code path across all three platforms. The deeplink scheme rides in `state` so the `POST` callback handler can read it back, and `response_mode: 'form_post'` delivers the result to `/api/public/apple-callback`. Set `APP_BASE_URL` to your stable published host, `project--<id>.lovable.app`, not a preview URL, and register that exact `/api/public/apple-callback` URL with Apple, since Apple does exact-match on the redirect and a preview host breaks it.

One note on the path. TanStack server routes are not gated by the framework itself, but some Lovable templates add global auth middleware that fronts `/api/*`, and Android calls this route before the user has a session. If your project applies such a guard, a gated path would 401 in production, so a clearly public prefix is the safe default. If it does not, the exact path does not matter, just keep it consistent.

```ts
// src/routes/api/public/apple-url.ts  (public path, in case your template gates /api/*)
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/api/public/apple-url')({
  server: {
    handlers: {
      POST: async ({ request }) => {
        const { deeplink_scheme } = await request.json()

        const url = 'https://appleid.apple.com/auth/authorize?' + new URLSearchParams({
          client_id: 'com.yourcompany.yourapp.webauth',              // your Services ID
          redirect_uri: `${process.env.APP_BASE_URL}/api/public/apple-callback`, // the POST handler below, registered with Apple
          response_type: 'code id_token',
          response_mode: 'form_post',                                // required with name/email scopes
          scope: 'name email',
          state: deeplink_scheme,                                    // the callback reads this back
        })

        return Response.json({ url })
      },
    },
  },
})
```

Two Lovable specifics. Read `process.env` inside the handler rather than at module scope; if Lovable deploys your server route to a Workers-style runtime, env is bound per request and a module-scope read can be undefined at import time, so reading inside the handler is reliable either way. And this is a TanStack server route with `process.env`, not a Deno edge function; if you would rather keep server logic in a Supabase Edge Function, that works too, and the Legacy section shows that shape.

One Apple rule shapes this route. Apple hard-requires `response_mode: 'form_post'` the moment you request the `name` or `email` scope, and a static callback file cannot read a POST body. So the Return URL is a second server-route handler, `/api/public/apple-callback`, whose `POST` reads Apple's body, `id_token`, `state` carrying the scheme, and a `user` JSON field on the very first authorization only, that is where the name comes from on Android, and answers with a tiny HTML bridge page firing `myapp://oauth/auth?id_token=...&full_name=...`. It is intentionally public, Apple calls it, and it only relays, `signInWithIdToken` still verifies the token. Two footnotes: skipping the name and email scopes makes `fragment` legal again and a static callback file suffices, and letting Supabase broker the Android leg sidesteps all of this, Apple then form\_posts to Supabase's server and your static callback receives the token pair in the hash like the Google flow.

## The callback layer, three files that work together

One context rule keeps this layer honest: the Chrome Custom Tab and the WebView are separate browsers with separate `sessionStorage`. Everything that crosses between them travels in the URL, the bridge page fires the deeplink from the tab, and the boot capture and `/auth` stash run in the WebView after the deeplink lands. Never try to read the WebView's `sessionStorage` from the callback or bridge page, it is a different browser and the value is not there.

The Android leg does not hand the token back through one file. It is three cooperating pieces, and missing any one makes the token silently vanish. iOS and web never touch this layer, they exchange the `id_token` inline from the popup. This is the Android return path only.

### 1\. public/native-callback.html, the return page

Apple redirects the in-app browser here with the `id_token` in the hash and the scheme in `state`. This static page forwards the `id_token` through the `oauth/` deeplink and offers a tappable fallback for schemes that need a user gesture. It must be a real static file at `public/native-callback.html`, not a TanStack route.

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
    // Apple fragment mode returns the id_token in the hash; the scheme rides in state.
    var q = new URLSearchParams(location.search);
    var hash = new URLSearchParams(location.hash.slice(1));

    var scheme = q.get('deeplink_scheme') || hash.get('state') || q.get('state') || '';
    var idToken = hash.get('id_token') || '';
    var error = hash.get('error') || q.get('error') || '';

    var query = idToken
      ? 'id_token=' + encodeURIComponent(idToken)
      : 'error=' + encodeURIComponent(error || 'no_token');

    if (scheme && /^[a-z][a-z0-9+.-]*$/i.test(scheme)) {
      // Native: hand the id_token to the app via its deep link (Despia path oauth/auth).
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

### 2\. Boot-time capture, before the router mounts

When Despia reopens the app, the deeplink lands as a history URL change on any path, often `/`, and the WebView does not reload. TanStack Start has no hand-written entry file to hook, so run this as an inline script in the `<head>` of `index.html`, before the router mounts. It stashes the `id_token` and normalizes the URL to `/auth` so the route guard cannot eat it. Prefer `index.html` over `shellComponent` in `__root.tsx`: the shell runs during SSR where `sessionStorage` and `location.hash` do not exist, so if you must use it, gate the whole thing behind `typeof window !== 'undefined'`.

```html
<!-- inline in index.html <head>, runs before the router mounts -->
<script>
  (function () {
    var h = new URLSearchParams(location.hash.slice(1));
    var p = new URLSearchParams(location.search);
    var hasToken = h.get('id_token') || p.get('id_token') || h.get('error') || p.get('error');
    if (hasToken) {
      sessionStorage.setItem('pending_oauth', location.search + location.hash);
      window.history.replaceState(null, '', '/auth');
    }
  })();
</script>
```

### 3\. /auth must be a public route

This is the Lovable-specific footgun. In the TanStack template the protected gate lives at `src/routes/_authenticated/route.tsx` and redirects unauthenticated users to `/auth`. So `/auth` has to be a top-level public route at `src/routes/auth.tsx`, not under `_authenticated/`. Put it under the gate and the token-carrying URL bounces before `signInWithIdToken` ever runs.

The route reads whatever was stashed at boot, or the live URL if nothing was stashed, since the token can arrive as a later history change while `/auth` is already mounted. It watches `popstate` and `hashchange`, then exchanges the `id_token`, the same call iOS and web make.

```tsx
// src/routes/auth.tsx  (public, NOT under _authenticated/)
import { createFileRoute, useNavigate } from '@tanstack/react-router'
import { useEffect } from 'react'
import { supabase } from '@/integrations/supabase/client'

export const Route = createFileRoute('/auth')({ component: Auth })

function Auth() {
  const navigate = useNavigate()

  useEffect(() => {
    let done = false

    const complete = async () => {
      if (done) return
      const stash = sessionStorage.getItem('pending_oauth')
      const src = stash || (location.search + location.hash)
      const h = new URLSearchParams((src.split('#')[1] || ''))
      const p = new URLSearchParams(src.split('#')[0].replace(/^\?/, ''))
      const idToken = h.get('id_token') || p.get('id_token')
      const error = h.get('error') || p.get('error')
      if (error) { done = true; sessionStorage.removeItem('pending_oauth'); return navigate({ to: '/login' }) }
      if (!idToken) return // keep watching, it may still be arriving

      done = true
      sessionStorage.removeItem('pending_oauth')
      const { error: sessErr } = await supabase.auth.signInWithIdToken({ provider: 'apple', token: idToken })
      navigate({ to: sessErr ? '/login' : '/' })
    }

    complete()
    window.addEventListener('popstate', complete)
    window.addEventListener('hashchange', complete) // hash-only change, popstate does not fire
    return () => {
      window.removeEventListener('popstate', complete)
      window.removeEventListener('hashchange', complete)
    }
  }, [navigate])

  return <p>Signing you in...</p>
}
```

## Legacy Lovable (Vite + React Router)

Older Lovable projects run a different stack: Vite, React, and React Router DOM, with the Supabase client at `src/integrations/supabase/client.ts`, server logic in Supabase Edge Functions under `supabase/functions/`, and static assets in `public/`. No TanStack, no server routes, no file-based routing. The Apple flow is the same shape, but the wiring differs. Here is the delta.

The Apple provider is still configured in Lovable Cloud, Auth, Providers, Apple, never by logging into Supabase directly, and the callback HTML still lives at `public/native-callback.html`, served verbatim by Vite. What changes:

The Android URL builder is a real Supabase Edge Function on this stack, at `supabase/functions/apple-auth-url/index.ts`. It is Deno, so it imports from `deno.land/std` and reads `Deno.env.get(...)`, and its secrets go in Lovable Cloud, Edge Functions, Secrets, not `supabase secrets set`. Set `APP_BASE_URL` to your stable published host and register that exact `native-callback.html` URL with Apple, since Apple exact-matches the redirect.

```ts
// supabase/functions/apple-auth-url/index.ts
import { serve } from 'https://deno.land/std/http/server.ts'

serve(async (req) => {
  const { deeplink_scheme } = await req.json()

  const url = 'https://appleid.apple.com/auth/authorize?' + new URLSearchParams({
    client_id: Deno.env.get('APPLE_CLIENT_ID')!,        // your Services ID
    redirect_uri: `${Deno.env.get('APP_BASE_URL')}/functions/apple-callback`, // a second edge function, the form_post target
    response_type: 'code id_token',
    response_mode: 'form_post', // required with name/email scopes
    scope: 'name email',
    state: deeplink_scheme,
  })

  return Response.json({ url })
})
```

Calling it through `supabase.functions.invoke` handles CORS for you, so the function above stays lean. If you ever hit it cross-origin by hand instead, add `Access-Control-Allow-Origin` and answer the `OPTIONS` preflight. The client calls it with `supabase.functions.invoke`, not `fetch('/api/...')`, since that path does not exist here. The Apple handler lives in `src/lib/apple-auth.ts`, imports the client from `@/integrations/supabase/client` (there is no `lovable.auth`), and keeps the one-token flow: iOS and web call `signInWithIdToken`, Android invokes the function and opens the bridge.

```ts
// src/lib/apple-auth.ts
import despia from 'despia-native'
import { supabase } from '@/integrations/supabase/client'

const ua = navigator.userAgent.toLowerCase()
const isDespiaAndroid = ua.includes('despia') && ua.includes('android')

export async function signInWithApple() {
  if (isDespiaAndroid) {
    const { data } = await supabase.functions.invoke('apple-auth-url', { body: { deeplink_scheme: 'myapp' } })
    despia(`oauth://?url=${encodeURIComponent(data.url)}`)
    return
  }

  window.AppleID.auth.init({
    clientId: 'com.yourcompany.yourapp.webauth',
    scope: 'name email',
    redirectURI: window.location.origin + '/',
    usePopup: true,
  })
  const response = await window.AppleID.auth.signIn()
  const { error } = await supabase.auth.signInWithIdToken({ provider: 'apple', token: response.authorization.id_token })
  if (error) throw error
}
```

The rest maps cleanly: the SDK script tag goes in `index.html` at the project root (no `__root.tsx`), the `/auth` return page is a normal React Router route at `src/pages/AuthCallback.tsx`, and only frontend `import.meta.env.VITE_*` values are exposed to the client, so the Apple Team ID, Key ID, and private key stay in the Edge Function's secrets and never touch frontend code. Do not reach for `@supabase/ssr` on this stack; the plain `@supabase/supabase-js` client already wired in `src/integrations/supabase/client.ts` is all you need.

One bug to avoid in the callback file: decode the `state` before you trust it. Apple returns it URL-encoded, so `decodeURIComponent` it first, then read the scheme, rather than splitting the raw string.

## The things that trip people up

`/auth` must be a public route at `src/routes/auth.tsx`, not under `_authenticated/`, or the gate redirects away the token before sign-in runs. `usePopup: true` is required in the Despia WebView, where the redirect variant shows a blank white screen and fails review (plain web browsers handle redirect mode fine). The SDK script tag must load before your app script, and only over HTTPS, which rules out the HTTP Local Server offline mode, switch to PWA offline support instead. The `redirectURI` in `AppleID.auth.init` must match a registered return URL character for character, trailing slash included. App-internal URL building goes in a TanStack server route with `process.env` on the current stack, or a Supabase Edge Function if you prefer, and either must be reachable before sign-in. And the Android deeplink still lives or dies by the `oauth/` prefix and a static `native-callback.html` in `public/`.

## Ship both logins together

Apple next to Google is exactly the pairing iOS review expects, and on this setup they share one bridge pattern, one callback file, and one `/auth` route. Take your Lovable app to iOS and Android with both sign-ins built for review. Code signing and submission run from the browser, no Mac and no CLI.

[See the setup docs at setup.despia.com](https://setup.despia.com/native-features/oauth/apple)