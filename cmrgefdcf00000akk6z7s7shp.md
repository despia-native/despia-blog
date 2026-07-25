---
title: "Lovable Google Login (Native Sign-In)"
seoTitle: "Lovable Google Login (Native Sign-In)"
seoDescription: "Lovable Google login done natively: open a secure browser with the Despia OAuth bridge, handle the callback, and set the session on iOS and Android."
datePublished: 2026-07-11T13:27:03.777Z
cuid: cmrgefdcf00000akk6z7s7shp
slug: lovable-google-login-native-sign-in
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/16352416-b7b8-4e6a-bd66-e317d2ea7ba6.png
tags: webview, google-login, lovable

---

Lovable builds the app, but a Google sign-in button that works on the web breaks inside a mobile app. Google will not run its OAuth screen in an embedded WebView, and the App Store expects a real browser session. Despia gives you the native `oauth://` bridge, which opens the secure system browser, ASWebAuthenticationSession on iOS and Chrome Custom Tabs on Android, and returns the tokens to your Lovable app. One flow covers both platforms, with a normal redirect on web. Lovable Cloud is Supabase under the hood, so everything here is plain `supabase-js`. This guide targets the current Lovable stack, TanStack Start. If yours is an older Vite and React Router project, jump to the Legacy Lovable section near the end for that wiring.

## Why the web Google button fails in a mobile app

Google blocks its consent screen inside an embedded WebView, the `disallowed_useragent` error, because an embedded view could read the user's password, and App Review expects third-party login flows to run in a secure browser or native authentication context. So the plain web button either errors or gets rejected. The fix is to leave the WebView for the sign-in. Despia's `oauth://` bridge opens the system browser, the user authenticates, and Despia closes it and returns the tokens. Unlike Apple Sign In, Google uses the same bridge on iOS and Android, so your sign-in code has no platform split.

## How the flow works

1.  The user taps Sign in with Google.
    
2.  In Despia, your backend builds a Google OAuth URL that redirects to a small file, `native-callback.html`, and returns it.
    
3.  `despia('oauth://?url=...')` opens that URL in the secure browser.
    
4.  The user signs in. Tokens arrive at `native-callback.html`.
    
5.  That file fires `myapp://oauth/auth?access_token=...`. The `oauth/` prefix tells Despia to close the browser and move the WebView to `/auth`.
    
6.  Your `/auth` page reads the token and creates the session.
    

On web the button runs your provider's normal redirect and lands on the same `/auth`.

## Wire the sign-in button

Before any code, register your deeplink scheme in the Despia dashboard under Publish > Deeplink, that is the value `myapp` stands for below. Then install the SDK with `bun add despia-native`. The button is a plain handler:

