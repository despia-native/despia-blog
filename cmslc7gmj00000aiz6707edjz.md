---
title: "Lovable Offline Mode: Service Workers and Local Data"
seoTitle: "Lovable Offline Mode: Service Workers and Local Data"
seoDescription: "Lovable offline mode done properly: service worker caching, an IndexedDB mirror of your database, and a write queue that syncs when you reconnect."
datePublished: 2026-08-09T05:03:28.774Z
cuid: cmslc7gmj00000aiz6707edjz
slug: lovable-offline-mode-service-workers-and-local-data
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/4f613d86-144f-4143-9ace-aae93f2bddab.png
tags: webapp, pwa, web-api, indexeddb, lovable

---

%[https://youtu.be/dSqgoAYM6qw] 

Most people try this once, in the wrong order. They flip Offline Support to PWA in Despia, rebuild, put the phone in airplane mode, and the app opens to the native network error page. Or it opens, shows the shell, and then spins forever on an empty screen. Or worse: it works offline, and now none of their updates reach users ever again.

Two things are going on. Caching the app and running the app offline are separate problems, and a service worker that solves the first one badly will quietly break your deployments. A worker keeps your HTML, JavaScript and CSS available without a network. It does nothing for the Lovable Cloud queries, edge functions and third party APIs your app calls the moment it boots. And if it answers every request from its stored copy, your users are pinned to whatever version they installed first.

The video above is the whole build, unedited: prompting Lovable, catching the worker serving stale code, arguing it back into shape, and testing it on a real phone. The code is open source at [despia-native/lovable-service-worker-sample](https://github.com/despia-native/lovable-service-worker-sample), a small Lovable todo app with email login, a hand-written worker, an IndexedDB mirror and an outbox, under Apache 2.0. Getting it right took about 42 minutes even with good documentation, and you can hand that repository URL to Lovable as a reference instead of repeating the exercise. This post is the reasoning behind it.

## What the Despia offline toggle actually does

In the Despia Editor, go to Settings > Configuration and scroll to Dynamic App Source. That is where your Start URL points at your published Lovable project, and directly underneath it is Offline Support, with three options: none, PWA, and native. Select PWA. Then bump the version code and publish a new build for iOS, Android, or both.

That toggle provisions the Xcode and Android Studio projects to allow service workers in the native binary, without you touching either. Done manually, this is registering app bound domains on iOS and the equivalent on Android, in native project settings, which is not a path a non-native developer wants to walk. Here it is one setting and a rebuild.

What it does not do is write a service worker. If your Lovable app does not ship one, or ships one that never caches anything, the toggle changes nothing and every offline launch fails exactly as before. Despia removes the native obstacle. The caching strategy is yours, in your web codebase, using standard web APIs.

The other thing worth saying out loud, because it costs people an afternoon: offline support is a native runtime setting, so it only exists in binaries produced after the toggle was flipped. If you enable it and keep testing the build already on your phone, service worker registration is silently ignored, caches stay empty, and the Editor still reads as on. Install the new build before you conclude anything.

## Which Lovable stack are you on

TanStack Start became the default for new Lovable projects on May 13, 2026, and for new Enterprise projects on June 22, 2026. Anything created before that is the older React plus Vite stack, a client-rendered single page app deployed as static files. The stack is locked at creation and a prompt will not change it, so this is a fact about your project, not a choice you get to make now. Check your exported code for `vite.config.ts`, or open your published URL with JavaScript disabled and see whether content arrives in the HTML.

Both stacks work. The sample repository is TanStack Start on Lovable Cloud, so server rendering is not a blocker. Two things differ. Hashed build output lives under `/assets/` on the Vite stack and `/_build/` on TanStack Start, and your worker needs to recognise both if you want a rule that survives a stack change. And on the SSR stack a cached route document is a snapshot of a server render taken when it was cached, so it is an offline fallback rather than a source of truth, which is exactly how the strategy below treats it.

## A worker that stays fresh online and only falls back offline

This is the part that decides whether your app can still be updated. Say it as a rule and it becomes obvious:

A bad worker asks "is there a cached copy" and serves it. A good worker asks "am I online", takes the network when the answer is yes, and only reaches for the cache when the network genuinely fails.

Cache-first on navigation is the default in most tutorials and in most AI-generated workers, and it is the single worst choice for an app that updates over the air. You publish in Lovable, the content updates on your host, and nobody sees it. Here is the shape that works, condensed from `public/sw.js` in the sample:

```javascript
const VERSION = 'v7'
const CACHE = `app-${VERSION}`
const PRECACHE = ['/', '/offline.html']
const NETWORK_TIMEOUT_MS = 3000

// content-hashed build output, safe to cache forever because the name changes
function isHashedAsset(url) {
    return /^\/(assets|_build)\//.test(url.pathname)
        && /\.[0-9a-zA-Z_-]{8,}\.(js|css|woff2?|png|jpg|jpeg|svg|webp|avif|ico)$/.test(url.pathname)
}

self.addEventListener('install', event => {
    event.waitUntil((async () => {
        const cache = await caches.open(CACHE)
        await Promise.allSettled(PRECACHE.map(async path => {
            const res = await fetch(path, { cache: 'no-store' })
            if (res.ok) await cache.put(path, res)
        }))
        await self.skipWaiting()
    })())
})

self.addEventListener('activate', event => {
    event.waitUntil((async () => {
        if (self.registration.navigationPreload) {
            await self.registration.navigationPreload.enable().catch(() => {})
        }
        const names = await caches.keys()
        await Promise.allSettled(
            names.filter(n => n.startsWith('app-') && n !== CACHE).map(n => caches.delete(n))
        )
        await self.clients.claim()
    })())
})

self.addEventListener('fetch', event => {
    const { request } = event
    if (request.method !== 'GET') return

    const url = new URL(request.url)
    if (url.origin !== self.location.origin) return

    if (isHashedAsset(url)) {
        event.respondWith(cacheFirst(request))
        return
    }

    const isDocument = request.mode === 'navigate'
    event.respondWith((async () => {
        try {
            return await fromNetwork(request, {
                preload: isDocument ? event.preloadResponse : undefined,
                noStore: isDocument
            })
        } catch {
            return offlineFallback(request)
        }
    })())
})

async function fromNetwork(request, { preload, noStore } = {}) {
    const cache = await caches.open(CACHE)
    const attempt = preload
        ? Promise.resolve(preload).then(res => res || fetch(request))
        : fetch(noStore ? new Request(request, { cache: 'no-store' }) : request)

    const timeout = new Promise((_, reject) =>
        setTimeout(() => reject(new Error('network-timeout')), NETWORK_TIMEOUT_MS)
    )

    const res = await Promise.race([attempt, timeout])
    if (res && res.ok && res.type !== 'opaque') {
        cache.put(request, res.clone()).catch(() => {})
    }
    return res
}

async function offlineFallback(request) {
    const cache = await caches.open(CACHE)
    return (await cache.match(request))
        || (request.mode === 'navigate'
            ? (await cache.match('/')) || (await cache.match('/offline.html'))
            : undefined)
        || Response.error()
}
```

Four details in there are not optional.

The timeout matters more than it looks. A hanging connection does not reject `fetch()`, it sits there, so a plain `fetch().catch()` fallback stalls on a bad train connection instead of falling back. Race it.

`cache: 'no-store'` on document fetches bypasses the browser's own HTTP cache and any CDN copy. Without it, the worker can go to the network, do everything right, and still be handed a stale HTML response, which looks identical to a broken worker and sends you debugging the wrong file.

Navigation preload starts the document request in parallel with worker startup, which removes the roughly one second delay you will otherwise see on the document in the network tab.

And the offline fallback is a chain, not a single file: the last cached copy of that exact page, then the app shell, then `/offline.html`. Make `/offline.html` render something useful, because it is the last thing standing between a dropped request and a blank screen.

Registration matters as much as the worker. `updateViaCache: 'none'` stops the browser reusing a cached copy of `sw.js` itself, which is how a broken worker becomes permanent. Then check for updates on load, on focus, on reconnect, and reload when control changes hands:

```javascript
const registration = await navigator.serviceWorker.register('/sw.js', {
    scope: '/',
    updateViaCache: 'none'
})

let reloading = false
navigator.serviceWorker.addEventListener('controllerchange', () => {
    if (reloading) return
    reloading = true
    window.location.reload()
})

const checkForUpdate = () => void registration.update().catch(() => {})
window.addEventListener('focus', checkForUpdate)
window.addEventListener('online', checkForUpdate)
document.addEventListener('visibilitychange', () => {
    if (document.visibilityState === 'visible') checkForUpdate()
})
checkForUpdate()
```

One more thing the sample does that is easy to miss: it refuses to register at all on Lovable preview and dev hostnames, and unregisters anything already there. The editor preview is not your deployment, and a worker installed against it will fight hot reload and confuse every test you run afterwards.

Do not hand-write a precache list of hashed filenames. Vite renames them on every build, so a list written today points at files that will not exist next week, `install` fails, and the worker never activates. Precache the shell and the offline page, and let everything else fill in as it is visited.

## Offline means your database is gone, and that is the part people miss

Here is what turns a working service worker into a broken app. Your cached shell boots. It renders. Then it does what Lovable wrote it to do, which is call your backend.

Lovable Cloud is a managed Supabase instance, and connecting your own Supabase project is the alternative. Either way, every `supabase.from(...)` read is an HTTPS request to a server on the other side of the planet. So is every edge function, every AI call, every Stripe request. With no network there is no server to talk to, and a cached page that renders a spinner until a query resolves will spin until the request times out and then show an error, if you wrote one.

The mental model has to change. You are not fetching data, you are reading data that is already on the device and refreshing it when you can.

| What the user does offline | Works from cache alone | Needs a local data layer |
| --- | --- | --- |
| Opens the app, sees the shell and navigates | Yes | No |
| Loads JS, CSS and previously visited images | Yes | No |
| Reads their existing records | No | Yes, mirrored into IndexedDB |
| Creates or edits a record | No | Yes, optimistic write plus outbox |
| Runs an edge function, AI call or payment | No | No, disable and explain |
| Stays signed in | Session persists locally | Token refresh needs network |
| Receives a realtime update | No | No, resync on reconnect |

So: on first load, while online, pull the records this user needs and write them into IndexedDB along with the current user. From then on, read local first, render immediately, then refresh from the backend in the background and merge. That is one code path that feels instant on a good connection and keeps working on none.

Use IndexedDB for records. `localStorage` is fine for a token and a small user object, and wrong for lists: synchronous, string-only, and small enough that a few hundred rows hit the wall. Keep authenticated backend responses out of the Cache API too. That cache is keyed by URL with no notion of who is signed in. Mirror rows into IndexedDB instead, where you control the shape and the lifetime.

## The write path: optimistic locally, queued, flushed on reconnect

Every offline write is two operations. Update the local copy so the UI responds immediately, and record the intent so it can be replayed later. The sample keeps three IndexedDB stores for this: the mirrored table, an `outbox` of queued jobs, and `tombstones` for deletes.

```javascript
async function saveTodo(todo) {
    const row = { ...todo, updated_at: new Date().toISOString(), pending: true }

    await localTodos.put(row)
    await outbox.add({ type: 'insert', row })

    flushOutbox()
    return row
}

async function flushOutbox() {
    if (!navigator.onLine) return

    for (const job of await outbox.all()) {
        const error = await runJob(job)

        if (error) {
            // never reached the server, keep it queued and try again later
            if (isNetworkFailure(error)) return

            // server said no, retrying will not change that
            await outbox.remove(job.key)
            await revertJob(job)
            reportSyncError(error)
            continue
        }

        await outbox.remove(job.key)
        await settleJob(job)
    }
}

window.addEventListener('online', flushOutbox)
document.addEventListener('visibilitychange', () => {
    if (document.visibilityState === 'visible') flushOutbox()
})
```

Five details separate a sync layer from a data loss bug.

Strip your local-only fields before sending. A `pending` flag that reaches Postgres is an unknown column and the write is rejected, and you will spend an hour blaming the queue.

Separate a transport failure from a server rejection. A dropped connection should keep the job queued. A row your Row Level Security policies refuse will never succeed, so drop it, undo the local optimistic change, and tell the user. An offline write is a request, not a guarantee, and the UI should say pending until the server has accepted it.

Flush on `visibilitychange` as well as the `online` event. A phone that was asleep in a lift comes back through resume, and does not always fire a connectivity event.

Queue deletes explicitly and keep a tombstone until the delete lands, or the row reappears on the next pull.

For conflicts, compare `updated_at` and let the newer write win, with one exception the sample makes explicit: a pending local write always wins over the server copy, because it has not been sent yet and dropping it would lose the user's work.

## Auth, and why the app can look signed out when it is not

The Supabase client persists the session in local storage by default, so a cold start with no network still has a session to read. The app appears signed out anyway when first render is blocked on a call that cannot complete: a session refresh, a profile fetch, a role lookup. The await never resolves, the loading state never clears, and the user sees a spinner or a login screen.

Read the stored session, render from it, then revalidate when the network returns.

```javascript
function loadSession() {
    try {
        return JSON.parse(localStorage.getItem('session'))
    } catch {
        return null
    }
}

async function boot() {
    const session = loadSession()

    if (!navigator.onLine) {
        return session ? renderApp(session) : renderOfflineNotice()
    }

    const fresh = await refreshSession()
    localStorage.setItem('session', JSON.stringify(fresh))
    renderApp(fresh)
}

window.addEventListener('online', boot)
boot()
```

Treat that cached session as a rendering hint, not as authorization. Anything sensitive is verified server side on the next successful request, and the local mirror should only ever hold data that user was already allowed to read.

If you want the offline state to look different inside the native app than in a browser tab, the environment check is the user agent:

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')
```

## Verify in three steps, and never skip one

This is the part that saves the money. Service workers, the Cache API and IndexedDB are web platform APIs. The Despia runtime runs the same engine your browser runs, so it does not add behaviour to them and it cannot repair them. Works in the browser, works in the PWA, works in the native build. Fails anywhere earlier and it fails everywhere later, except that in the binary the failure hides behind a native error screen and looks like a runtime bug.

So the order is fixed. Step one, prove updates still reach users. Step two, prove the web app works offline as an installed PWA. Step three, prove the native build works offline. People who jump to step three burn hundreds of dollars in AI tokens debugging something they could have excluded in ten minutes.

If you want to watch this happen rather than read it, this is most of the video above, including the parts where it does not work.

**Step one: prove over-the-air updates still work, online.**

Do this before you test offline at all, because a worker that caches beautifully and never updates is worse than no worker.

Publish from Lovable and open the published URL, not the editor preview. Open developer tools, two finger click and choose inspect if you do not have the menu, and enable developer tools in browser settings if the option is missing. Go to the network tab and reload. The gear icon next to a request means it was served by the service worker. Check that your CSS is in there too, not only the HTML and JavaScript, or the app will render as a mess offline. Screenshot that tab and paste it into Lovable with a line like "here is the network tab from the dev tools, make sure the full app experience is cached". Being proactive here is much cheaper than debugging from inside the failure.

Ignore requests injected by browser extensions. They show up in the network tab, they look like your app failing to cache something, and they are not.

Then change something obvious and publish. The theme colour is the crudest and fastest signal: make it pink, publish, reload. If the colour does not change, the worker is corrupt, and no amount of offline testing means anything yet. Escalate through all three kinds of change before you move on: CSS styling, then HTML structure, then JavaScript logic, since a worker can pass one and fail the next.

**You will trap yourself, and that is normal.** The first bad worker you ship installs itself on your own machine, and from then on your browser keeps serving the old copy no matter how many times you fix the code and reload. Adding a `?cachebust=123` query parameter sometimes works. Opening a 404 route sometimes works. Often neither does. The practical answer is a second browser and a private window: Safari when you have been testing in Chrome, then a fresh incognito window each time. Clearing the cache properly is workable on macOS and Windows, awkward on Android, and genuinely painful on iOS. Assume you will need clean sessions and stop fighting it.

When the worker is wrong, tell the AI exactly what you observed. These two prompts came out of the sample build and both moved it forward:

```text
I made the update to pink and if I reload my app, the update did not go through.
This means the service worker you wrote is corrupted. The service worker needs to load
the latest web code if the user is online and update over the air. If the user is
offline, serve cache, but once online make a new latest copy.
```

```text
I opened a private Safari window and tested the colour change. The app is still green
and does not show the updates I made, so it is infinitely stuck on old page code, which
means the service worker is corrupted. Please fix it.
```

Attach the network tab screenshot. Ask for a plan first when you are two rounds in and it is still wrong, so the model reads the whole registration path instead of patching one file. And read the plan before approving it: at one point in the sample build the AI proposed replacing the worker with a cleanup worker that unregisters itself, which does guarantee fresh code and also deletes the offline support you asked for.

Do not take "I tested it and it works" from an AI tool as a result. Their browser testing and your own physical device are not the same thing.

**Step two: install it as a PWA and go offline.**

Open your published URL on a real phone and add it to the home screen. If it opens with an address bar at the top or a share and reload bar at the bottom, it opened as a browser tab, not a PWA, and you are missing a web app manifest and the Apple specific head tags. Fix that even if you never intend to ship a PWA, because you need it to run this test. If it still opens wrong after the fix, delete it from the home screen, reload the page, and add it again, waiting a couple of seconds before tapping add.

Then use the app online first. Load the screens that matter, sign in, create a record. That is what fills the caches and writes rows into IndexedDB, and a worker with an empty cache is indistinguishable from no worker.

Now turn off Wi-Fi and cellular data separately in system settings. Not airplane mode, since some devices keep Wi-Fi alive in it and the test passes for the wrong reason. Force quit the installed app and open it cold, because a warm relaunch proves nothing. You want to see: the app opens, your offline indicator appears, your existing records are on screen, a new record can be created and shows as pending. Turn the radios back on and watch that record sync.

**Step three: the native build.**

Only now flip Offline Support to PWA in Despia, bump the version code, publish, and install the new build. Run the same sequence: use it online, kill both radios, force quit, cold launch past the native splash screen, read your data, write a record, reconnect and watch it sync. If steps one and two passed, this one passes, because it is the same web app with the same caches inside a native shell.

The one thing that genuinely differs is the very first launch. A hosted app has to be opened once with a working connection before the worker can install and precache anything, so a user who installs from the store on a plane has nothing to serve.

## The pitfalls that generate the most support tickets

**A stale worker that outlives your fix.** A cache-first worker keeps serving the old app to everyone who already installed it, including after you fix the worker, because `sw.js` is itself a file the old worker is caching. `updateViaCache: 'none'` prevents it. If you are already in that hole with real users, ship a kill switch, serve the worker path with no-cache headers, and let it run once.

```javascript
self.addEventListener('install', () => self.skipWaiting())
self.addEventListener('activate', event => {
    event.waitUntil(
        caches.keys()
            .then(keys => Promise.all(keys.map(k => caches.delete(k))))
            .then(() => self.registration.unregister())
            .then(() => self.clients.claim())
    )
})
```

**A cached document pointing at deleted assets.** If the HTML is cached and the hashed JavaScript it references is not, the app opens to a white page. Network-first documents plus a versioned cache prevents it.

**Testing in the Lovable editor preview.** The preview is not your deployment. Scope, headers and worker behaviour only mean anything on the published URL that Despia actually loads.

**Debugging the native build first.** Almost every reported offline failure is a worker that never cached anything, and the binary is the worst place to find that out.

## A prompt that sets this up in one shot

Paste this into Lovable. It covers the worker, the local mirror and the queue together, because asking for offline support alone reliably produces a cache-first worker and nothing else. Expect to iterate anyway. Getting the sample right took roughly 42 minutes of the loop above, and that is with the same instructions.

```text
Add offline support to this app. Read https://setup.despia.com/native-features/service-workers first,
and use https://github.com/despia-native/lovable-service-worker-sample as a reference implementation
(public/sw.js, src/lib/sw-register.ts, src/lib/local-db.ts, src/lib/sync.ts). This app is shipped as a
native iOS and Android build with Despia, and the runtime allows a standard service worker.

Service worker, always fresh online, cache only as an offline fallback:
- Serve the worker from the root of the site, registered with scope '/' and updateViaCache: 'none'.
- Documents and non-hashed files: network-first with a 3 second timeout and cache: 'no-store' on the
  document request, so no browser or CDN copy shadows a new build. Enable navigation preload.
- Content-hashed build output under /assets/ or /_build/: cache-first, safe because the name changes.
- Offline fallback chain: last cached copy of that page, then the app shell, then /offline.html.
- Do NOT use cache-first for navigation. It pins users to an old build and breaks over-the-air updates.
- Do NOT hand-write a precache list of hashed filenames. Precache only '/' and '/offline.html'.
- Version the cache name, delete every non-matching cache on activate, skipWaiting on install,
  clients.claim on activate.
- Call registration.update() on load, focus, online and visibilitychange, and reload the page on
  controllerchange so a new worker takes effect immediately.
- Do not register the worker on Lovable preview or dev hostnames, and unregister any worker found there.
- Add a web app manifest and Apple specific head tags so the app installs as a real PWA.

Local data layer, because the database is unreachable offline:
- Mirror the tables this app reads into IndexedDB. Read from IndexedDB first and render from it, then
  refresh from the backend in the background and merge.
- Never block first render on a network call. No await on a query, session refresh or profile fetch
  before something is on screen.
- Writes are optimistic: update the local row with a pending flag and queue a job in an IndexedDB outbox.
- Flush the outbox on the window 'online' event and on visibilitychange. Strip local-only fields such as
  pending before sending. Keep a job queued on a network failure. On a server rejection, drop the job,
  revert the local change and surface an error, since RLS failures never succeed on retry.
- Queue deletes explicitly with a local tombstone so deleted rows do not come back on the next pull.
- Resolve conflicts on updated_at, newest write wins, except that a pending local write always wins.
- Show a clear pending or offline indicator in the UI.
- Disable online-only features (AI calls, payments, edge functions, email) with an explanation rather
  than letting them fail silently.

Do not add a PWA plugin that generates a cache-first worker.
```

Then enable it on the native side: Settings > Configuration > Dynamic App Source > Offline Support > PWA, bump the version code, and publish a new build.

## Get it on the stores

Take the app you just built and ship it to iOS and Android without a CLI or a Mac. Code signing and submission run from the browser.

[See the setup docs at setup.despia.com](https://setup.despia.com)