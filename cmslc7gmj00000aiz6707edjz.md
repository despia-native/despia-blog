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

Most people try this once, in the wrong order. They flip Offline Support to PWA in Despia, rebuild, put the phone in airplane mode, and the app opens to the native network error page. Or it opens, shows the shell, and then spins forever on an empty screen.

Both failures have the same root cause: caching the app and running the app offline are two different problems. A service worker keeps your HTML, JavaScript and CSS available without a network. It does nothing at all for the Lovable Cloud queries, edge functions and third party APIs your app calls the moment it boots. If you only solve the first half, you have shipped an app that loads offline and then immediately fails offline.

This is the full path: which Lovable stack you are on, the worker itself, the local data layer that makes the cached app actually usable, and the failure modes that eat the most time.

## What the Despia offline toggle actually does

The setting lives in the Despia Editor under App > Settings > Offline Support, and it has one job: allow your web app's service worker to register, install and serve fetch events inside the native runtime, with cache storage available to it. Set the mode to PWA, bump the version code, and trigger a fresh build.

It does not write a service worker for you. If your Lovable app does not ship one, or ships one that never caches anything, the toggle changes nothing and every offline launch fails exactly as before. Despia removes the native obstacle. The caching strategy is yours, in your web codebase, using standard web APIs.

The other thing worth saying out loud, because it costs people an afternoon: offline support is a native runtime setting, so it only exists in binaries produced after the toggle was flipped. If you enable it and keep testing the build already on your phone, service worker registration is silently ignored, caches stay empty, and the Editor still reads as on. Install the new build before you conclude anything.

## Which Lovable stack are you on, because the answer forks here

TanStack Start became the default for new Lovable projects on May 13, 2026, and for new Enterprise projects on June 22, 2026. Anything created before that is the older React plus Vite stack, a client-rendered single page app deployed as static files. The stack is locked at creation and a prompt will not change it, so this is a fact about your project, not a choice you get to make now.

Check your exported code for `vite.config.ts` and a `public/` directory, or open your published URL with JavaScript disabled and see whether content arrives in the HTML.

On the Vite stack, life is simple. The build produces static files, `public/sw.js` is served from the root, and one document plus a folder of hashed assets covers the whole app. Every route resolves to the same `index.html`, so precaching the shell precaches every screen.

On the TanStack Start stack, each route is rendered into HTML on the server, so a cached document is a snapshot of a server render taken at the moment it was cached. That is fine for a dashboard or a settings page. It is wrong for anything showing live data, because the reader gets a page that looks current and is not. Cache route documents deliberately, one at a time, and only where a stale first paint is acceptable and the client refetches on load. If your app is data-heavy on every route, the cleaner answer is a client-rendered build on a mobile subdomain with Despia's start URL pointed at it, so the offline path is a single shell you fully control.

## The worker: network-first for documents, cache-first for hashed assets

Registration is ordinary. The file must be served over HTTPS from a scope that covers every route the app can open, which in practice means the root.

```javascript
if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js', { scope: '/' })
}
```

The strategy is where posts on this subject usually go wrong. Cache-first on navigation requests is the default in most tutorials and in most generated workers, and it is the single worst choice for an app that updates over the air. A cache-first worker answers every navigation from its stored copy, so your users stay pinned to whatever version was current the first time they opened the app. You publish in Lovable, the content updates on your host, and nobody sees it.

Network-first with a timeout is the correct default for documents. The timeout matters more than it looks: a hanging connection does not reject `fetch()`, it just sits there, so a plain `fetch().catch()` fallback stalls on a bad train connection instead of falling back.

