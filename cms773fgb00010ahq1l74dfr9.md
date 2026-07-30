---
title: "Lovable to Mobile App: How to Get on the Stores"
seoTitle: "Lovable to Mobile App: How to Get on the Stores"
seoDescription: "Lovable builds web apps, not mobile apps. What is actually missing, the four routes to an App Store listing, and the path that keeps one codebase."
datePublished: 2026-07-30T07:31:36.069Z
cuid: cms773fgb00010ahq1l74dfr9
slug: lovable-to-mobile-app-how-to-get-on-the-stores
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/0cb51af0-bc77-4255-967c-f2ab435b9700.png
tags: android-app-development, ios, android, ios-app-development, ai-tools, appstore, ai-agents, googleplay, lovable

---

You want an icon on a home screen, a listing people can search for, and a notification that arrives while the phone is in someone's pocket. What you have is a URL. Lovable does not close that gap and does not claim to, and [its own documentation](https://docs.lovable.dev/features/publish) says publishing always deploys to the web.

The gap is narrower than it looks. Your components, your Supabase schema and your auth all survive whichever route you take. What is missing is a signed binary, a store listing, and device capability the browser does not hand you.

## Lovable stops at the URL, and that is the whole problem

A published Lovable project is a React app served over HTTPS. Apple will not accept that. The App Store takes signed binaries, and a URL is not one, which is why there is no export button that produces an IPA.

Worth clearing up early, because it sends people in circles: Lovable ships a mobile app of its own for editing projects from your phone. That is a tool for you. It is not an output format for your users.

## The four ways out

| Route | App Store | Second codebase | Needs a Mac | Updates |
| --- | --- | --- | --- | --- |
| Progressive Web App | No | No | No | Instant |
| React Native rebuild | Yes | Yes, permanently | No | Native changes need review |
| Lovable's Capacitor export | Yes | The native project | Yes | No default answer |
| Despia | Yes | No | No | Whole UI over the air |

A Progressive Web App is the cheapest option and does not answer the question. Android will list one as a Trusted Web Activity. Apple lists no PWAs at all, so if your definition of a real app includes the App Store, this route ends here.

Rebuilding in React Native gives you genuinely native views, and it is the right call for animation-heavy, gesture-heavy or 3D products. What it costs is permanent. You keep 2 codebases, fix every bug twice, and build every feature twice, for UI the web platform already renders at 60 frames per second.

Lovable's Capacitor export hands you a real native project you own, along with everything nobody mentions in the pitch. You need a Mac and Xcode to compile and sign. Billing is not included. Push needs a plugin. Its offline mode serves your app from `file://`, and Supabase auth, OAuth redirects and Sign in with Apple all fail on that origin because the SDK will not initialise there. If you employ mobile engineers this is a legitimate path. If you built your app in a chat panel, it is a project.

Despia takes the app at its live URL and ships it as a signed binary for both stores, with 50+ native device features reachable from your existing JavaScript through a single function. One codebase, which is still your website. Web changes reach installed apps over the air with no resubmission, and the full Xcode and Android Studio projects export whenever you want them.

## A WebView is not the compromise you have been told it is

Search this and you will find pages arguing that wrapped apps are second-rate, mostly run by tools selling a React Native rebuild. Check it instead of arguing about it.

