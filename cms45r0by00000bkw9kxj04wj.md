---
title: "window.despia Is Undefined: What to Call Instead"
seoTitle: "window.despia Is Undefined: What to Call Instead"
seoDescription: "window.despia is undefined in your app because it is not a public API. Here is what the Despia SDK exposes and how to call native features correctly."
datePublished: 2026-07-28T04:30:38.478Z
cuid: cms45r0by00000bkw9kxj04wj
slug: window-despia-is-undefined-what-to-call-instead
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/88a3c187-ae1c-45fd-a370-36035d2ed2a0.png
tags: cursor, lovable, vibecoding, claude-code, base44

---

Your purchase button works in the browser. You install the TestFlight build, tap it on a real device, and nothing happens. You add a debug alert and find the reason: `window.despia` is `undefined`. The environment check passed, the user is signed in, the integration is configured in the dashboard, and the object your code is reaching for simply is not there.

It is not there because it was never a public API. The code calling `window.despia(...)` was almost certainly written by an AI assistant that guessed at the shape of the SDK, and the guess is wrong in a way that fails silently in the browser and loudly on the device.

## The short version

*   `window.despia` is an internal proxy, not a function. You assign to it, you never call it. `window.despia('x://')` throws.
    
*   It is a legacy path kept alive for backwards compatibility. Assigning to it still works and still will. It is not a supported API and it carries no stability guarantee.
    
*   Reading `window.despia` tells you nothing about whether the runtime is present, so it is useless as a feature detect.
    
*   Detect the runtime with the user agent: `navigator.userAgent.toLowerCase().includes('despia')`.
    
*   Call native features with the `despia()` function from the `despia-native` package, or from the CDN build if you do not bundle.
    
*   If your assistant keeps inventing APIs, point it at the Despia MCP server at `https://setup.despia.com/mcp` so it reads the real reference instead of guessing.
    

## What is window.despia, and why is it undefined?

`window.despia` is a proxy. Assigning to it updates `window.virtual.href`, a virtualized href that the native layer watches. It takes the same scheme strings a real navigation would, without the length ceiling a real URL redirect imposes, so payloads that would blow past a URL limit still go through intact. The native layer picks the assignment up on both platforms, runs the Swift or Java implementation, and writes results back into the page as named window variables, which the SDK resolves into a promise.

So `window.despia` does exist inside the machinery. You assign a string to it and the runtime picks the assignment up:

```javascript
window.despia = 'successhaptic://'
```

That is the entire interface. It is a property you write to, not a function you call, and it dates from an older generation of the runtime that predates the SDK. The runtime still honours it, and will keep honouring it, because backwards compatibility on that path has been maintained since the runtime was first built and every app that ever shipped against it still works. That is a compatibility promise, not an invitation. It is an internal proxy with no stability guarantee, and it should not appear anywhere in your application code.

The scheme pattern itself is not a leftover, and it is worth separating the two. `scheme://` interception was the standard way hybrid apps reached native code in 2011, which is when this runtime was first built, and it survived because it turned out to be the right shape: a flat command string in, a named result out, trivial to read, trivial to grep, and consistent across iOS and Android. It also turned out to be the shape AI coding tools generate most reliably, because a flat structured string has no nested config object to malform. What is legacy is the raw property, not the command format.

Two consequences follow, and both of them look like the runtime is broken when it is not.

First, `window.despia(...)` throws `window.despia is not a function`, because a property holding a string is not callable. The generated code got the transport right and the shape wrong. Second, and more confusing, reading `window.despia` before anything has been written to it returns `undefined`. A perfectly healthy build, with the runtime present and every integration configured, reports `undefined` on that property at startup. Any guard written as `if (window.despia)` fails closed on every launch and your native path never runs.

That is the whole failure. The build is fine. The check is wrong.

## How do you detect the Despia runtime?

With the user agent. The string `despia` is present in the user agent whenever your web app is running inside the runtime, on both platforms, from the first byte of the first page load, before any SDK code has executed.

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

const isDespiaIOS = isDespia && (
    navigator.userAgent.toLowerCase().includes('iphone') ||
    navigator.userAgent.toLowerCase().includes('ipad')
)

const isDespiaAndroid = isDespia && navigator.userAgent.toLowerCase().includes('android')
```

This is the only supported environment check. It costs nothing, it works before the SDK loads, and it is stable across every build of your app because the runtime controls the user agent, not your web bundle.

Native calls go inside that gate. Global callbacks that the runtime invokes are assigned outside it, at module scope, so they are defined before any native event can fire.

## How do you call a native feature?

Through the `despia()` function. Install the package and import the default export.

```bash
npm install despia-native
```

```javascript
import despia from 'despia-native'

