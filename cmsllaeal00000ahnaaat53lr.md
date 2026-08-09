---
title: "Base44 Offline Mode: Service Workers and Local Data"
seoTitle: "Base44 Offline Mode: Service Workers and Local Data"
seoDescription: "Base44 offline mode done properly: a service worker in /public, an IndexedDB mirror of your entities, and a write queue that syncs on reconnect."
datePublished: 2026-08-09T09:17:42.267Z
cuid: cmsllaeal00000ahnaaat53lr
slug: base44-offline-mode-service-workers-and-local-data
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/158d5975-247e-4038-96d3-76a35ddf22d0.png
tags: webapi, webapp, pwa, indexeddb, base44

---

%[https://youtu.be/ebflykFHCkg] 

Most people try this once, in the wrong order. They flip Offline Support to PWA in Despia, rebuild, put the phone in airplane mode, and the app opens to the native network error page. Or it opens, shows the shell, and then spins forever on an empty screen. Or worse: it works offline, and now none of their updates reach users ever again.

Two things are going on. Caching the app and running the app offline are separate problems, and a service worker that solves the first one badly will quietly break your deployments. A worker keeps your HTML, JavaScript and CSS available without a network. It does nothing for the entity reads, backend functions and third party APIs your app calls the moment it boots. And if it answers every request from its stored copy, your users are pinned to whatever version they installed first.

The video above is the whole build: prompting Base44, catching the first attempt failing on a real phone, correcting it, and testing the same app as a PWA and as a native build. The code is open source at [despia-native/base44-service-worker-sample](https://github.com/despia-native/base44-service-worker-sample), a small task app with email login, a hand-written worker, an IndexedDB mirror and an outbox. You can hand that repository URL to Base44 as a reference. This post is the reasoning behind it.

## Where the service worker goes

A service worker only controls the routes inside its own scope, so it has to be served from the root of your domain to control the whole app. Base44 serves files from a `public` folder at the root of your published app, so the worker goes at `public/sw.js` and is reachable at `https://your-app-domain.com/sw.js`. Put `offline.html` and your web app manifest in the same folder.

Confirm that before you write a line of caching logic. Open `https://your-app-domain.com/sw.js` directly in a browser. You want your JavaScript, served as JavaScript. If you get your app's HTML instead, the path is falling through to the single page app rewrite, the file is not at the root, and every registration attempt fails with a content type error that reads like a code problem and is not one.

Registering from a blob URL is not an alternative. `register()` requires an http or https script URL, so a blob is rejected outright.

## What the Despia offline toggle actually does

In the Despia Editor, go to Settings > Configuration and scroll to Dynamic App Source. That is where your Start URL points at your published Base44 app, and directly underneath it is Offline Support, with three options: none, PWA, and native. Select PWA. Then bump the version code and publish a new build for iOS, Android, or both.

That toggle provisions the Xcode and Android Studio projects to allow service workers in the native binary, without you touching either. Done manually, this is registering app bound domains on iOS and the equivalent on Android, in native project settings, which is not a path a non-native developer wants to walk. Here it is one setting and a rebuild.

What it does not do is write a service worker. If your Base44 app does not ship one, or ships one that never caches anything, the toggle changes nothing and every offline launch fails exactly as before. Despia removes the native obstacle. The caching strategy is yours, in your web codebase, using standard web APIs.

The other thing worth saying out loud, because it costs people an afternoon: offline support is a native runtime setting, so it only exists in binaries produced after the toggle was flipped. If you enable it and keep testing the build already on your phone, service worker registration is silently ignored, caches stay empty, and the Editor still reads as on. Install the new build before you conclude anything.

## A worker that stays fresh online and only falls back offline

This is the part that decides whether your app can still be updated. Say it as a rule and it becomes obvious:

A bad worker asks "is there a cached copy" and serves it. A good worker asks "am I online", takes the network when the answer is yes, and only reaches for the cache when the network genuinely fails.

Cache-first on navigation is the default in most tutorials and in most AI-generated workers, and it is the single worst choice for an app that updates over the air. You publish in Base44, the content updates on your host, and nobody sees it. Here is the shape that works:

```javascript
const VERSION = 'v1'
const CACHE = `app-${VERSION}`
const PRECACHE = ['/', '/offline.html']
const NETWORK_TIMEOUT_MS = 3000

// content-hashed build output, safe to cache forever because the name changes
function isHashedAsset(url) {
    return /^\/assets\//.test(url.pathname)
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

There is no stack question to answer here. Every Base44 app is React and Vite, built to a static frontend with an `index.html` rewrite for routes, which makes two things true. Hashed build output lands under `/assets/`, so the test above is the right one. And every route resolves to the same document, so precaching the shell precaches every screen in the app rather than one page at a time.

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

Skip registration entirely in the Base44 editor preview, and unregister anything already installed there. The reliable test is not the hostname, it is whether you are running inside a frame:

```javascript
function isEditorPreview() {
    try {
        return window.self !== window.top
    } catch {
        return true // cross-origin frame, treat as preview
    }
}
```

The preview is not your deployment, and a worker installed against it will fight the editor's live reload and confuse every test you run afterwards.

Do not hand-write a precache list of hashed filenames. The build renames them every time, so a list written today points at files that will not exist next week, `install` fails, and the worker never activates. Precache the shell and the offline page, and let everything else fill in as it is visited.

## Offline means your database is gone, and that is the part people miss

Here is what turns a working service worker into a broken app. Your cached shell boots. It renders. Then it does what Base44 wrote it to do, which is call your backend.

Every `base44.entities.Todo.list()` is an HTTPS request to a server on the other side of the planet. So is `base44.auth.me()`, so is every backend function you invoke, every AI call, every payment. With no network there is no server to talk to, and a cached page that renders a spinner until a query resolves will spin until the request times out and then show an error, if you wrote one.

The mental model has to change. You are not fetching data, you are reading data that is already on the device and refreshing it when you can.

| What the user does offline | Works from cache alone | Needs a local data layer |
| --- | --- | --- |
| Opens the app, sees the shell and navigates | Yes | No |
| Loads JS, CSS and previously visited images | Yes | No |
| Reads their existing records | No | Yes, mirrored into IndexedDB |
| Creates or edits a record | No | Yes, optimistic write plus outbox |
| Runs a backend function, AI call or payment | No | No, disable and explain |
| Stays signed in | Only if you cached the user | Cache the user object locally |
| Sees another user's changes | No | No, resync on reconnect |

So: on first load, while online, pull the records this user needs and write them into IndexedDB along with the current user. From then on, read local first, render immediately, then refresh from Base44 in the background and merge. That is one code path that feels instant on a good connection and keeps working on none.

Use IndexedDB for records. `localStorage` is fine for a token and a small user object, and wrong for lists: synchronous, string-only, and small enough that a few hundred rows hit the wall. Keep entity responses out of the Cache API too. That cache is keyed by URL with no notion of who is signed in. Mirror rows into IndexedDB instead, where you control the shape and the lifetime.

## The write path: optimistic locally, queued, flushed on reconnect

Every offline write is two operations. Update the local copy so the UI responds immediately, and record the intent so it can be replayed later. Keep the mirrored entity, an `outbox` of queued jobs, and tombstones for deletes.

```javascript
const LOCAL_FIELDS = ['_key', '_entity', '_pending', '_deleted', '_localId']

function stripLocal(row) {
    const clean = { ...row }
    LOCAL_FIELDS.forEach(f => delete clean[f])
    return clean
}

async function create(values) {
    const id = `local_${Date.now().toString(36)}`
    const now = new Date().toISOString()

    await putRecord(entityName, { ...values, id, created_date: now, updated_date: now, _pending: true })
    await outboxAdd({ jobId: uid(), entity: entityName, op: 'create', id, values: stripLocal(values) })

    flush()
}

async function flush() {
    if (flushing || !navigator.onLine) return
    flushing = true

    try {
        for (const job of await jobsFor(entityName)) {
            try {
                if (job.op === 'create') {
                    const created = await api.create(job.values)
                    await deleteRecord(entityName, job.id)   // drop the temporary local row
                    await putRecord(entityName, created)
                } else if (job.op === 'update') {
                    await putRecord(entityName, await api.update(job.id, job.values))
                } else {
                    await api.delete(job.id)
                    await deleteRecord(entityName, job.id)
                }
                await outboxDelete(job.jobId)
            } catch (err) {
                const rejected = typeof err?.status === 'number' && err.status >= 400 && err.status < 500

                if (!rejected) break // network failure, keep the job queued

                await outboxDelete(job.jobId)
                await revert(job)
                emitError(err?.message || 'A change could not be saved and was undone.')
            }
        }
    } finally {
        flushing = false
    }
}

window.addEventListener('online', flushAll)
document.addEventListener('visibilitychange', () => {
    if (document.visibilityState === 'visible') flushAll()
})
```

Six details separate a sync layer from a data loss bug.

Strip your local-only fields before sending. Prefix them so this is mechanical rather than remembered: `_pending`, `_deleted`, `_localId`. A `_pending` flag that reaches the entity is a field the schema does not have, and the write is rejected. You will spend an hour blaming the queue.

Separate a transport failure from a server rejection, and use the status code rather than a string match on the error. A 4xx is your entity permissions or your schema refusing the write, and it will never succeed on retry, so drop the job, undo the local optimistic change, and tell the user. Anything else is the network, so stop and keep the queue intact for the next attempt. An offline write is a request, not a guarantee, and the UI should say pending until the server has accepted it.

A row created offline has no server id, so give it a local one, and when the create finally lands, delete the temporary row and store the server's version. Skip that and you get the same task twice.

Flush on `visibilitychange` as well as the `online` event. A phone that was asleep in a lift comes back through resume, and does not always fire a connectivity event.

Queue deletes explicitly and write a tombstone, a row flagged `_deleted` that your list filters out, until the delete lands. Without it the row reappears on the next pull.

For conflicts, compare `updated_date` and let the newer write win, with one exception: a pending or deleted local row always wins over the server copy, because it has not been sent yet and dropping it would lose the user's work. Base44 timestamps entities with `created_date` and `updated_date`, so use those field names rather than inventing your own.

## Auth, and why the app can look signed out when it is not

`base44.auth.me()` is a network call. If your app awaits it before rendering anything, an offline cold start hangs on it: the promise never resolves, the loading state never clears, and the user sees a spinner or a login screen even though they are signed in.

Cache the user object locally on every successful call, render from that copy when you are offline, and revalidate when the network returns.

```javascript
function loadUser() {
    try {
        return JSON.parse(localStorage.getItem('user'))
    } catch {
        return null
    }
}

async function boot() {
    const user = loadUser()

    if (!navigator.onLine) {
        return user ? renderApp(user) : renderOfflineNotice()
    }

    const fresh = await base44.auth.me()
    localStorage.setItem('user', JSON.stringify(fresh))
    renderApp(fresh)
}

window.addEventListener('online', boot)
boot()
```

Treat that cached user as a rendering hint, not as authorization. Anything sensitive is verified server side on the next successful request, and the local mirror should only ever hold data that user was already allowed to read.

If you want the offline state to look different inside the native app than in a browser tab, the environment check is the user agent:

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')
```

## Verify in three steps, and never skip one

This is the part that saves the money. Service workers, the Cache API and IndexedDB are web platform APIs. The Despia runtime runs the same engine your browser runs, so it does not add behaviour to them and it cannot repair them. Works in the browser, works in the PWA, works in the native build. Fails anywhere earlier and it fails everywhere later, except that in the binary the failure hides behind a native error screen and looks like a runtime bug.

So the order is fixed. Step one, prove updates still reach users. Step two, prove the web app works offline as an installed PWA. Step three, prove the native build works offline. People who jump to step three burn hundreds of dollars in AI tokens debugging something they could have excluded in ten minutes. Worse, they debug the wrong layer: hand an AI a failure from inside a native binary and it will confidently tell you the Xcode project is misconfigured, when the actual fault is a worker that never cached anything.

Before any of that, pick your model. Base44 lets you choose, and a service worker is exactly the kind of task where that matters, since it is one file that has to be right in several ways at once. Use the strongest model available to you and put it in build mode. Tools that decide the model for you are where these implementations usually go wrong.

**Step one: prove over-the-air updates still work, online.**

Do this before you test offline at all, because a worker that caches beautifully and never updates is worse than no worker.

Publish from Base44 and open the published URL, not the editor preview. Open developer tools, two finger click and choose inspect if you do not have the menu, and enable developer tools in browser settings if the option is missing. Go to the network tab and reload. The gear icon next to a request means it was served by the service worker. Check that your CSS is in there too, not only the HTML and JavaScript, or the app will render as a mess offline. Screenshot that tab and paste it into Base44 with a line like "here is the network tab from the dev tools, make sure the full app experience is cached". Being proactive here is much cheaper than debugging from inside the failure.

Ignore requests injected by browser extensions. They show up in the network tab, they look like your app failing to cache something, and they are not.

Then change something obvious and publish. The theme colour is the crudest and fastest signal: make it pink, publish, reload. If the colour does not change, the worker is corrupt, and no amount of offline testing means anything yet. Escalate through all three kinds of change before you move on: CSS styling, then HTML structure, then JavaScript logic, since a worker can pass one and fail the next.

**You will trap yourself, and that is normal.** The first bad worker you ship installs itself on your own machine, and from then on your browser keeps serving the old copy no matter how many times you fix the code and reload. Adding a `?cachebust=123` query parameter sometimes works. Opening a 404 route sometimes works. Often neither does. The practical answer is a second browser and a private window: Safari when you have been testing in Chrome, then a fresh incognito window each time. Clearing the cache properly is workable on macOS and Windows, awkward on Android, and genuinely painful on iOS. Assume you will need clean sessions and stop fighting it.

When something is wrong, tell the AI exactly what you observed, in plain language, including where you observed it. These two cover the two failures you will actually hit:

```text
I made the update to pink and if I reload my app, the update did not go through.
This means the service worker you wrote is corrupted. The service worker needs to load
the latest web code if the user is online and update over the air. If the user is
offline, serve cache, but once online make a new latest copy.
```

```text
I tested it as a PWA on iOS and I get an error saying Safari cannot load the website
while offline. That means your service worker does not cache the data for offline PWA
usage. I need it to load the latest version if the network is online, and load cache
only if the network is offline. Never show old cache when online, and show the latest
cache when offline.
```

Attach the network tab screenshot. Ask for a plan first when you are two rounds in and it is still wrong, so the model reads the whole registration path instead of patching one file. And read the plan before approving it: a common AI escape route is to replace the worker with a cleanup worker that unregisters itself, which does guarantee fresh code and also deletes the offline support you asked for.

Do not take "I tested it and it works" from an AI tool as a result. Their browser testing and your own physical device are not the same thing. Getting a service worker right usually takes several rounds even with good documentation, so plan for the loop rather than a one-shot.

**Step two: install it as a PWA and go offline.**

You cannot test offline behaviour in a Safari tab on iOS, and testing it inside a native binary is the hardest environment to debug. The installed PWA is the middle rung: no Xcode, no Android Studio, no store review, and it exercises exactly the same worker. If your app is not meant to ship as a PWA, ask Base44 to make it one anyway and strip it later.

Open your published URL on a real phone and add it to the home screen. If it opens with an address bar at the top or a share and reload bar at the bottom, it opened as a browser tab, not a PWA, and you are missing a web app manifest and the Apple specific head tags. Fix that even if you never intend to ship a PWA, because you need it to run this test. If it still opens wrong after the fix, delete it from the home screen, reload the page, and add it again, waiting a couple of seconds before tapping add.

Then use the app online first. Load the screens that matter, sign in, create a record. That is what fills the caches and writes rows into IndexedDB, and a worker with an empty cache is indistinguishable from no worker.

Now turn off Wi-Fi and cellular data separately in system settings. Not airplane mode, since some devices keep Wi-Fi alive in it and the test passes for the wrong reason.

Your offline indicator lighting up proves nothing. It is interface, rendered by an app that is already open and already running. Force quit from the app switcher and launch it cold. That is the only version of this test that means anything, and it is where a worker that looked fine in the network tab will show you an address bar and a message about not being able to load the page.

What you want to see: the app opens, your existing records are on screen, and a new record can be created and shows as pending. Then turn the radios back on, watch the pending state clear, and reload the app on your desktop to confirm the record actually reached the database.

**Step three: the native build.**

Only now flip Offline Support to PWA in Despia, mark the version bump as a medium update, publish, and install the new build from TestFlight or internal testing. A deployment takes a few minutes, occasionally up to thirty. Run the same sequence: use it online, kill both radios, force quit, cold launch past the native splash screen, read your data, write a record, reconnect and watch it sync. If steps one and two passed, this one passes, because it is the same web app with the same caches inside a native shell.

The one thing that genuinely differs is the very first launch. A hosted app has to be opened once with a working connection before the worker can install and precache anything, so a user who installs from the store on a plane has nothing to serve.

## Wrapping it as a native app, and why the built-in publishing will not do

Base44's own mobile publishing is a thin wrapper. It puts your web app in a binary and that is the extent of it: no in-app purchases, no StoreKit or Android billing, no push provider, no iCloud key value storage or Android backup for data that survives an uninstall. Most relevant here, it does not register app bound domains on iOS, which is what a service worker needs in order to register inside a native binary at all. Offline support is an advanced feature, and a minimal wrapper is the wrong tool for it.

Two options actually work.

Capacitor is free and open source. You will need a Mac, Xcode, and Android Studio, and you will be archiving projects, editing plist values and the Android manifest by hand. If that is comfortable, it costs nothing but time.

Despia is the paid one, and it is ours, so weigh that accordingly. It exposes the same native capabilities without a Mac or a CLI: copy your Base44 project link into Settings > Configuration > Dynamic App Source as the app start URL, set Offline Support to PWA, then under Versioning choose a medium build version update, confirm, and publish. That is the app bound domain registration and the native offline provisioning done as a setting rather than an Xcode session.

One thing you will have to fix by hand either way: safe areas. Wrapped web content runs edge to edge, so your header slides under the status bar and the notch. Pull the safe area documentation, paste it into Base44, and let it apply the padding utilities. That is a web change, so it ships over the air with your next publish and needs no resubmission.

## The pitfalls that generate the most support tickets

**A worker that is not really at the root.** If `/sw.js` returns your app's HTML rather than JavaScript, the file is not being served from `/public` the way you think it is. Registration fails on the content type, and the error message points at your code rather than your hosting.

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

**iOS holding on to an old registration.** Safari caches service worker registrations aggressively. If you installed an earlier version to the home screen, a new worker may not take, and deleting the icon and adding it again is faster than any amount of debugging.

**Testing in the Base44 editor preview.** The preview is not your deployment. Scope, headers and worker behaviour only mean anything on the published URL that Despia actually loads.

**Debugging the native build first.** Almost every reported offline failure is a worker that never cached anything, and the binary is the worst place to find that out.

## A prompt that sets this up in one shot

Paste this into Base44. It covers the worker, the local mirror and the queue together, because asking for offline support alone reliably produces a cache-first worker and nothing else. Expect to iterate anyway.

```text
Add offline support to this app. Read https://setup.despia.com/native-features/service-workers first.
This app is shipped as a native iOS and Android build with Despia, and the runtime allows a standard
service worker.

Service worker, always fresh online, cache only as an offline fallback:
- Put the worker at public/sw.js so it is served from the root of the published app, and register it
  with scope '/' and updateViaCache: 'none'. Put offline.html and the web app manifest in public/ too.
- Documents and non-hashed files: network-first with a 3 second timeout and cache: 'no-store' on the
  document request, so no browser or CDN copy shadows a new build. Enable navigation preload.
- Content-hashed build output under /assets/: cache-first, safe because the file name changes on every
  build.
- Offline fallback chain: last cached copy of that page, then the app shell, then /offline.html.
- Do NOT use cache-first for navigation. It pins users to an old build and breaks over-the-air updates.
- Do NOT hand-write a precache list of hashed filenames. Precache only '/' and '/offline.html'.
- Version the cache name, delete every non-matching cache on activate, skipWaiting on install,
  clients.claim on activate.
- Call registration.update() on load, focus, online and visibilitychange, and reload the page on
  controllerchange so a new worker takes effect immediately.
- Do not register the worker in the Base44 editor preview (detect it with window.self !== window.top),
  and unregister any worker found there.
- Add a web app manifest and Apple specific head tags so the app installs as a real PWA.

Local data layer, because the backend is unreachable offline:
- Mirror the entities this app reads into IndexedDB. Read from IndexedDB first and render from it, then
  refresh from Base44 in the background and merge.
- Never block first render on a network call. No await on an entity read or base44.auth.me() before
  something is on screen. Cache the user object locally and render from it when offline.
- Writes are optimistic: update the local row with a _pending flag and queue a job in an IndexedDB outbox.
- Give rows created offline a temporary local id, and when the create lands, delete the temporary row and
  store the server's version so the record does not appear twice.
- Flush the outbox on the window 'online' event and on visibilitychange. Strip local-only fields (_pending,
  _deleted, _localId) before sending. On a 4xx, drop the job, revert the local change and surface an error,
  since a permissions or schema failure never succeeds on retry. On anything else, stop and keep the queue.
- Queue deletes explicitly and write a _deleted tombstone so deleted rows do not come back on the next pull.
- Resolve conflicts on updated_date, newest write wins, except that a pending local row always wins.
- Show a clear pending or offline indicator in the UI.
- Disable online-only features (backend functions, AI calls, payments, email) with an explanation rather
  than letting them fail silently.

Do not add a PWA plugin that generates a cache-first worker.
```

Then enable it on the native side: Settings > Configuration > Dynamic App Source > Offline Support > PWA, bump the version code, and publish a new build.

## Get it on the stores

Take the app you just built and ship it to iOS and Android without a CLI or a Mac. Code signing and submission run from the browser.

[See the setup docs at setup.despia.com](https://setup.despia.com)