```javascript
import despia from 'despia-native'
import { supabase } from '@/integrations/supabase/client'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

async function signInWithGoogle() {
  if (isDespia) {
    // get the OAuth URL from your server route and open it in the secure browser
    const { url } = await fetch('/api/public/google-url', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ deeplink_scheme: 'myapp' }), // Despia > Publish > Deeplink
    }).then(r => r.json())
    despia(`oauth://?url=${encodeURIComponent(url)}`)
  } else {
    // web: normal redirect, supabase-js installs the session itself on return
    await supabase.auth.signInWithOAuth({ provider: 'google', options: { redirectTo: location.origin + '/auth' } })
  }
}
```

Note the `/api/public/` prefix. TanStack server routes are not gated by the framework itself, but some Lovable templates add global auth middleware that fronts `/api/*`, and this route is called before the user has a session. If your project applies such a guard, a gated path would 401 in production, so a clearly public prefix is the safe default. If your project has no such middleware, the exact path does not matter, keep it consistent with your callback and Google config.

## Generate the OAuth URL in a TanStack server route

On the current stack, app-internal logic is a TanStack server route. (A Supabase Edge Function works too, since Lovable Cloud is Supabase underneath, and that is what the Legacy section uses.) It hits Supabase's authorize endpoint with `native-callback.html` as the redirect, and Supabase returns its own access and refresh token pair in the hash.

```ts
// src/routes/api/public/google-url.ts  (public prefix, in case your template gates /api/*)
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/api/public/google-url')({
  server: {
    handlers: {
      POST: async ({ request }) => {
        const { deeplink_scheme } = await request.json()

        // read env inside the handler; some runtimes only expose it per request
        const base = process.env.SUPABASE_URL // server var, not VITE_SUPABASE_URL
        const redirectUrl = `${process.env.APP_BASE_URL}/native-callback.html?deeplink_scheme=${encodeURIComponent(deeplink_scheme)}`

        const url = `${base}/auth/v1/authorize?` + new URLSearchParams({
          provider: 'google',
          redirect_to: redirectUrl,
        })

        return Response.json({ url })
      },
    },
  },
})
```

Two Lovable specifics. Read `process.env` inside the handler rather than at module scope. If Lovable deploys your server route to a Workers-style runtime, env is bound per request and a module-scope read can be undefined at import time, so reading inside the handler is the reliable choice either way. And the server-side Supabase URL is `SUPABASE_URL`, the frontend `VITE_SUPABASE_URL` is a separate client-exposed value. Supabase stays on the implicit flow whenever you omit a `code_challenge`, so there is no flag to set for it.

Set `APP_BASE_URL` to your stable published host, `project--<id>.lovable.app` or your custom domain, not a preview URL, and add that `redirect_to` target to the allowed list under Lovable Cloud, Auth, URL Configuration. Supabase already whitelists its own `/auth/v1/callback`, but the `redirect_to` you send must be an allowed URL or Supabase drops it.

Why implicit here: Lovable's Supabase client session needs the tokens back in the browser context that calls `setSession()`. The OAuth screen runs in a separate secure-browser context, ASWebAuthenticationSession or Chrome Custom Tabs, whose cookies and storage cannot cross into the WebView by design, so passing the tokens through the deeplink is the bridge between the two worlds. Because the hash carries Supabase's own access and refresh token pair, not a raw Google grant, `setSession()` yields a full session that supabase-js keeps refreshing, so users are not signed out when the access token expires. If you later want a code exchange, Supabase's `exchangeCodeForSession()` covers it and the Despia side stays identical.

## Add public/native-callback.html

This small page runs inside the secure browser. It reads the tokens from the URL hash and fires the deeplink that closes the browser and passes them to your app. Keep it as a plain HTML file in `public/`, not a React route, because a router can strip the hash before you read it.

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

One context rule keeps this layer honest: the Chrome Custom Tab and the WebView are separate browsers with separate `sessionStorage`. Everything that crosses between them travels in the URL, the bridge page fires the deeplink from the tab, and the boot capture and `/auth` stash run in the WebView after the deeplink lands. Never try to read the WebView's `sessionStorage` from the callback or bridge page, it is a different browser and the value is not there.

## Capture the token before the router mounts

When Despia reopens the app, the deeplink can land on any path, often `/`, as a history URL change with no reload. If the router mounts a protected gate before `/auth`, the token-carrying URL bounces away first. So stash it before the router mounts, with an inline script in the `<head>` of `index.html`, and normalize the URL to `/auth`.

```html
<!-- inline in index.html <head>, runs before the router mounts -->
<script>
  (function () {
    var h = new URLSearchParams(location.hash.slice(1));
    var p = new URLSearchParams(location.search);
    var hasToken = h.get('access_token') || p.get('access_token') || h.get('error') || p.get('error');
    if (hasToken) {
      sessionStorage.setItem('pending_oauth', location.search + location.hash);
      window.history.replaceState(null, '', '/auth');
    }
  })();
</script>
```

Prefer `index.html` over a shell component in `__root.tsx`: the shell runs during SSR, where `sessionStorage` and `location.hash` do not exist, so gate it behind `typeof window !== 'undefined'` if you ever put it there.

## Read the token on /auth, a public route

Keep `/auth` as a top-level public route at `src/routes/auth.tsx`, not under `_authenticated/`. If your template uses the common `_authenticated` layout guard, anything beneath it redirects unauthenticated visitors away, so a `/auth` placed under the gate would bounce the token-carrying URL before `setSession` runs. Outside the gate it is safe regardless. It reads whatever the boot script stashed, or the live URL if nothing was stashed, and watches `hashchange` because the token can arrive as a later hash change while the page is already mounted.

```tsx
// src/routes/auth.tsx  (public, NOT under _authenticated/)
import { createFileRoute, useNavigate } from '@tanstack/react-router'
import { useEffect } from 'react'
import { supabase } from '@/integrations/supabase/client'

export const Route = createFileRoute('/auth')({ component: Auth })

function Auth() {
  const navigate = useNavigate()

  useEffect(() => {
    const check = async () => {
      const stash = sessionStorage.getItem('pending_oauth')
      const src = stash || (location.search + location.hash)
      const q = new URLSearchParams(src.split('#')[0].replace(/^\?/, ''))
      const hash = new URLSearchParams(src.split('#')[1] || '')
      const accessToken = q.get('access_token') || hash.get('access_token')
      const refreshToken = q.get('refresh_token') || hash.get('refresh_token') || ''
      if (!accessToken) return
      sessionStorage.removeItem('pending_oauth')

      const { error } = await supabase.auth.setSession({ access_token: accessToken, refresh_token: refreshToken })
      navigate({ to: error ? '/login' : '/' })
    }
    check()
    // the token can arrive as a later hash change while /auth is already mounted
    window.addEventListener('hashchange', check)
    return () => window.removeEventListener('hashchange', check)
  }, [navigate])

  return <p>Signing you in...</p>
}
```

## Set up Google in the Cloud Console

1.  In console.cloud.google.com, open APIs and Services, then Credentials, and create an OAuth 2.0 Client ID of type Web application.
    
2.  Add your app domain under Authorized JavaScript origins.
    
3.  Under Authorized redirect URIs, add Supabase's callback, `https://YOUR-PROJECT.supabase.co/auth/v1/callback`, not your app URL.
    
4.  In Lovable Cloud, Auth, Providers, Google, paste the Client ID and Client Secret, and add your `native-callback.html` host under Auth, URL Configuration so Supabase accepts it as a `redirect_to`.
    
5.  Set your deeplink scheme from Despia > Publish > Deeplink and replace `myapp` in the code.
    

## Legacy Lovable (Vite + React Router)

Older Lovable projects run Vite, React, and React Router DOM, with the Supabase client at `src/integrations/supabase/client.ts` and server logic in Supabase Edge Functions under `supabase/functions/`. The flow is identical, the wiring differs.

The URL builder is a real Supabase Edge Function at `supabase/functions/google-url/index.ts`, Deno, reading `Deno.env.get('SUPABASE_URL')`, with secrets set in Lovable Cloud, Edge Functions, Secrets. Calling it through `supabase.functions.invoke` handles CORS for you; if you ever hit it cross-origin by hand, add `Access-Control-Allow-Origin` and answer the `OPTIONS` preflight.

```ts
// supabase/functions/google-url/index.ts
import { serve } from 'https://deno.land/std/http/server.ts'

serve(async (req) => {
  const { deeplink_scheme } = await req.json()
  const redirectUrl = `${Deno.env.get('APP_BASE_URL')}/native-callback.html?deeplink_scheme=${encodeURIComponent(deeplink_scheme)}`

  const url = `${Deno.env.get('SUPABASE_URL')}/auth/v1/authorize?` + new URLSearchParams({
    provider: 'google',
    redirect_to: redirectUrl,
  })

  return Response.json({ url })
})
```

The client invokes it, it does not fetch a path:

```javascript
import despia from 'despia-native'
import { supabase } from '@/integrations/supabase/client'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

async function signInWithGoogle() {
  if (isDespia) {
    const { data } = await supabase.functions.invoke('google-url', { body: { deeplink_scheme: 'myapp' } })
    despia(`oauth://?url=${encodeURIComponent(data.url)}`)
  } else {
    await supabase.auth.signInWithOAuth({ provider: 'google', options: { redirectTo: location.origin + '/auth' } })
  }
}
```

`native-callback.html` is unchanged and served verbatim from `public/`. The `/auth` return page is a normal React Router route at `src/pages/AuthCallback.tsx`, using `useSearchParams` and `useNavigate` from `react-router-dom` in place of the TanStack imports, and it still reads the boot stash plus the live query and hash. Add the same boot-capture script to `index.html`. `APP_BASE_URL` must be your stable host and registered as an allowed redirect in Cloud, Auth, URL Configuration, same as the modern stack.

## The things that trip people up

Keep `/auth` a public route, not under an `_authenticated/` guard if your template has one, so it does not redirect the token away before `setSession` runs. Put the URL builder on a public path in case your template gates `/api/*`, since Android calls it before it has a session. Read server env inside the handler, and use the server `SUPABASE_URL`, not the client `VITE_SUPABASE_URL`. `APP_BASE_URL` must be your stable published host and an allowed `redirect_to` in Cloud, Auth, URL Configuration, or Supabase drops it. The `oauth/` prefix must be in the deeplink, `native-callback.html` must be a static file in `public/`, and `/auth` must read both the query and the hash and re-run on a URL change, which is why it watches `hashchange` and consumes the boot stash.

## Ship it with native sign-in built in

Take your Lovable app to iOS and Android with Google sign-in built for review. Code signing and submission run from the browser, no Mac and no CLI.

[See the setup docs at setup.despia.com](https://setup.despia.com/native-features/oauth/google)