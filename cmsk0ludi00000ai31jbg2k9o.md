---
title: "Lovable Mobile App Slow? Turn Off SSR in TanStack Start"
seoTitle: "Lovable Mobile App Slow? Turn Off SSR in TanStack Start"
seoDescription: "A slow Lovable mobile app that reloads on every page change is not a WebView problem. It is SSR in TanStack Start, and you can turn it off."
datePublished: 2026-08-08T06:50:58.186Z
cuid: cmsk0ludi00000ai31jbg2k9o
slug: lovable-mobile-app-slow-turn-off-ssr-in-tanstack-start
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/4b2c271a-84d4-4d98-9794-18fce7dadd17.png
tags: reactjs, webapp, pwa, tanstack, lovable

---

Every tap flashes white. The screen you were on tears down, the spinner comes back, the data you already had is fetched again. On a laptop you would barely register it. On a phone, inside your own app, it reads as a website in a frame, and the conclusion most people reach is that the WebView is the problem and the fix is to rebuild the whole thing natively.

It usually is not. Look at what your Lovable project is rendering before you rewrite it.

**Short answer:** new Lovable projects ship on TanStack Start with server-side rendering turned on. SSR puts your server in the middle of page transitions. In a browser tab on wifi that costs almost nothing. In a WebView on cellular it is the whole difference between an app and a website. Turn SSR off for the routes that make up the app, keep it on the routes that need to rank, and navigation gets fast without changing your stack.

## What changed in Lovable

Lovable used to generate React plus Vite single page apps, deployed as static files and rendered entirely on the client. New projects moved to TanStack Start with SSR on by default on May 13, 2026, and new Enterprise projects on June 22, 2026. Older projects stayed on the Vite stack unless you took the upgrade.

That means there are two populations of Lovable app, and only one of them has this problem. If your project predates May 2026 and you never upgraded, your slowness is something else: bundle size, unindexed Supabase queries, or fetch waterfalls. If you started after that date, or you took the SSR upgrade because someone told you it would fix your Google ranking, keep reading.

The reason Lovable made the change is sound. A client-rendered app hands crawlers a mostly empty page, and SSR hands everyone the finished HTML: visitors, search engines, AI crawlers, link preview bots. That is a real win for a public website.

Your mobile app has no crawlers in it.

## Why SSR is the wrong trade inside a WebView

On the web, SSR trades a round trip for a faster first paint and readable HTML. The round trip is short because your user is on a laptop on wifi, and they land on one page from Google and mostly stay there.

A mobile app inverts every one of those assumptions. Your user is on cellular with 80 to 200ms of latency before anything moves. They do not land on one page, they tap through twenty screens in a session. And there is no crawler to serve, because the App Store listing is what gets indexed, not your routes.

So you are paying the round trip twenty times to buy something the app cannot use.

## The two things producing the full reload

**Navigation that is not router navigation.** Generated code reaches for `<a href="/dashboard">` and `window.location.href = '/settings'` more often than you would expect, and both of those are full document loads. The WebView throws away the page, asks your server for HTML, waits for the server render, downloads and re-evaluates the bundle, hydrates, and re-runs every fetch on the new route. Your in-memory state is gone. So is anything you had registered on `window`, which matters more in a native runtime than in a browser tab, because your native callbacks get torn down and re-registered on every screen change.

Under the old Vite stack this was already wasteful. Under SSR you added a server render in front of it.

**Redirects in route guards.** Auth-gated apps redirect on entry. When that redirect is thrown during a server render, it comes back as an HTTP redirect, and an HTTP redirect is another document load. Sign-in flows and protected sections are exactly the places people report the worst lag, and this is why.

Even when navigation is correct and the router handles it, SSR still costs you. Route loaders run on the server, so a screen change waits for your server, which is waiting for Supabase. Two hops instead of one, on every tap.

## Check this before you change anything