```javascript
const CACHE = 'app-v3'
const DOC_TIMEOUT = 3000

const PRECACHE = ['/', '/offline.html']

self.addEventListener('install', event => {
    self.skipWaiting()
    event.waitUntil(caches.open(CACHE).then(cache => cache.addAll(PRECACHE)))
})

self.addEventListener('activate', event => {
    event.waitUntil(
        caches.keys()
            .then(keys => Promise.all(
                keys.filter(key => key !== CACHE).map(key => caches.delete(key))
            ))
            .then(() => self.clients.claim())
    )
})

self.addEventListener('fetch', event => {
    const req = event.request
    if (req.method !== 'GET') return

    const url = new URL(req.url)
    if (url.origin !== self.location.origin) return

    // hashed build output is immutable, everything else goes through the network first
    const immutable = url.pathname.startsWith('/assets/')
    event.respondWith(immutable ? cacheFirst(req) : networkFirst(req))
})

async function networkFirst(req) {
    const cache = await caches.open(CACHE)
    try {
        const res = await withTimeout(fetch(req), DOC_TIMEOUT)
        cache.put(req, res.clone())
        return res
    } catch {
        return (await cache.match(req))
            || (await cache.match('/offline.html'))
            || Response.error()
    }
}

async function cacheFirst(req) {
    const cache = await caches.open(CACHE)
    const hit = await cache.match(req)
    if (hit) return hit

    const res = await fetch(req)
    if (res.ok) cache.put(req, res.clone())
    return res
}

function withTimeout(promise, ms) {
    return Promise.race([
        promise,
        new Promise((_, reject) => setTimeout(() => reject(new Error('timeout')), ms))
    ])
}
```

Change `CACHE` on every release. The activate handler deletes every cache that is not the current one, and `skipWaiting` plus `clients.claim` hands control to the new worker immediately rather than waiting for every tab to close.

Do not hand-write a precache list of asset filenames. Vite hashes them on every build, so a list written today points at files that will not exist next week, `addAll` rejects, install fails, and the worker never activates. Cache assets as they are visited, which the code above does, or use `vite-plugin-pwa` in `injectManifest` mode so the list is generated at build time from the real output. On the Vite stack `injectManifest` is the better answer, because you keep the fetch handler above and let the tooling own the file list.

## Offline means your database is gone, and that is the part people miss

Here is the thing that turns a working service worker into a broken app. Your cached shell boots. It renders. Then it does what Lovable wrote it to do, which is call your backend.

Lovable Cloud is a managed Supabase instance, and connecting your own Supabase project is the alternative. Either way, every `supabase.from(...)` read is an HTTPS request to a remote host. So is every edge function, every AI call, every Stripe request, every webhook-backed integration. With no network there is no server to talk to, and a cached page that renders a spinner until a query resolves will spin until the request times out and then show an error state, if you wrote one.

That is the real work of offline support: a local copy of the data the user needs, and a queue for the changes they make while disconnected.

| What the user does offline | Works from cache alone | Needs a local data layer |
| --- | --- | --- |
| Opens the app, sees the shell and navigates | Yes | No |
| Loads JS, CSS and previously visited images | Yes | No |
| Reads their existing records | No | Yes, mirrored into IndexedDB |
| Creates or edits a record | No | Yes, optimistic write plus outbox |
| Runs an edge function, AI call or payment | No | No, defer and tell the user |
| Stays signed in | Session persists locally | Token refresh needs network |
| Receives a realtime update | No | No, resync on reconnect |

Read from local storage first, always, even when online. Render from it. Then refresh from the network in the background and write the fresh rows back. This is the pattern that makes the app feel fast on a good connection and keeps working on none, and it is one code path instead of two.

Use IndexedDB for records. `localStorage` is fine for a token and a small user object, and wrong for lists: it is synchronous, string-only, and small enough that a few hundred rows will hit the wall. Keep authenticated responses out of the Cache API too. That cache is keyed by URL with no notion of who is signed in, and cross-origin backend responses do not belong there. Mirror the rows into IndexedDB instead, where you control the shape and the lifetime.

## The write path: optimistic locally, queued, flushed on reconnect

Every offline write is two operations. Update the local copy so the UI responds immediately, and record the intent so it can be replayed against the server later.

