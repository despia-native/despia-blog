---
title: "Despia Offline Not Working: Local Server vs PWA"
seoTitle: "Despia Offline Not Working: Local Server vs PWA"
seoDescription: "Despia offline not working after you enabled offline support? Here is why Local Server and PWA both fail, and how to test offline before you ship."
datePublished: 2026-07-03T09:42:26.402Z
cuid: cmr4qvose00000aj8969gcgcs
slug: despia-offline-not-working-local-server-vs-pwa
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/dbb704f1-7829-4777-a6e8-9141d72974af.jpg
tags: webview, webapp, pwa

---

You watched the setup video, filled in every field, pushed a build, and now TestFlight opens to "Unable to connect to server. Please check your internet." Your internet is fine. The build is fine. What happened is that you turned on offline support, skipped the warning the dashboard showed you, and the app is now trying to serve itself from a place that has nothing in it.

Despia has two offline modes, Local Server and PWA, and the thing to understand is that they fail in opposite directions. Local Server fails closed: set it up wrong and the app will not load even when the device is online. PWA fails open: set it up wrong and the app loads fine online, then drops to a default offline screen the moment the network is gone. This post is how to tell them apart, which one you actually want, and the one test that tells you the truth before you ship.

## The two offline modes, and what each one really does

Under Despia > Settings > Offline Support you get two choices. They are not interchangeable, and picking the wrong one for your stack is the most common reason a first build won't load.

**Local Server (Native)** copies your web app onto the device's native file system and serves it from there. The WebView loads assets off local storage instead of hitting your origin. This is a real offline-first setup: the app boots with no network at all, because everything it needs is already on the phone.

**PWA** leaves your app hosted where it already is and relies on a service worker you wrote to cache assets in the browser and serve them when the network is gone. Despia does not write the service worker for you. Your web app owns that logic, exactly as it would in Safari or Chrome.

The difference that trips people up is not just what each mode needs, it is how each one behaves when it is set up wrong.

PWA fails open. If the service worker is missing or broken, the app still loads normally as long as the device is online, because Despia is serving it straight from your live origin. It only falls over when you go offline with nothing cached, and then Despia shows its default offline screen, the one you can style in the Editor under Settings. That is the trap: online, everything looks correct, so you assume offline is handled. It is not, and you only find out with both radios off, staring at the default screen and wondering why your app will not load.

Local Server fails closed. Once it is enabled, Despia always loads the app from the local cache built off the manifest, online or not. It does not fall back to your live https site when the manifest will not resolve, because in this mode the app is meant to run entirely from the device. So a Local Server that is not set up correctly does not load at all, even on a full network. That is why you can be sitting on Wi-Fi and still get "Unable to connect to server."

## Despia Local Server not working: the missing manifest

When you enable Local Server, Despia looks for a manifest at `/despia/local.json` on your origin. That file lists the assets to pull onto the device. If that URL does not return a real Despia manifest, there is nothing to bundle, and the local server has nothing to serve. Boot, empty, error.

The failure almost always looks like this. Open the URL directly:

```plaintext
https://your-app.com/despia/local.json
```

If you get your app's client-side 404 page back instead of JSON, that is the whole problem. Your SPA router caught the request, saw a route it didn't recognise, and served the catch-all 404 view. Despia asked for a manifest and got an HTML error page. The dashboard warned you about exactly this before you enabled it, and it is easy to click past.

Here is the part people miss: you do not write that manifest by hand. It is generated at build time by the `@despia/local` plugin, which scans your output directory and writes `despia/local.json` with the list of assets. A 404 at that URL almost always means the plugin was never added to your build, so the file does not exist to serve.

Install it:

```shellscript
npm install --save-dev @despia/local
```

Then wire it into your build. Vite:

```javascript
// vite.config.js
import { defineConfig } from 'vite';
import { despiaLocalPlugin } from '@despia/local/vite';

export default defineConfig({
  plugins: [despiaLocalPlugin()]
});
```

There are official plugins for Webpack, Rollup, Nuxt, SvelteKit, Astro, Remix, and esbuild, plus a universal CLI you can drop into a `postbuild` script if your build tool is not in the list:

```json
{
  "scripts": {
    "build": "your-build-command",
    "postbuild": "despia-local"
  }
}
```

Rebuild and redeploy your web app, then confirm `/despia/local.json` returns JSON with an `entry`, a `deployed_at` timestamp, and an `assets` array. You do not need a new Despia build for this. Despia checks the manifest on launch, so once the correct file is live, close the app and open it again and it resolves. Keep the manifest at exactly `despia/local.json`. Despia looks for it at that path to detect and apply updates, so do not move or rename it. If the URL still returns your SPA shell after adding the plugin, your host is rewriting unknown paths to `index.html` before the static file is served, and you need to let that one path through untouched.

## Local Server runs over HTTP, and that has consequences

Local Server serves from the native file system over `http://`, not `https://`. For most apps that is invisible. But anything that hard-requires a secure origin will refuse to run. Sign in with Apple and other Apple JS flows are the usual casualty: they check for a secure context and bail on HTTP.

The trade you get for that limitation is the native file system, which comes with much larger storage limits than a browser cache. For a media-heavy app that wants to hold a lot of assets locally, that headroom matters.

So Local Server is a good fit when:

*   You built a client-rendered SPA (React with Vite is the common case).
    
*   You want true zero-network boot.
    
*   You are not depending on secure-context-only browser APIs.
    

If any of those does not hold, PWA is the better path.

## Local Server and SSR: it is the rendering mode, not the framework