Open the deployed app in a desktop browser, open the Network panel, tick preserve log, and tap through five screens.

If the log clears itself or you see a new `document` request per navigation, those are real page loads and you have the first problem. If you only see fetch and XHR entries and the screens are still slow, routing is fine and you are looking at loader latency, which is the second problem. Both are fixed by the same change, but knowing which one you have tells you what to expect.

Test the deployed URL, not the Lovable preview. The preview runs under different conditions and will lie to you about this specific thing.

## Turn SSR off for the app routes

You do not have to write any of this. Open the Lovable chat and paste one of the two prompts below. Which one depends on whether your Lovable project contains a public marketing site as well as the app.

**If the Lovable project is only the app** (a login screen and everything behind it, with your marketing site somewhere else or not existing yet), paste this:

```text
My project is a mobile app. It does not need SEO and nothing in it is
crawled by search engines, so server-side rendering is costing me speed
for no benefit. Please make it a fully client-rendered app and make
in-app navigation instant.

Do all of the following:

1. Disable SSR globally in TanStack Start by setting defaultSsr to false
   in the start instance.
2. Find every place that navigates by changing the URL directly, such as
   window.location.href, window.location.assign, window.location.replace,
   or a plain <a href> pointing at an internal route. Replace all of them
   with the router's Link component or the navigate function so no
   navigation triggers a full page load. Leave external links alone.
3. Move route data loading to the client so route changes do not wait on
   a server round trip.
4. Make sure any redirects in auth guards happen through the router on
   the client, not as server redirects.
5. Enable link preloading on intent so a screen starts loading when the
   user touches the link.
6. Add a lightweight loading state for routes that fetch data, so the
   screen changes immediately and the content fills in, instead of the
   app freezing until the data arrives.

Do not change my UI, my styling, or my Supabase schema. Tell me which
files you changed and list every navigation you converted.
```

**If the same project holds your landing page and your app,** paste this instead:

```text
My project has a public marketing site and a logged-in app. The marketing
pages need SEO. The app does not, and server-side rendering is making
in-app navigation slow on mobile.

Do all of the following:

1. Keep the public marketing routes server-side rendered exactly as they
   are. Do not set defaultSsr to false, because that would disable SSR on
   the root route and cascade to everything.
2. Set ssr: false on the layout route that all of the logged-in app
   routes sit under, so it cascades down to its children only.
3. Inside the app section, replace every window.location navigation and
   every plain <a href> pointing at an internal route with the router's
   Link component or the navigate function.
4. Move data loading for the app routes to the client so route changes do
   not wait on a server round trip.
5. Make sure auth guard redirects inside the app happen through the
   router on the client, not as server redirects.
6. Enable link preloading on intent, and add a loading state for app
   routes that fetch data.

Do not change my UI, my styling, or my Supabase schema. Tell me which
routes are still server-rendered when you are done.
```

Ask for the list of changed files at the end because that is your receipt. If Lovable comes back having touched your marketing routes, say so and have it put them back.

The code below is here so you can check what you got, which is a different skill from writing it and a more useful one. The global switch lives in the start instance:

```ts
// src/start.ts
import { createStart } from '@tanstack/react-start'

export const startInstance = createStart(() => ({
  defaultSsr: false,
}))
```

And the per-route version, which is the one you usually want:

```ts
export const Route = createFileRoute('/_app')({
  ssr: false,
  component: AppLayout,
})
```

The inheritance rule matters and it catches people. A parent route's `ssr` value cascades to its children. Setting `defaultSsr: false` turns SSR off at the root, which means a route-level `ssr: true` further down will not bring it back. So split downward: leave the default alone, and switch SSR off on the layout route that everything in the app sits under.

## It will look like nothing happened

Give it a few minutes. Edge caches and the WebView both hold onto the old build, so the first test after the change often shows you exactly what you saw before and sends you off chasing the wrong thing.