```javascript
const DB = 'app'

function idb() {
    return new Promise((resolve, reject) => {
        const req = indexedDB.open(DB, 1)
        req.onupgradeneeded = () => {
            const db = req.result
            db.createObjectStore('tasks', { keyPath: 'id' })
            db.createObjectStore('outbox', { keyPath: 'jobId' })
        }
        req.onsuccess = () => resolve(req.result)
        req.onerror = () => reject(req.error)
    })
}

async function tx(store, mode, fn) {
    const db = await idb()
    return new Promise((resolve, reject) => {
        const t = db.transaction(store, mode)
        const req = fn(t.objectStore(store))
        t.oncomplete = () => resolve(req && req.result)
        t.onerror = () => reject(t.error)
    })
}

async function saveTask(task) {
    const row = { ...task, updated_at: new Date().toISOString(), pending: true }

    await tx('tasks', 'readwrite', s => s.put(row))
    await tx('outbox', 'readwrite', s => s.put({
        jobId: crypto.randomUUID(),
        table: 'tasks',
        row
    }))

    flush()
    return row
}

async function flush() {
    if (!navigator.onLine) return

    const jobs = await tx('outbox', 'readonly', s => s.getAll())

    for (const job of jobs) {
        const { pending, ...payload } = job.row
        const { error } = await supabase.from(job.table).upsert(payload)

        // network failures stay queued, rejections do not
        if (error && !error.message.includes('Failed to fetch')) {
            await tx('outbox', 'readwrite', s => s.delete(job.jobId))
            reportRejected(job, error)
            continue
        }
        if (error) return

        await tx(job.table, 'readwrite', s => s.put({ ...job.row, pending: false }))
        await tx('outbox', 'readwrite', s => s.delete(job.jobId))
    }
}

window.addEventListener('online', flush)
document.addEventListener('visibilitychange', () => {
    if (!document.hidden) flush()
})
```

Four details in there are the difference between a sync layer and a data loss bug.

Strip your local-only fields before the upsert, or Postgres rejects the write on an unknown column and you will spend an hour blaming the queue. Separate a transport failure from a server rejection: a dropped connection should keep the job queued, but a row your policies refuse is never going to succeed, so drop it and tell the user rather than retrying forever in a loop. Row Level Security still applies at flush time, which means an offline write is a request, not a guarantee, and the UI should say pending until the server has accepted it. And flush on visibility change as well as the `online` event, because a phone that was asleep in a lift comes back through resume, not always through a connectivity event.

Deletes need their own treatment. A row removed locally and simply absent from the mirror will reappear on the next refresh from the server, so queue the delete as an explicit job and keep a tombstone locally until it flushes. For conflicts, comparing `updated_at` and letting the newer write win is enough for most single-user apps. If two people edit the same record, decide per field or keep a server-side revision counter, but decide it deliberately rather than discovering the rule from behaviour.

## Auth, and why the app can look signed out when it is not

The Supabase client persists the session in local storage by default, so a cold start with no network still has a valid-looking session to read. The app appears signed out anyway when first render is blocked on a call that cannot complete: a session refresh, a profile fetch, a role lookup. The await never resolves, the loading state never clears, and the user sees a spinner or a login screen.

Read the stored session synchronously, render from it, then revalidate when the network returns.

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

Treat that cached session as a rendering hint, not as authorization. Anything sensitive is still verified server side on the next successful request, and the local mirror should only ever contain data that user was already allowed to read.

If you want the offline state to look different inside the native app than in a browser tab, the environment check is the user agent:

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')
```

## Test it as a home screen PWA before you build the native app

Do this first, every time. Service workers, the Cache API and IndexedDB are web platform APIs. The Despia runtime runs the same engine your browser runs, so it does not add behaviour to them and it cannot repair them. If your offline layer works in an installed PWA it will work in the native build. If it fails in the PWA, it will fail in the native build in exactly the same way, and the only thing that changes is that the error is now hidden behind a native network error screen and looks like a runtime bug.

That is where most reports of broken offline support come from. The worker never cached anything, the app fails in the binary, and the conclusion is that Despia is broken. Install the same URL as a PWA and the identical failure shows up with a browser error message attached, which tells you what actually went wrong.

1.  Deploy. Use your published Lovable URL, not the editor preview.
    
2.  Open that URL in Safari on iOS or Chrome on Android, on a physical device, and add it to the home screen.
    
3.  Launch the installed app and leave it open for about ten seconds so the worker can install, activate and populate its caches.
    
4.  Use it online first. Load the screens that matter, create a record, edit a record. This is what fills the caches and writes rows into IndexedDB. A worker with an empty cache is indistinguishable from no worker at all.
    
5.  Confirm the caches exist. On Android, connect the device and open `chrome://inspect` on the desktop, then look at Application > Cache Storage and Application > IndexedDB. On iOS, enable Web Inspector in Settings > Safari > Advanced and attach Safari on a Mac. If those panels are empty, stop here, because nothing downstream will work.
    
6.  Turn off Wi-Fi and cellular separately in system settings. Not airplane mode, since some devices keep Wi-Fi alive in it and the test then passes for the wrong reason.
    
7.  Force quit the installed PWA and reopen it cold. A warm relaunch proves nothing, because the page was never torn down.
    