People assume Local Server means React with Vite only. It does not. The `@despia/local` plugin supports Vite, Webpack (so Create React App, Vue CLI, Angular), Rollup, Nuxt, SvelteKit, Astro, Remix, and esbuild, plus the universal CLI. What actually matters is not the framework name, it is what your build produces.

Local Server serves static files off the device. So the rule is about rendering mode:

*   If your build outputs static assets to a directory (a client-rendered SPA, or a prerendered or statically exported site), the plugin scans that directory, writes the manifest, and Local Server works. This is true even for Nuxt, SvelteKit, Astro, and Remix when you build them to static output.
    
*   If your app renders every page on a server, per request, at runtime, there is nothing static to put on the device and no server on the phone to render it. That app cannot use Local Server.
    

A default Next.js app in full server mode is the clearest case that will not work. Prerender or statically export it, or switch to PWA. When you genuinely need runtime server rendering, PWA is the offline route that fits, because it keeps your app on its real origin and lets a service worker handle the offline shell.

## Despia PWA offline not working: the service worker is yours

Select PWA in Despia's offline settings and the offline behaviour becomes your service worker's job. A service worker is client-side code. Offline handling has to be client-side by nature: the whole point is that there is no connection to the server when it runs, so the logic that decides what to serve has to already be sitting on the device, in the browser, inside the app's own web logic.

Two things break service workers constantly, and both show up more when the code was generated rather than written:

1.  **It was never set up correctly.** The worker registers but caches nothing useful, or scopes the cache wrong, so the first offline load has no shell to fall back to.
    
2.  **It never clears its cache when online.** The worker keeps serving an old cached version even when the device has a network. That is the failure that quietly wrecks your Despia OTA updates, covered below.
    

A service worker that looks plausible in code review can still fail both of these. The only way to know is to test it, and there is exactly one test that counts.

## The one test that tells you the truth

Do not test offline handling inside your Despia app first. Test it as a plain PWA in the phone's browser, because that is the control group. If the service worker does not work there, it will not work in Despia either, and testing in Despia first just hides which layer is broken.

The procedure:

1.  Install your web app as a PWA on your phone. Not the Despia build. The web app, added to the home screen from the browser.
    
2.  Open it with both Wi-Fi and cellular on. Let it sit for about 10 seconds so the service worker installs and caches.
    
3.  Turn Wi-Fi **and** cellular off. Both of them, manually.
    
4.  Open the PWA again.
    

If you see a "page cannot be found" or connection error, your service worker is not set up correctly. Full stop. It will not magically start working once it is wrapped in Despia. Fix the worker until the PWA opens offline, and only then rely on it in your Despia app. Since Despia loads the same origin, a corrected worker shipped to your site is picked up on the next launch, no new build required.

One trap in the test itself: do not use airplane mode. Airplane mode can leave Wi-Fi on, so the device is still online and you are testing nothing. Turn the two radios off by hand.

The PWA opening cleanly with both radios off is the only real measure of whether offline works. If the control group passes, Despia will pass. If the control group fails, no Despia setting will save it.

## The OTA trap: serving stale cache breaks your updates

Despia ships web content updates over the air through remote hydration, so you push a fix and users get it without an App Store resubmission. A service worker that caches aggressively and never revalidates when online will hand your users the old bundle forever, and your OTA updates stop reaching them.

The rule is simple and non-negotiable: when the device is online, your service worker must clear the old cache and re-cache the new version. If it keeps serving the old cache while a network is available, you have broken OTA for everyone running that worker. This is the single most common way a working offline setup quietly kills update delivery.

You want the worker to serve cache offline, and refresh cache online. Not one or the other.

Local Server handles this differently, and in your favour. The plugin stamps a fresh `deployed_at` timestamp into the manifest on every build. Despia reads that timestamp, notices the build changed, and applies the update atomically. So for Local Server, OTA is not something you hand-code, it is something you get as long as you rebuild your web app and redeploy so a new manifest ships. No new Despia binary, no store round trip. The stale-cache trap is a PWA problem, because on PWA the caching logic is yours.

## Which one to pick

For roughly 99% of apps, PWA offline support is the right call. It is simpler, it stays on `https://` so secure-context APIs keep working, and it fits the stacks people actually build in. It works well with TanStack, Next.js, and anything server-rendered or hybrid.

Reach for Local Server when you specifically want true zero-network boot on a client-rendered SPA, you can live without HTTPS-only browser APIs, and you want the larger native storage limits for a lot of local assets.

|  | Local Server | PWA |
| --- | --- | --- |
| How it serves | From device file system | From your origin, cached by your service worker |
| Protocol | HTTP | HTTPS |
| Rendering mode | Static, prerendered, or client-rendered build | Any, including runtime SSR |
| Secure-context APIs (Apple JS) | No | Yes |
| Storage headroom | Large, native file system | Browser cache limits |
| Who owns the offline logic | Despia manifest at `/despia/local.json` | Your service worker |
| Best fit | Client-rendered SPA, zero-network boot | Almost everything else |

Whichever you pick, the environment check inside your app stays the same:

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')
if (isDespia) despia('successhaptic://')
```

## Add offline support to your app

Offline support in Despia is a setting, but the thing that makes it actually work is on your side. For Local Server, that means the `@despia/local` plugin in your build so `/despia/local.json` is a real manifest and not a 404. For PWA, it means a service worker that passes the plain-PWA offline test. Get that green first. Neither one needs a fresh Despia build to take effect: redeploy the corrected manifest or worker to your origin, reopen the app, and it resolves.

[Read the full docs at setup.despia.com](https://setup.despia.com)