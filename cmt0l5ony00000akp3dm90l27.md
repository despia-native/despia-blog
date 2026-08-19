---
title: "Base44 Push Notifications and Apple Guideline 4.2"
seoTitle: "Base44 Push Notifications and Apple Guideline 4.2"
seoDescription: "Base44 apps can send push notifications. Why that alone does not clear a guideline 4.2 minimum functionality rejection, and what actually does."
datePublished: 2026-08-19T21:10:35.070Z
cuid: cmt0l5ony00000akp3dm90l27
slug: base44-push-notifications-and-apple-guideline-4-2
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/9021c391-6519-4edb-8ced-560a43e182e5.png
tags: ios-apps, ios, webview, ios-app-development, no-code, ai-tools, ai-agents, base44

---

Base44 apps can send push notifications, and the setup is documented and straightforward. Push alone does not satisfy Apple's guideline 4.2. Apple's own minimum functionality rejection letter names push notifications, Core Location and sharing as features that do not make an app robust enough on their own. Here is what does.

## What did Base44 actually ship?

**Push notifications for the generated iOS and Android builds, configured in the same window where you generate your store files. It is a real feature and it works.**

It is also, per Base44's own documentation, the only native capability the Base44 mobile app currently supports. The setup runs through your own credentials on both platforms. On iOS you create an APNs auth key under Certificates, Identifiers and Profiles in your Apple Developer account, download the `.p8` file, and upload it with the matching Key ID. This is a different key from the App Store Connect API key you upload in the same flow, and both files end in `.p8`, so label them as you download them. Apple lets you download each one exactly once.

On Android you create your own Firebase project, register the Android app against the package name Base44 gives you, then upload two files: `google-services.json`, which is compiled into the build so the device can receive notifications, and a service account key, which lets Base44 send them on your behalf.

Sending happens from your app backend through the SendPushNotification integration. Each notification carries a title, a message, an optional action button label, and an optional link to a page inside your app.

One operational detail worth planning around: people who already installed your app do not start receiving notifications until you generate new store files and ship an update through App Store Connect and Play Console. Push is compiled into the shell, so turning it on is a new binary, not a publish.

## Does adding push notifications clear a guideline 4.2 rejection?

**No. Guideline 4.2 asks for features, content and UI that elevate the app beyond a repackaged website.**

Push is one input to that judgment, and Apple's standard 4.2 rejection letter singles it out by name as something that does not carry the app on its own.