8.  Read the result. Content on screen means the worker is serving. A browser-level error such as Safari reporting no internet connection means the worker is not serving cached content, and that is a web app problem.
    
9.  Read your own data. Navigate to the list of records. If the shell renders and the list spins or errors, the worker is fine and your local data layer is missing, which is the failure this post exists for.
    
10.  Write while offline. Create or edit something and confirm it appears immediately with a pending state.
     
11.  Turn the radios back on and reopen the app. The queued write should reach the backend, and the record should stop being pending. If it does not, the outbox is not flushing, and it will not flush in the native build either.
     

Only once steps 8 through 11 pass do you flip Offline Support to PWA in the Despia Editor, bump the version code and rebuild. At that point the native build is just the same web app in a native shell with the same caches, and offline behaviour carries over unchanged.

The one thing that genuinely differs is the very first launch. A hosted app has to be opened once with a working connection before the worker can install and precache anything. A user who installs from the store on a plane and opens the app for the first time offline has no cache to serve.

## The pitfalls that generate the most support tickets

**A stale worker that outlives your fix.** This is the big one. A cache-first worker shipped by a PWA plugin keeps serving the old app to everyone who already installed it, including after you fix the worker, because the fix itself is a file the old worker is caching. Ship a kill switch, deploy it with no-cache headers on the worker path, and let it run once.

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

**A cached document pointing at deleted assets.** If the HTML is cached and the hashed JavaScript it references is not, the app opens to a white page. Network-first on documents plus a versioned cache prevents it, and `/offline.html` is what stands between a dropped request and a blank screen, so make it render something useful.

**Testing in the Lovable editor preview.** The preview is not your deployment. Service worker behaviour, scope and headers only mean anything on the published URL that Despia actually loads.

**Debugging the native build first.** Almost every reported offline failure is a worker that never cached anything, and the binary is the worst place to find that out. Run the home screen PWA test above, where the same failure comes with a readable error and a working inspector.

## A prompt that sets this up in one shot

Paste this into Lovable. It covers the worker, the local mirror and the queue together, which matters, because asking for offline support alone reliably produces a cache-first worker and nothing else.

```text
Add offline support to this app. Read https://setup.despia.com/native-features/service-workers first,
since this app is shipped as a native iOS and Android build with Despia and the runtime allows a
standard service worker.

Service worker:
- Create a service worker served from the root of the site, registered with scope '/'.
- Documents and navigation requests: network-first with a 3 second timeout, falling back to the cache,
  then to /offline.html. Do not use cache-first for navigation, it pins users to an old build.
- Hashed build assets under /assets/: cache-first.
- Do not hand-write a precache list of hashed filenames. Precache only '/' and '/offline.html',
  and cache everything else at runtime as it is visited.
- Name the cache with a version constant, and delete every non-matching cache on activate.
  Call skipWaiting on install and clients.claim on activate.
- Create an /offline.html that renders the app shell and a short message.

Local data layer, because the database is unreachable offline:
- Mirror the tables this app reads into IndexedDB. Read from IndexedDB first and render from it,
  then refresh from the backend in the background and write the fresh rows back.
- Never block first render on a network call. No await on a query, session refresh or profile fetch
  before something is on screen.
- Writes are optimistic: update the local row with a pending flag, and add a job to an IndexedDB
  outbox store.
- Flush the outbox on the window 'online' event and on visibilitychange when the document becomes
  visible. Strip local-only fields such as pending before sending. Keep a job queued on a network
  failure, drop it and surface an error on a server rejection, since Row Level Security failures will
  never succeed on retry.
- Queue deletes explicitly with a local tombstone so deleted rows do not come back on the next sync.
- Resolve conflicts by comparing updated_at and keeping the newer write.
- Show a clear pending or offline indicator in the UI for anything not yet synced.
- Keep the auth session read from local storage and render from it when offline. Revalidate when the
  connection returns. Do not treat it as authorization.
- Features that cannot work offline (AI calls, payments, edge functions, email) should be disabled
  with an explanation rather than failing silently.

Do not add a PWA plugin that generates a cache-first worker.
```

Then enable it on the native side: App > Settings > Offline Support > PWA in the Despia Editor, bump the version code, rebuild, and install the new binary before testing.

## Get it on the stores

Take the app you just built and ship it to iOS and Android without a CLI or a Mac. Code signing and submission run from the browser.

[See the setup docs at setup.despia.com](https://setup.despia.com)