Wait, force quit the app, and reopen it. Despia's default deployment fetches the current web build on launch, so there is no rebuild and no store submission for a change like this, but the launch has to actually happen. Reopening from the app switcher is not the same as reopening the app.

The reports we get back after this land around 5x on in-app navigation. That number is not a benchmark, it is what happens when a screen change stops being a document load and starts being a state change.

## The split that solves it properly

The version of this that scales is the one large apps already use: the marketing site and the product are two different things with two different jobs.

Your landing page, pricing page, and blog need SSR. They exist to be found. Rendering them on the server is the entire point, and you should leave that alone.

Your app does not need SSR at all. Nobody is crawling a screen behind a login. It needs to change screens instantly and hold state between them, which is what a client-rendered app is good at.

You can express that split inside one Lovable project with the route tree, which is what the `ssr: false` layout route above is doing. Or you can go further and deploy them separately, the site on its own domain and the app on `app.yourdomain.com`, which is cleaner, decouples your release cadence, and means a marketing change can never slow down the product. Either shape works. The route-level split is a prompt away and worth doing first.

## When to leave SSR on

If the app itself contains public content that has to rank, keep SSR on those routes specifically. Public profiles, marketplace listings, event pages, anything with a shareable URL that a stranger might land on from search. Those pages benefit from server rendering for the same reason your landing page does.

Set `ssr: false` on the authenticated layout and leave the public branch of the route tree rendering on the server. Selective SSR exists precisely for this, and a per-route decision beats a global one on both sides.

## Turning the Lovable app into a mobile app

If you are reading this because your app already runs on a phone, skip ahead. If you are still at the "I have a Lovable app and I want it in the App Store" stage, the short version:

Publish the Lovable app so it has a live URL. In Despia, create an app and point it at that URL. Install the `despia-native` package by asking Lovable to add it, which is how you get the device features. Connect your Apple and Google accounts, build, and submit. There is no Mac, no Xcode, and no CLI in that path.

What you end up with is one codebase. The Lovable project stays the thing you edit, it ships as a real signed binary on both stores, and web changes keep going out over the air without a new store submission. That is also why the SSR fix above is a five minute change rather than a release: you fix it in Lovable, the app picks it up on next launch.

## Then make it feel native

Fixing navigation speed is the prerequisite, not the finish line. Once a screen change is instant, the things that read as native start landing, because there is no lag underneath them to fight.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

function go(path) {
  if (isDespia) despia('lighthaptic://')
  navigate({ to: path })
}
```

A light haptic on tap, correct safe area handling, native share and biometrics where they belong. None of that helps while every navigation costs a second, which is why the order matters: make it fast, then make it feel native.

That one is a prompt too:

```text
Install the despia-native package. Detect whether the app is running
inside Despia by checking for "despia" in the lowercased user agent, and
only call despia() when it is. Fire a light haptic on tab bar taps and
primary button presses. Do not add haptics to scrolling or to every
element.
```

## FAQ

**Do I need to rebuild the app natively?** Not for this. A rewrite gives you a second codebase, a second release process, and store review on every change. What you are looking at is a rendering setting.

**How do I know which stack my project is on?** Look for TanStack Start in your dependencies. New Lovable projects created after May 13, 2026 use it. Older ones are React plus Vite unless you ran the upgrade.

**Will turning off SSR hurt my SEO?** Only on the routes you turn it off for, and those are the routes behind your login. Leave the public pages server rendered.

**Why does it still feel slow after the change?** Either the old build is still cached, or your navigation is still doing document loads. Check the Network panel for `document` requests per screen change before assuming SSR was not the cause.

**Does this need a new App Store build?** No. Web changes go out over the air. The binary picks up the current build on next launch.

## Get it on the stores

Take the app you just made fast and ship it to iOS and Android without a CLI or a Mac. Code signing and submission run from the browser, and web changes keep going out over the air.

[See the setup docs at setup.despia.com](https://setup.despia.com)