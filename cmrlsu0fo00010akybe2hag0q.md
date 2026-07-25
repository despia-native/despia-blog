---
title: "Despia Is Not Capacitor: Why Your AI Gets It Wrong"
seoTitle: "Despia Is Not Capacitor: Why Your AI Gets It Wrong"
seoDescription: "Despia is not Capacitor, so AI tools scaffold the wrong native code. Here is why window.despia fails and the despia-native pattern that works."
datePublished: 2026-07-15T08:09:12.384Z
cuid: cmrlsu0fo00010akybe2hag0q
slug: despia-is-not-capacitor-why-your-ai-gets-it-wrong
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/29d68fb4-ca4f-40fb-a6b9-b83bf1365c05.png
tags: webview, pwa, capacitor

---

You describe the app, the AI builds the web part in minutes, then you ask for push notifications or in-app purchases and it wires them up as Capacitor. The code compiles. It looks right. It never runs. The tell is almost always the same: a call to `window.despia()`, an API that does not exist.

This is one of the most common issues builders hit when they use an AI tool to add native features to a Despia app. It is not a Despia bug and it is not your mistake. It is a training-data problem, and once you see why it happens the fix is short.

## Your AI defaults to Capacitor, and you can see why

Capacitor and Cordova have close to a decade of public documentation, tutorials, and Stack Overflow answers online. When a model is asked to turn a web app native, that is the pattern it has seen most, so that is the pattern it reaches for. Despia's context is not in the training data by default. It has to be supplied.

Capacitor itself is a legitimate, well-built native runtime. There is nothing wrong with it. The problem is narrower than that: Despia is a different runtime with a different API, and the two are not interchangeable. Code written for one does not work in the other, even though both take a web app and ship it to the App Store and Google Play.

So the AI produces something that reads like a native integration, passes a compile, and silently does nothing on the device.

## window.despia is not a real API

Here is the shape of the broken code, lightly cleaned up from a real support case:

```js
// AI-generated, does not work in Despia
if (typeof window.despia === 'function') {
  window.despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)
} else {
  console.warn('Despia bridge not available')
}
```

`window.despia` is not a function, and never has been. It is invented. So `typeof window.despia === 'function'` is always false, the code takes the `else` branch every time, and the paywall call never fires. The user taps the button, nothing opens, and the app either shows a fallback message or optimistically flips some local premium flag.

The important part: this is not a paywall problem. It is a runtime-detection problem, and the same hallucinated check breaks every native feature the AI wires this way. Haptics, push, biometrics, purchases, all of it dies in the same dead branch.

You will often see it paired with a Capacitor check for good measure:

```js
// also wrong for Despia
const isNative = window.Capacitor?.isNativePlatform?.() ?? false
```

Despia does not populate `window.Capacitor` either. Both checks resolve to false, and the native path is never taken.

## The pattern that actually works

Despia exposes native features through the `despia-native` package and a single `despia()` function. You detect the native runtime from the user agent, not from a global that does not exist.

```js
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

if (isDespia) {
  despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)
}
```

Two rules keep this correct across every feature. Put your `despia()` calls inside the `isDespia` gate so they only run in the native runtime. Assign global callbacks outside the gate, because the runtime calls them and they need to exist regardless of when they fire.

```js
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// haptic on a successful action, native only
if (isDespia) despia('successhaptic://')

// the runtime calls this when a purchase completes, so it lives outside the gate
window.onRevenueCatPurchase = () => checkEntitlements()
```

That is the whole shape. No bridge object to probe, no plugin import per feature, no `window.despia`.

## Capacitor and Despia, side by side

Both run your web code in the platform WebView and ship a real app to both stores. The difference is structural, and it is the part your AI cannot infer on its own.

|  | Capacitor | Despia |
| --- | --- | --- |
| What you ship | A native project you own and maintain | Your web app as a native binary, one codebase |
| Native calls | A plugin import per feature | One `despia()` function, 50+ features |
| Runtime detection | `Capacitor.isNativePlatform()` | `navigator.userAgent` includes `despia` |
| Updating content | Rebuild and resubmit for store review | Over the air, no resubmission |
| AI training data | A decade of public docs | Must be supplied via MCP or the docs |

The line that matters to most builders is the first row. Despia keeps one codebase. Your web app stays the source of truth, and you do not carry a second native project alongside it.

## When Capacitor is the right call

Be honest about this. Capacitor is open source, it hands you the Xcode and Android Studio projects to own outright, and it has a large plugin ecosystem. If you want to live in that native toolchain yourself, hand-maintain the generated projects, or you depend on a specific community plugin with no Despia equivalent, Capacitor is a reasonable choice and you should use it.

Despia's answer to the ownership objection is the export escape hatch: it produces full Xcode and Android Studio projects any time you want them. But the default path, and the reason to pick it, is that you do not have to. One web codebase ships and updates on both stores.

## How to make your AI write correct Despia code

The reason the AI guesses is that it has no grounded context. Give it some and the Capacitor code stops.

The most reliable option is the Despia MCP. It gives your assistant live, correct knowledge of the `despia-native` package, so it writes the right syntax without you explaining the API. Most builders support it, including Lovable, Base44, Bolt, Cursor, and Claude.

```text
https://setup.despia.com/mcp
```

If your tool cannot use an MCP, point it at the machine-readable docs index instead, or drop the setup pages into your project knowledge:

```text
https://setup.despia.com/llms.txt
```

At minimum, add one instruction to your system prompt:

```text
This app uses Despia, not Capacitor or Cordova. Despia has its own native
runtime and API. Never use window.despia, window.Capacitor, or Capacitor
plugins. Detect the runtime with navigator.userAgent and call native
features through the despia-native package. Reference only setup.despia.com.
```

Any of the three stops the hallucination at the source. The MCP is the one that also keeps up as the API grows.

## Ship one app, not two

If you are going to keep building in your web stack, keep building there. Despia ships that app to the App Store and Google Play, gives it native device access through one JavaScript call, and updates it over the air, from the one codebase you already have. It is a native runtime, not Capacitor, and the code has to match.

[Learn more about Despia in the docs](https://setup.despia.com)