[Canva's technical requirements](https://www.canva.com/help/technical-requirements/) list Android System WebView 89 or higher as a requirement of the Canva app. Chrome does not use System WebView, so that line is not about browsing Canva in a browser. It is about the app.

On iOS you can see the same thing in 10 seconds. Open Canva, tap 3 or 4 times quickly on an ordinary part of the interface, hold the last tap and drag.

![](https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/4654a92d-9de9-4bf4-8cbd-413580bc0c8d.jpg align="center")

The text-selection magnifier appears over a tile label. That label is not a text field and not an input, it is ordinary interface chrome. Native labels are not selectable, so pressing and holding one does nothing at all. Web text is selectable by default, which is why the loupe shows up. The test is iOS only, since it is a WebKit behaviour and Android's WebView does not produce the same magnifier.

Run it on the Lovable app and it happens over the header.

![](https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/ac7f366d-3038-4ff2-826f-e3ce5a43a951.jpg align="center")

Run it on the Base44 app and it happens over the greeting on the home screen.

![](https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/98d5c3aa-1fee-4bc4-89d1-d5be981ed128.jpg align="center")

Both of the AI builders you are being told to rebuild away from render their own mobile interfaces as web content. So does one of the highest-rated design apps on either store. [Shopify's engineering team](https://shopify.engineering/mobilebridge-native-webviews) published the same decision in April 2025, describing WebViews as central to their mobile strategy across roughly 600 screens, while keeping native and React Native for the handful of features that need it.

None of those companies is short of engineers. The claim that native UI is a prerequisite for a successful app does not survive contact with the apps people actually use. Where the web platform genuinely loses is custom gesture recognizers, frame-by-frame game loops and heavy 3D, and if that is your product you should rebuild it.

## What Apple actually rejects

Not web apps. Apps that offer a user nothing they could not get by opening the same URL in Safari.

[Guideline 4.2](https://developer.apple.com/app-store/review/guidelines/) exists for binaries that load a website and stop there. Two or three real native behaviours on paths a reviewer will walk is the difference between a listing and a rejection letter: haptics on your primary action, push notifications, biometric login, the native camera, native billing.

A React Native rebuild clears 4.2 more easily, because native views are self-evidently not a webpage. That is true and worth conceding. It is also a rewrite, where the alternative is an afternoon.

## Shipping it

Fix the layout first, because it is cheap now and expensive later. Move primary navigation to a bottom tab bar, since a top nav or a hamburger tells anyone holding a phone they are on a website. Pad against the device insets so content clears the notch and the home indicator, with a fallback so the same CSS keeps working in a browser.

```css
.tab-bar { padding-bottom: calc(12px + var(--safe-area-bottom, 0px)); }
```

Then open the preview on your actual phone rather than a narrow desktop window. They are not the same thing, and every layout problem you find now costs nothing to fix.

Next, connect the Despia MCP so the agent writes real calls instead of inventing them. In Lovable that is Connectors in the sidebar, the Chat connectors tab, New MCP server, `https://setup.despia.com/mcp` as the URL. Set Authentication to No authentication. Lovable defaults that field to OAuth, and leaving the default is what makes the connection fail without telling you. Ask the chat to list the haptic schemes afterwards, and you should get 5 real ones back.

Native features are one call each, gated on the environment check because the same code still runs in a browser.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

export function completeTask(id) {
  markDone(id)
  if (isDespia) despia('successhaptic://')
}
```

Push is one more line once someone is signed in, mapping your own user id to the device so your backend targets a person rather than a handset.

```javascript
if (isDespia && userId) despia(`setonesignalplayerid://?user_id=${userId}`)
```

Before you build anything, connect a custom domain in Lovable and use it as the app's address. The URL is compiled into the binary and is the one thing an over-the-air update cannot change. Ship on `yourapp.lovable.app` and moving hosts later means a new binary and a new review. Ship on your own domain and moving is a DNS change, with the listing, the ratings and the reviews intact.

Then build. The build runs in the cloud and returns an IPA for App Store Connect and an AAB for Google Play, with signing handled. Install it on your own phone before you go near a store.

## The things that silently break

A reviewer who cannot sign in rejects the app as incomplete, so put working credentials in App Review Information, and use a password account rather than a magic link they cannot receive. This costs more first submissions than anything else.

Purpose strings get auto-rejected under Guideline 5.1.1 when they are generic. Write what the permission is for: "Used to attach photos to your entries", not "This app requires camera access".

Selling subscriptions through Stripe is a Guideline 3.1.1 rejection, because digital content requires Apple's own billing. RevenueCat covers both stores. Stripe stays correct for physical goods and real-world services.

And the Android back button should go back rather than close the app. It is a router problem, it is the most common complaint on converted apps, and nobody catches it before submitting.

## The 14-day Google Play wait

If your Play account is a personal account created after 13 November 2023, you need [12 testers opted in for 14 continuous days](https://support.google.com/googleplay/android-developer/answer/14151465) before you can apply for production. Internal testing does not count and the requirement is per app.

Friends are the free route if you can get to 12. Beyond that, [Testers Community](https://www.testerscommunity.com) runs a free app where developers test each other's projects to earn credits, and sells a done-for-you version from around $15. We have no affiliation and get nothing if you use it, and their pricing moves, so check it yourself. Judge any service on whether written feedback comes back, because Google's production application asks what feedback you received and what you changed because of it. Installs with no feedback leave you three empty boxes.

Do not sit out the 14 days. Ship the AAB to closed testing on day 1 to start the clock, then keep improving the web app, which reaches your testers with no new build and no reset. Apple has no equivalent rule, so submit there in parallel.

## What you end up with

One codebase that is still your website, listed on both stores under your own developer accounts. Copy fixes, new screens and bug fixes reach installed apps on next launch with no review queue. Only the native layer needs a new binary, which in practice means an app icon, a splash screen, or a genuinely new native integration.

## Get it on the stores

Take the app you just built and ship it to iOS and Android without a CLI or a Mac. Code signing and submission run from the browser.

[See the setup docs at setup.despia.com](https://setup.despia.com)