despia('successhaptic://')

const device = await despia('get-uuid://', ['uuid'])
console.log(device.uuid)
```

If your app is not built with a bundler, load the CDN build instead and use the global the script defines.

```html
<script src="https://cdn.jsdelivr.net/npm/despia-native/index.min.js"></script>
```

The second argument is the list of window variables to watch. `despia()` returns a promise that resolves once the native layer has written those variables back, with a 30 second timeout. Calls that return nothing, like haptics, take no watch array and need no `await`.

There is no initialization step, no client object to construct, no provider to wrap your app in. If your generated code has any of those, it was invented.

## Why go through a package at all?

It is a fair question. The internal path works, it has no install step, and writing one line to a property is less machinery than a dependency. The answer is that the package is the part that stays stable while the thing underneath it moves.

Native platforms change. A capability gets a better implementation, a scheme gets replaced by a more secure one, an API gets a stricter shape. When your app calls `despia('feature123://')` through the package, a rename or a redesign underneath is a proxy inside the package, and your code keeps working without you touching it. When your app writes the raw string to the internal property, your app is pinned to that string forever and every change becomes your migration.

That is the trade. The internal path is frozen by the compatibility promise, which sounds like safety and is actually the opposite: frozen means it cannot be improved under you, so nothing improves for you either. The package is where deprecations get absorbed, where new capabilities arrive, and where the API can evolve without a coordinated rewrite across every app that ships on the runtime.

That evolution is already underway. A type-safe option is coming to the package for developers who want typed calls and editor completion instead of strings, added carefully and gradually rather than as a cutover. The scheme API is not being deprecated and is not going anywhere, so nothing you have written needs changing. The point is where you get it: apps calling through the package get the typed surface as an update, and apps writing raw strings to the internal property get nothing, because there is no package in the path to hand it to them.

## Why do AI coding tools invent Despia APIs?

Because the shape they invent is the shape almost every other SDK has. A JavaScript library that talks to a native layer usually exposes an object on `window`, and a model that has seen ten thousand of those will produce an eleventh when it has not read the actual reference. `window.despia.purchase()`, `window.Despia.isAvailable()`, `despia.init({ apiKey })`, and `window.despia(...)` are all plausible, all consistent with the ecosystem, and all fictional.

The tell is that hallucinated APIs are usually more elaborate than the real one. Despia's surface is a single function that takes a string. When generated code contains a config object, a lifecycle hook, or a namespaced method chain, that is the signal to check it against the docs.

Two things fix this properly.

Point your assistant at the MCP server. It is a documentation server, and it hands Cursor, Claude Code, VS Code, Windsurf, Lovable, Base44, Bolt and any other MCP-capable tool the actual `despia-native` reference, so generation happens against the real API instead of against the model's priors.

```plaintext
https://setup.despia.com/mcp
```

There is no authentication and nothing to restructure. Per-tool setup paths, and a prompt for verifying that your agent is actually reading the server rather than just showing it as connected, are in [the MCP server post](https://blog.despia.com/despia-mcp-server-stop-burning-credits-on-bad-code).

If your tool has no MCP support, paste the relevant feature page from the docs into the chat before asking for the code. Both approaches beat correcting the output afterwards, because a wrong native call does not fail in preview. It fails on a device, in a build, days later.

## Four errors and what each one means

| What you see | What it means | Fix |
| --- | --- | --- |
| `window.despia is not a function` | Generated code is calling an internal write channel as if it were an API | Call the imported `despia()` function instead |
| `window.despia` is `undefined` in a guard | Feature detect written against the wrong thing | Detect with the user agent |
| `despia is not defined` | The SDK was never imported, or the CDN script did not load | Import the default export, or add the script tag |
| Call runs, nothing happens on device | Wrong scheme name, or the integration is off in the dashboard | Check the scheme against the reference, then check the integration toggle |

The last row is the one that costs the most time, because nothing in the web layer reports it.

## The call runs and nothing happens

Take purchases. The runtime ships a RevenueCat integration, and the web layer configures no billing SDK at all, so there is exactly one place the configuration can be wrong: RevenueCat has to be enabled in Despia under App, Settings, Integrations, RevenueCat, with your API key set, before any purchase call does anything.

With that in place, launching a paywall is one call, gated on the runtime, with a web fallback for the same button in a browser.

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

if (isDespia) {
    despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)
} else {
    window.location.href = `https://pay.rev.cat/<your_token>/${encodeURIComponent(userId)}`
}
```

Entitlements are read back through the runtime, and the purchase callback is assigned at module scope so it exists before the native layer fires it.

```javascript
async function checkEntitlements() {
    const data   = await despia('getpurchasehistory://', ['restoredData'])
    const active = (data.restoredData ?? []).filter(p => p.isActive)
    if (active.some(p => p.entitlementId === 'premium')) unlockPremium()
}

