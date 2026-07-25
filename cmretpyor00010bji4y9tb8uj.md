---
title: "Native Overscroll Bounce in WebView Apps, iOS and Android"
seoTitle: "Native Overscroll Bounce in WebView Apps, iOS and Android"
seoDescription: "Get native overscroll in a WebView app with pure CSS: rubber-band on iOS, stretch on Android. Scroll an inner container, not the body, no native code."
datePublished: 2026-07-10T10:59:39.878Z
cuid: cmretpyor00010bji4y9tb8uj
slug: native-overscroll-bounce-in-webview-apps-ios-and-android
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/910f4f06-672c-48c1-b7fd-dccbbfe878c5.png
tags: webview, webapp, pwa

---

The thing that makes a wrapped web app feel like a website is usually the scroll. Drag past the top of a real native screen and it stretches and springs back. Drag past the top of a web app that scrolls the body and you get a dead stop, the fixed header dragging with the page, and a flash of the WebView background underneath. The fix is not native code. It is one decision about which element scrolls.

Scroll an inner container instead of the body, and iOS and Android each draw their own native edge effect for free. Same CSS, both platforms.

## Why this works at all

Despia ships your web app inside the real platform WebView, WKWebView on iOS and the Chromium-based WebView on Android. Those are the same engines the OS uses everywhere else, so they already know how to draw a rubber-band bounce or an edge stretch. You do not implement the effect. You just have to hand the scroll to an element the WebView will treat as a scroll port, and stop the body from eating the gesture first.

The body is the problem. When the document body scrolls, the WebView applies the overscroll to the whole viewport, which drags anything you positioned as fixed chrome and exposes the background behind the page. Move the scroll into a child container and the effect stays where it belongs.

## Drop-in CSS

```css
/* body never scrolls */
html, body { margin: 0; height: 100%; overflow: hidden; overscroll-behavior: none; }

/* one-viewport app shell */
#app { height: 100dvh; display: flex; flex-direction: column; }

/* fixed chrome, stays put, owns the safe areas */
.app-header { flex: none; padding-top:    calc(0.5rem + var(--safe-area-top,    env(safe-area-inset-top,    0px))); }
.tab-bar    { flex: none; padding-bottom: calc(0.5rem + var(--safe-area-bottom, env(safe-area-inset-bottom, 0px))); }

/* the scroller, this is where overscroll lives */
.scroll-container {
  flex: 1;
  min-height: 0;                   /* required in flex, or it collapses */
  overflow-y: auto;
  overscroll-behavior-y: contain;  /* keep the bounce, stop chaining to body */
}
```

```html
<div id="app">
  <header class="app-header">...</header>
  <main class="scroll-container"><!-- all scrollable content --></main>
  <nav class="tab-bar">...</nav>
</div>
```

Three lines carry the whole thing. `overflow: hidden` on the body kills the body scroll. `100dvh` on the shell measures against the dynamic viewport so the layout is right whether or not the URL bar is showing. `min-height: 0` on the scroller is the one people forget: a flex child defaults to `min-height: auto`, which refuses to shrink below its content, so the container never actually scrolls and you spend an hour wondering why.

The chrome padding uses Despia's runtime insets, not bare `env()`. Inside a Despia app the runtime injects `--safe-area-top` and `--safe-area-bottom`, and that is the source of truth. The `env(safe-area-inset-*)` value is the fallback for web preview before the runtime variables exist, and the `0px` at the end keeps the `calc()` valid when neither resolves. Write bare `env()` on its own and your header will sit under the notch in preview and jump once the app loads.

## What each platform draws

Same CSS, one native effect per OS, none of it written by you.

| Platform | Effect |
| --- | --- |
| iOS, WKWebView | Rubber band, stretch past the edge and spring back |
| Android 12 and up | Stretch overscroll, content stretches then snaps |
| Android 11 and below | Edge glow, a glow at the edge with the content held still |

The edge effect is drawn by the OS, not by CSS, so it looks native on each platform but is not pixel-identical across them. That is correct, not a bug. A user on Android 13 should see the Android stretch, not an iOS bounce faked in JavaScript.

## The one setting that quietly kills it

`overscroll-behavior-y` has two values people mix up.

`contain` stops the scroll from chaining to the parent but keeps the local overscroll affordance, so you still get the bounce or the stretch. That is what you want on the inner container.

`none` also suppresses the overscroll effect itself. Set `none` on your scroller by reflex and the native bounce disappears, then you assume the WebView does not support it and go looking for a native plugin that does not exist. Use `none` on the body to lock it down, `contain` on the scroller to keep the feel.

## Short pages will not overscroll

There is nothing to spring back from if the content fits on screen, so a short page shows no effect. That is expected behaviour, not a broken setup. If you want the bounce to be reachable even on a short screen, give the content a hair more height than the viewport:

```css
.scroll-container > .content { min-height: calc(100% + 1px); }
```

## Add this to your app

Native scroll physics is a CSS decision in a Despia app, not a native project. The same web codebase that runs in your browser ships to the App Store and Google Play, and the WebView hands you the platform overscroll for free.

[Read the full reference at setup.despia.com](https://setup.despia.com)