The guideline text itself, [last updated June 8, 2026](https://developer.apple.com/app-store/review/guidelines/), is short:

> Your app should include features, content, and UI that elevate it beyond a repackaged website. If your app is not particularly useful, unique, or "app-like," it doesn't belong on the App Store.

The sentence people quote about push notifications is not in the guideline. It is in the rejection letter reviewers send under 4.2, and developers have posted it on Apple's own forums for years:

> Your app provides a limited user experience as it is not sufficiently different from a mobile browsing experience. As such, the experience it provides is similar to the general experience of using Safari. Including iOS features such as push notifications, Core Location, and sharing do not provide a robust enough experience to be appropriate for the App Store.

That distinction matters. The rulebook states the standard. The letter tells you which specific additions reviewers have already decided do not meet it. If you are shipping a wrapper whose one native addition is push, the letter you are most likely to receive has a sentence in it about push.

Guideline 4.2.2 sits underneath the same idea: other than catalogs, apps should not primarily be marketing materials, advertisements, web clippings, content aggregators, or a collection of links.

Note that the guideline names 3 things, features, content and UI. Teams reading it after a rejection tend to fix only the first, add a capability, resubmit, and receive the same letter, because the interface half never changed. An app that navigates like a desktop website, with a sidebar, a hamburger menu or a cookie banner, reads as a website no matter what runs underneath it.

## Why is push a weaker differentiator than it used to be?

**Because the web platform caught up. Web Push has worked on iOS and iPadOS since 16.4 for web apps added to the Home Screen, and Safari 18.4 added Declarative Web Push.**

The Badging API and the Web Share API are available in the same context. This is the part of the argument most wrapper marketing gets wrong, and getting it wrong is expensive, because any developer can dismantle the claim with one link. Push notifications are not something the web cannot do. Neither is location: the Geolocation API is standard and has been for well over a decade.

So the useful question is not what a browser is technically capable of. It is the one a reviewer is actually asking: **what does the installed app experience add over opening the same site in Safari?** If the honest answer is "notifications and an icon," you have described a Home Screen web app with an App Store listing attached, and 4.2 exists precisely for that case.

Examples of app-level integration that answer the question properly:

*   StoreKit and Google Play Billing for digital purchases
    
*   A platform biometric flow tied to real credential storage
    
*   Native camera, photo picker and file picker flows
    
*   Native share integration in both directions
    
*   Contacts, calendar and precise or background location
    
*   Native haptics
    
*   Background audio with lock screen and Control Center controls
    
*   Storage that survives a cold start, and offline that survives one too
    
*   Deep links and universal links
    
*   Health data through HealthKit or Health Connect
    

None of these are magic. What they have in common is that they are integrations with the operating system rather than features of the page.

## Can push notifications be the feature your app depends on?

**No, and this is the trap that catches apps whose only native capability is push.**

Three separate guidelines constrain what you can do with it, and a wrapper leaning entirely on push tends to collide with at least one of them.

**Guideline 4.5.4** states that push notifications must not be required for the app to function. If your onboarding dead-ends when a user declines the permission prompt, that is a rejection independent of 4.2.

**Guideline 5.1.2(i)** goes further: your app may not require users to enable system functionality such as push notifications, location services or tracking in order to access functionality, content, use the app, or receive compensation of any kind. A "turn on notifications to continue" gate is a documented violation, not a growth tactic.

**Guideline 4.10** prohibits monetizing built-in capabilities provided by the hardware or operating system, and names push notifications explicitly. Selling notifications as the premium tier is out.

Read together, the position is consistent: Apple treats push as an enhancement layered on top of an app that already justifies itself. An app whose value proposition is the notification has the dependency backwards.

## What does the Base44 mobile app support today?

**A web view around your published Base44 app, plus push notifications. Base44's documentation is direct about the shape of it and about what is missing, which is worth reading before you plan a submission around it.**

From Base44's own store submission documentation: - The mobile app runs your published app inside a secure web view, described as a lightweight native wrapper that opens only your app's URL

*   Full offline mode and HealthKit are not supported
    
*   Device permissions are set by an AI scan and are not editable in the Base44 interface
    
*   The main entry URL is chosen automatically from your published app and cannot be changed for the app specifically
    
*   Bundle ID and signing key are configured automatically and cannot be changed inside the generated IPA or AAB, and the package name follows a documented `com.base[app-id].app` format that is permanent and publicly visible in the Play Store URL
    
*   For digital goods, Stripe is not permitted inside the mobile app, and a StoreKit and Play Billing integration is described as in progress
    

That last one is the sharpest constraint for anyone selling subscriptions. Apple and Google require their own billing systems for digital content, and Base44's documentation states plainly that an app using Stripe for digital content is rejected. Until the native billing integration ships, a Base44 app that sells digital content has no compliant path to the stores through the built-in wrapper.

There is also a known limitation where the iOS build includes a HealthKit entitlement without the matching `NSHealthShareUsageDescription` key, which surfaces as an App Store Connect rejection even for apps that never touch health data.

Base44's documentation also says the platform continues to expand native feature coverage and points readers to its release notes. That is a fair statement of intent and there is no reason to doubt it. It does mean the practical question for anyone planning a submission is not whether a capability is coming, but whether it is there on the day you need to ship.

## Does it matter that Despia only does this?

**It matters at the edges, and store submissions fail at the edges. Base44 is an AI app builder where the mobile wrapper is one feature among many. Despia's runtime has done nothing but this since 2011.**

Both are legitimate positions, and they produce different release cadences. The runtime started life in 2011 as Advanced WebView, built for agency client apps, and over 7,500 apps shipped on it before it became Despia in 2023. That period covers the introduction of guideline 4.2, several rewrites of the review rules, the arrival of App Tracking Transparency, the privacy manifest requirements, and two full generations of store submission tooling on both platforms. Watching those land, one after another, and keeping shipped apps compliant through each of them is not a feature you can add to a roadmap. It is just time.

That shows up in two places a builder actually feels.

**Coverage.** Over 50 native capabilities against a wrapper that currently supports push notifications. The number matters less than the shape of the gap: in-app purchases, biometrics, persistent credential storage, offline, background audio and health data are the ones that decide whether a specific app can ship at all.

**Cadence.** On a general-purpose platform, native capability competes for engineering time with the editor, the AI, the hosting, the backend and the billing, and it should, because those are the product. On a specialist runtime it is the product. When a store policy changes or a new platform capability lands, the response time is different, and if you want to be among the first to ship against something new, that difference is the whole reason to pick one over the other.

None of this makes the built-in wrapper a bad choice. Fewer vendors is a genuine advantage, and plenty of apps never need a capability the wrapper lacks. The trade is straightforward: one platform to manage against a deeper native surface and a team whose entire job is the part where submissions get rejected.

## Who actually sends the notification?

**With the Base44 wrapper, Base44 does, using the APNs key and Firebase service account you upload. With Despia, OneSignal does, from an account you own and log into.**

That difference has almost nothing to do with delivery and almost everything to do with what surrounds the send. Despia does not run its own push service, and that is a deliberate decision rather than a gap. The principle is narrow and easy to state: build the runtime, and integrate the category leader for everything else. Nobody needs a fifth-best notification backend bolted onto a mobile runtime, and building one would mean spending years rebuilding a product that already exists and is better than anything we would ship.

OneSignal has been building messaging infrastructure since 2014 and does nothing else. It describes itself as serving over 1 million businesses and delivering more than 12 billion messages a day. When you ship a Despia app, you onboard to OneSignal directly, which means you get the platform rather than an endpoint:

*   A dashboard with delivery, open and conversion analytics rather than a send confirmation
    
*   Segmentation and targeting, so a campaign goes to the users who should get it
    
*   Scheduling, including delivery at a chosen local time in each user's own timezone
    
*   Journeys, which is the automation layer for onboarding sequences, win-back flows and lifecycle messaging
    
*   A/B testing and templates
    
*   In-app messages, email and SMS on the same user record, so retention is one system rather than four
    
*   Integrations with the tools you already use, including Zapier, n8n, warehouses and analytics platforms
    

OneSignal has also opened both an in-platform AI assistant and a hosted MCP server to all users. The MCP server runs on the open Model Context Protocol with OAuth, so Claude, ChatGPT, Cursor and Copilot can operate your OneSignal account directly across messaging, users, segments, templates, exports and analytics. Practically, that means you can build a segment, draft the copy and schedule the send in one conversation, and orchestrate it alongside whatever else your agent has access to.

None of that is push infrastructure. It is the marketing and retention layer that sits on top of push infrastructure, and it is the reason the choice of provider matters after launch rather than at integration time.

## What happens when you need a feature the runtime has not shipped?

**For push, usually nothing, because the OneSignal REST API is available to you directly.**

You are not waiting on a wrapper method to be exposed for each field, and you are not filing a request for something OneSignal already supports.

That is the practical payoff of integrating an established provider instead of inventing one. Your backend calls OneSignal, and the full payload surface is yours:

```javascript
// Backend: targeted send with in-app routing, no reload
await fetch('https://onesignal.com/api/v1/notifications', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        Authorization: `Basic ${process.env.ONESIGNAL_REST_API_KEY}`,
    },
    body: JSON.stringify({
        app_id: process.env.ONESIGNAL_APP_ID,
        include_external_user_ids: [userId],
        headings: { en: 'Order shipped' },
        contents: { en: 'Track it now' },
        data: { path: '/orders/1042' },
    }),
})
```

Routing is worth calling out because it is the difference between a notification that works and one that annoys people. Sending `data.path` updates the route with `pushState` and fires `popstate`, so the app moves to the right screen without a reload. The legacy `url` field triggers a full navigation and should be used only when a reload is genuinely wanted.

Two push features do need us, and both for the same reason: they touch the binary rather than the API.

**Custom notification sounds.** The audio file has to be compiled into the app, so it cannot be added from a dashboard. Email [support@despia.com](mailto:support@despia.com) with a `.wav` file and we add it to your build, after which you select it from your payload like any other OneSignal option.

**Critical alerts.** These bypass Do Not Disturb and silent mode, so Apple gates them behind a dedicated entitlement you request directly from Apple. Approval is a manual review and can take several weeks. Once Apple grants it, contact support with the confirmation, we enable it on your project, you rebuild, and then `ios_critical_alert: 1` works in your payload. The gate is Apple's rather than ours, which is why the approval has to be yours.

Rich push with images is a third case worth planning for, though it is setup rather than a request: on iOS it needs the notification service extension and a matching App Group configured against your bundle ID.

The same philosophy runs through the rest of the runtime. In-app purchases integrate the official RevenueCat SDK rather than a homegrown billing layer, and the SDK versions are maintained and provisioned on every rebuild, so you are not tracking StoreKit and Google Play Billing releases yourself.

If something reasonable is genuinely missing, [features@despia.com](mailto:features@despia.com) goes to the engineering team. There is no promise attached to that address, and a capability only one app in the world needs is not going to jump the queue. A feature that is executable, sensible and useful to more than one builder tends to get prioritized, and it ships quickly, because most of the engineers here have been doing this specific work for 15 years.

## How does the Despia runtime differ?

**Despia is a native runtime rather than a shell around a URL.**

Your Base44 app stays the source of truth, and native capabilities are called from that same web codebase through a single JavaScript function, with over 50 available.

|  | Base44 mobile app | Despia |
| --- | --- | --- |
| Rendering | Web view around your published URL | WKWebView on iOS, Chromium WebView on Android |
| Push notifications | Sent by Base44, via your APNs key and Firebase | Sent by OneSignal, from an account you own |
| In-app purchases | Not supported, Stripe not permitted | RevenueCat paywalls, purchases and Customer Center |
| Biometrics | Not supported | Face ID and Touch ID, bound to encrypted storage |
| Persistent storage | Not supported | Storage Vault, survives uninstall and reinstall |
| Offline | Not supported | Your own service worker, enabled by the runtime |
| Background audio | Not supported | Native player with lock screen and Control Center controls |
| Health data | Not supported | HealthKit and Health Connect |
| Haptics | Not supported | Supported |
| Bundle ID | Generated, not changeable | Chosen by you at project creation |
| Over-the-air updates | Web layer only | Remote hydration, no resubmission |
| Native project export | Not offered | Full Xcode and Android Studio projects |

Two of those rows do most of the work in a 4.2 conversation.

**Storage Vault** is encrypted key-value storage backed by iCloud Key Value Store on iOS and Android Key/Value Backup on Android. Values survive uninstall and reinstall and sync across devices on the same Apple ID or Google account. Setting `locked=true` on a value means reading it back requires Face ID or Touch ID, which turns "biometric login" from a UI gesture into an actual credential flow.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

if (isDespia) {
    // Store the session token behind biometrics
    await despia('setvault://?key=sessionToken&value=' + token + '&locked=true')

    // Reading it prompts for Face ID or Touch ID
    const data = await despia('readvault://?key=sessionToken', ['sessionToken'])
    resumeSession(data.sessionToken)
}
```

That is a native capability with no web equivalent in the relevant sense: a token that outlives a reinstall and is gated by the platform's own authentication. It is also the answer to two product problems at once, since the same persistence is what stops trial abuse by reinstall.

**RevenueCat** solves the billing problem without you writing against StoreKit and Google Play Billing separately. Those are two different APIs with two different lifecycles, and maintaining both is the reason most small teams get subscriptions wrong. With Despia you present a paywall configured in the RevenueCat dashboard, check entitlements on load, and open the native Customer Center for restores, subscription management and refund requests.

```javascript
if (isDespia) {
    despia('revenuecat://launchPaywall?external_id=' + userId + '&offering=default')
}

window.onRevenueCatPurchase = () => checkEntitlements()
```

## When is the Base44 wrapper the right call?

**When your app does not sell digital content, does not need to work offline, does not need biometrics or persistent credentials, and notifications genuinely are the only device integration your product design calls for.**

That is a real category. An internal tool, a members area, a booking front end for a business that already has a website, a content app where the reviewer can see a clear reason to install it. If your app is substantial in its own right and push is a nice addition rather than the whole native story, the built-in path is fewer moving parts and fewer accounts to manage, and it is free of the Stripe constraint if you sell nothing digital in the app.

Where it gets expensive is the case Base44 documents most clearly: you sell subscriptions. There is no compliant route through the built-in wrapper today, and no amount of push configuration changes that.

## What is the next step once the app is review-ready?

**Once the app itself is review-ready, the remaining problem is getting a compliant binary signed and onto both stores.**

Base44 can scan your app, generate the IPA and the AAB, and add push notifications, and for apps that need nothing else that may be all you require. Despia is what you reach for when in-app purchases, biometrics, persistent storage, offline or background audio are core to the product rather than optional, from the same Base44 codebase you already have.

[Learn more about Despia in the docs](https://setup.despia.com) or [start building at despia.com](https://despia.com).