checkEntitlements()
window.onRevenueCatPurchase = checkEntitlements
```

Note what is absent. No billing client, no product catalogue fetch in JavaScript, no store initialization. If your generated purchase code has those, it is written against a different SDK.

## How to debug this in a TestFlight build, step by step

There is no console on a device build, so put the state on screen. This is four lines and it answers the question in one tap.

```javascript
// on-device state dump, remove before shipping
document.body.insertAdjacentHTML('beforeend', `
  <pre style="position:fixed;bottom:0;left:0;z-index:99999;background:#fff;font-size:11px">
ua:  ${navigator.userAgent}
sdk: ${typeof despia}
  </pre>`)
```

`typeof` on an undeclared identifier does not throw, which is why `typeof despia` is safe here and `despia` on its own is not.

Read it in order.

1.  If the user agent does not contain `despia`, you are not in the runtime. You are in Safari, or in a preview pane, or the app loaded a different URL than you think.
    
2.  If the user agent is right and `sdk` says `undefined`, the SDK is missing from the bundle the device actually loaded. Web content ships over the air, so the build in TestFlight may be serving an older or a different deploy than your last push.
    
3.  If the user agent is right and `sdk` says `function`, the runtime and SDK are both fine. Everything after this point is a scheme name or an integration setting, not a missing object.
    
4.  At that stage, call something with no dependencies, like `despia('successhaptic://')`. A phone that buzzes proves the whole path end to end in one second.
    

Most tickets that start with a missing object end at step three.

## FAQ

**Is** `window.despia` **a real thing or not?** It is real, it is internal, and it is a leftover. Assigning a scheme string to it still reaches the native layer, and that will keep being true. Calling it as a function never worked. Neither belongs in your code: it is an internal proxy, so treat a generated `window.despia` line as a bug regardless of which form it takes.

**If assigning to it works, why not just do that?** Because your app would be pinned to the exact string forever. Going through the package means a capability can be replaced or hardened underneath you and the package proxies your existing call to the new implementation. Writing the raw string opts you out of that and makes every future change your problem.

**It worked in an earlier build. What changed?** Almost never the binary. The runtime does not add or remove the SDK between builds, because the SDK lives in your web bundle, and your web bundle ships over the air. A working-then-broken native call usually means a web deploy changed, an import got dropped, or the app is loading a different hosting URL than you expect.

**Could an App Store Connect issue cause this?** No. Product state in App Store Connect, an in-app purchase stuck in review, or a developer-rejected submission cannot affect whether a JavaScript function exists in your page. Those are billing-side problems and they surface as a paywall that opens with no products, not as a missing object. When both happen in the same week they are two unrelated problems.

**Should I use** `typeof despia === 'function'` **as my environment check?** No. That tells you the SDK is loaded, which is a bundling fact, not a runtime fact. It is true in a normal browser too. Gate on the user agent and use the `typeof` check only when debugging.

**Do I need to await every call?** Only calls that return data. `despia('successhaptic://')` fires and returns. `await despia('get-uuid://', ['uuid'])` waits for the named variable to come back.

**What about callbacks like** `window.onRevenueCatPurchase`**?** Assign them at module scope, outside the environment gate. The runtime invokes them, so they need to exist before the event, and defining them in a browser costs nothing because nothing will ever call them there.

**My assistant keeps regenerating the wrong call after I fix it.** It has no memory of your correction. Connect the [MCP server](https://blog.despia.com/despia-mcp-server-stop-burning-credits-on-bad-code), or paste the feature page into the project context so the correct shape is in front of it on every generation. Then check that it took: ask it how to detect the runtime and which package to import before you spend another build on the answer.

**Where do I check the real scheme name?** The capability reference in the docs. Every scheme, every parameter, and every callback name is listed there, and it is the same list the MCP server serves to your assistant.

## Add this to your app

Every native capability in this post is one JavaScript call away, inside your existing web codebase. No Xcode, no native project to maintain.

[Read the full reference at setup.despia.com](https://setup.despia.com)