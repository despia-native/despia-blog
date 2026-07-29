---
title: "How to Get React In-App Purchases Approved"
seoTitle: "How to Get React In-App Purchases Approved"
seoDescription: "Get your React app approved with real in-app purchases. StoreKit and Play Billing through RevenueCat, without a second native codebase."
datePublished: 2026-07-29T12:46:01.182Z
cuid: cms62vx6i00000bi711zmf7mg
slug: how-to-get-react-in-app-purchases-approved
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/f83aad3e-5173-4ca4-a6f6-726eb48194e5.png
tags: reactjs, revenuecat, storekit

---

To get approved selling digital content, your React app needs three things: purchases running through StoreKit on iOS and Google Play Billing on Android, a paywall the reviewer can actually reach, and a working restore. Stripe is correct for your website and rejected inside the app. RevenueCat supplies the native layer, compiled into the binary, and your existing web subscribers keep their access under Guideline 3.1.3(b) at the same time.

You keep your components, your router, your state management, and your API. What changes is where the money comes from on mobile, and how one hook answers who has access.

## The approval checklist

| Requirement | Why it is on the list |
| --- | --- |
| Digital goods sold through StoreKit and Play Billing | Guideline 3.1.1, the core rule |
| A restore purchases option that works | Its own rejection when missing |
| A paywall the reviewer can reach on a clean install | Correct billing still fails if it is unreachable |
| Demo credentials in App Review Information | The usual reason a reviewer cannot get there |
| Products submitted with the app version | Configured is not the same as submitted |
| Price and billing period shown on the paywall | Required on any subscription screen |
| Terms and privacy links on the paywall | Same |
| Discounts as Apple offer codes, not your own promo codes | Your own codes are an external unlock |
| Existing web subscribers honoured, with in-app purchase also offered | The 3.1.3(b) condition |
| No pricing that steers iOS users to your website | Prohibited outside the US storefront |

## What Apple requires, and why Stripe cannot satisfy it

Apple splits payments by what is being sold, not by who processes them.

| What you sell | Payment method required on iOS |
| --- | --- |
| Subscriptions to digital content | In-App Purchase (StoreKit) |
| Premium features or a full version unlock | In-App Purchase (StoreKit) |
| Credits, tokens, AI generations | In-App Purchase (StoreKit) |
| Courses, lessons, digital downloads | In-App Purchase (StoreKit) |
| Physical products | Any processor |
| Real-world services (delivery, booking, transport) | Any processor |
| Sales direct to organisations | Narrow carve-out, see 3.1.3(c) |

Guideline 3.1.1 is explicit that if money unlocks features or functionality inside your app, you must use In-App Purchase. Google applies the equivalent rule through Play Billing.

Stripe has no StoreKit path and no Play Billing path, because those are native frameworks that compile into an app binary and a hosted checkout has nowhere to put them. Stripe Elements inside a React component is still a web checkout, however native the styling looks.

One correction worth making early, because it costs people a review cycle. This is not only about subscriptions. Developers report one-time Stripe purchases getting rejected under the same guideline, because the rule is about unlocking digital content, not about whether the charge recurs. A lifetime unlock fails exactly like a monthly plan does.

## Keep the subscribers you already have

Guideline 3.1.3(b), Multiplatform Services, permits apps operating across multiple platforms to let users access content, subscriptions, or features they acquired in your app on other platforms or on your website, provided those same items are also available as in-app purchases within the app.

The condition is the whole thing. Honour the web subscription, and also sell the same tier through StoreKit.

| Setup | Review outcome |
| --- | --- |
| Web subscription honoured, StoreKit paywall also offered | Approved |
| Web subscription honoured, no in-app purchase option | Rejected under 3.1.1 |
| Login-only companion app, no purchase UI at all | Usually rejected |
| Digital goods sold in-app through Stripe | Rejected under 3.1.1 |

The third row surprises people. Companion apps with no sign-up, no pricing, and no purchase UI still get rejected, because access to paid digital content triggers the rule rather than the presence of a buy button.

## The setup, start to finish

### 1\. Create the RevenueCat project

At app.revenuecat.com, create a project, then add your apps. For iOS you need your bundle ID, an App Store Connect API key with App Manager access, and an in-app purchase key. For Android you need the same package name as your build plus a Google Play service account JSON with permissions granted in Play Console.

Create your products in App Store Connect and Play Console, import them into RevenueCat, attach each to an entitlement, group them into packages, and put those packages in an offering. Mark one offering as default. An offering with no packages returns an empty list rather than an error, which is a confusing hour if you skip it.

### 2\. Point your agent at the guide

If you are pairing with Cursor, Claude Code, or any AI assistant, do not let it write this from its own idea of how RevenueCat works. Models that have seen a lot of React Native reach for `react-native-purchases`, which does not exist in your project and never will.

```text
Read this page and implement it in this app exactly as written:
https://blog.despia.com/react-in-app-purchases-without-webhooks-revenuecat.md

Rules while you implement it:
- This is a React web app shipped to iOS and Android with Despia. Native
  features come from the despia-native npm package.
- Detect the native runtime with the user agent. Never use window.despia, it
  does not exist and the check will be false on every device.
- Do not install react-native-purchases, @revenuecat/purchases-js, Capacitor,
  or Cordova. None of them applies here.
- Gate on the entitlement the store reports, never on a tier flag in our own
  database that our UI set.
- No webhooks and no new subscriptions table.

Two additions that are not in the guide:
- Before granting access, check the user's existing subscription record first,
  and only ask the store if that returns nothing. A customer who already
  subscribed on our website must never be shown a paywall.
- Pass our signed-in user id to RevenueCat as external_id, not an anonymous id.

When you are done, list every value I have to supply myself, and tell me which
files you changed.
```

### 3\. Add the keys, then rebuild

In the Despia Editor open App, Integrations, RevenueCat, paste the iOS and Android public SDK keys, and trigger a new build.

The RevenueCat SDK compiles into the binary, so it cannot arrive over the air the way your bundle does. Until you rebuild and install that build, the integration stays dormant even with the keys saved. Purchases fail quietly and entitlement checks return nothing while every dashboard screen looks correct. This is the single largest source of support tickets on this integration.

## One hook, one answer

Put access in a single hook and consume it everywhere. Scattering entitlement checks through components is how you end up with three definitions of premium that disagree.

```javascript
import { useState, useEffect, useCallback } from 'react'
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// display only: what to render, never what work to run
export function usePremium(user) {
  const [premium, setPremium] = useState(null) // null means not resolved yet

  const resolve = useCallback(async () => {
    if (!user) return setPremium(false)

    // web subscribers keep their access on every platform
    if (user.subscription_status === 'active') return setPremium(true)

    if (!isDespia) return setPremium(false)

    const { restoredData } = await despia('getpurchasehistory://', ['restoredData'])
    setPremium((restoredData ?? []).some(p => p.entitlementId === 'premium' && p.isActive))
  }, [user])

  useEffect(() => {
    resolve()
    window.onRevenueCatPurchase = resolve
    return () => { window.onRevenueCatPurchase = undefined }
  }, [resolve])

  return premium
}
```

Four React-specific things that cause real bugs here.

**Three states, not two.** `null` while unresolved, then `true` or `false`. If you initialise to `false` you get a paywall flash on every launch for people who already pay you, and that flash is what produces the review saying they were charged twice. Render a skeleton while it is `null`.

**Resolve after auth, not alongside it.** The hook depends on `user`, so it re-runs when auth lands. If you fire the entitlement check in a mount effect that races your auth provider, it resolves against a null user and locks a paying subscriber out until they refresh.

**Clean up the global callback.** `window.onRevenueCatPurchase` is a single global slot. Two mounted components assigning it means the second wins and the first silently stops updating. The cleanup in the effect return is not optional, and StrictMode's double-invoke in development will surface this immediately if you skip it.

**Route guards read the hook, they do not re-implement it.** A `<RequirePremium>` wrapper that does its own restore call will disagree with the hook eventually. One source, consumed.

If you are on Next.js or any framework that renders on the server, resolve access in the browser only. There is no device user agent and no native bridge during a server render, so treat any server-rendered view as unentitled until the client says otherwise.

## Block the expensive calls on the server

The hook reflects what the client claims. That is fine for hiding a screen and useless for protecting anything that costs you money, because anyone who opens React DevTools can flip that boolean.

So every premium action with a real cost behind it, an AI generation, a paid export, a third-party API you are billed per call for, gets checked when the request arrives.

```javascript
// server-side only, secret key never reaches the bundle
async function hasActiveEntitlement(appUserId, entitlementId) {
  const id = encodeURIComponent(appUserId) // anon ids carry a $RCAnonymousID: prefix
  const res = await fetch(
    `https://api.revenuecat.com/v2/projects/${PROJECT_ID}/customers/${id}/active_entitlements`,
    { headers: { Authorization: `Bearer ${RC_SECRET_KEY}` } }
  )
  if (res.status === 404) return false        // customer RevenueCat has never seen
  if (!res.ok) throw new Error(`RevenueCat ${res.status}`)
  const { items = [] } = await res.json()
  return items.some(e => e.entitlement_id === entitlementId)
}
```

The environment variable naming matters more in React than anywhere else. Anything prefixed `VITE_` or `NEXT_PUBLIC_` is compiled into the bundle and readable by anyone who opens the network tab. The RevenueCat secret key must carry neither prefix and must never be imported into a component. If your build tooling has ever warned you about an undefined env var and you fixed it by adding the public prefix, check that file now.

Two more practicals. The `appUserId` must be the same id you passed as `external_id` when launching the paywall, or RevenueCat resolves a different customer and correctly reports no entitlement. And in sandbox this endpoint can read empty while the entitlement is genuinely live, so cross-check against RevenueCat's older subscriber lookup while testing.

## Make sure the reviewer can find your paywall

This is where teams who did everything else right still get bounced, with a notice saying the subscription is not discoverable in review.

*   **Sign-in blocks the paywall.** Supply working credentials in App Review Information, and test them on a device that has never opened your app. Magic-link sign-in makes the credentials useless, so give the reviewer a password account.
    
*   **Onboarding breaks before the paywall.** A flow that stalls on a slow network leaves the reviewer stuck upstream of the thing they are meant to test.
    
*   **The paywall is region-gated.** Review commonly runs from outside your main market.
    
*   **Offerings fail to load for the reviewer specifically.** Render a real error state rather than an empty view. A blank screen reads as a broken app.
    

Say where the paywall is in your review notes. One sentence naming the screen and how to reach it removes the entire category.

## Use offer codes, not your own promo codes

A field where users type a code to unlock premium fails review. Apple's notice names it directly, saying the app unlocks functionality with mechanisms other than in-app purchase, specifically that it uses promo codes to unlock digital content, and directs you to an Apple-supported offer code instead.

Implement discounts as offer codes and promotional offers in App Store Connect. Your Stripe coupons stay on the web.

## What you can show in the app

| In-app element | Allowed? |
| --- | --- |
| Sign-in for existing web subscribers | Yes, and it should be easy to find |
| Restore purchases | Yes, and a missing one is its own rejection |
| StoreKit paywall at your chosen price | Yes |
| Managing an existing subscription | Yes, without pricing or purchase calls to action |
| "Subscribe on our website and save" | No, outside the US storefront |
| A Stripe customer portal link that can upgrade | No, outside the US storefront |
| A link to your web checkout | US storefront only, and gated |

The portal row catches people. A portal that only cancels or updates a card is account management. A portal that can start or upgrade a paid plan is a purchase mechanism.

Apple does not require price parity, so a higher in-app tier is permitted. What you cannot do is explain why inside the app.

## Does the Epic ruling change any of this?

Partly, in one country. Since the 2025 Epic v. Apple injunction, apps on the United States storefront may include buttons and external links pointing at outside checkout, with no entitlement required and no commission currently attached. Google's US policy moved the same way.

It applies to the US storefront only, so a global build needs storefront gating and two purchase paths regardless. You still cannot process the payment inside the app on iOS. A browser handoff converts worse than a native sheet. And the current zero commission exists because a court has not set a rate, not because a rate was decided.

Rules current as of July 2026 and actively moving. Check Apple's and Google's own policy pages before building around them.

## Before you submit

Run these on a physical device, from a clean install, on cellular.

*   Buy the subscription in sandbox and confirm premium unlocks.
    
*   Delete the app, reinstall, sign in, and confirm restore returns your entitlement.
    
*   Sign in as an existing web subscriber and confirm no paywall appears.
    
*   Open the paywall with the demo account you are giving Apple.
    
*   Confirm your in-app purchase products are attached to the version you are submitting.
    
*   Check the paywall shows price, billing period, terms, privacy, and restore.
    
*   Confirm no screen links to Stripe Checkout or a portal that can upgrade.
    
*   Write one line in the review notes saying where the paywall is.
    

## When something does not work

| What you see | Most likely cause |
| --- | --- |
| Entitlement check empty, keys are saved | No rebuild since the keys were added |
| Paywall button does nothing, no error | Environment check false, so the call never ran |
| `window.despia is not a function` | That API does not exist, use the user agent check |
| `Error fetching offerings`, code 23 | Store cannot return the products yet |
| Offerings load but contain no packages | Products not attached to an offering |
| Access resolves, then flips back to locked | Two components assigning `window.onRevenueCatPurchase` |
| Paywall flashes then disappears | Initialised to `false` instead of a null loading state |
| Existing web subscriber sees a paywall | Access check only reads the store |

Error code 23, `CONFIGURATION_ERROR`, says none of the products registered in RevenueCat could be fetched from the store. RevenueCat is relaying a refusal rather than failing. Work these in order: the Paid Applications Agreement must be signed and active in App Store Connect, products must not be sitting in READY\_TO\_SUBMIT, tax and banking must be complete, the bundle ID must match across Despia, App Store Connect and RevenueCat, and product identifiers must match exactly including case. On Android, the app must have had an active release on a testing track.

There is no console in TestFlight, so render the user agent, whether the environment check passed, and the raw entitlement response into a debug panel you can toggle.

## Common questions

### Do I need React Native for in-app purchases?

No. Your React web app ships as a real iOS and Android binary with the RevenueCat SDK compiled in, reached from JavaScript. A rebuild in React Native replaces your entire product to acquire one capability.

### Can I use react-native-purchases or the RevenueCat web SDK?

Neither. `react-native-purchases` targets a React Native runtime you do not have. The RevenueCat web SDK bills through Stripe, which is the thing Apple rejects. The native SDK compiled into the binary is what you want, called through the bridge.

### Does this apply to one-time purchases, or only subscriptions?

Both. Developers regularly report one-time Stripe purchases being rejected under the same guideline. The rule is about unlocking digital content, not about whether the charge repeats.

### Will my existing subscribers have to pay twice?

Not if you build the two-source access check. Guideline 3.1.3(b) exists so they do not, on condition that the same subscription is also purchasable in-app.

### Does this work with Next.js?

Yes, with one rule: resolve access in the browser. There is no user agent from the device and no native bridge during a server render, so a server-rendered view is unentitled until the client resolves.

### Do I need webhooks?

No. Restoring entitlements on the device covers what the app displays, and one direct API call covers anything the server has to protect. Add a webhook later if you need real-time cancellation handling.

### How long does this take?

The hook is an afternoon. The slow parts are the RevenueCat project, store product configuration, and the build. Budget a day.

## When you want an answer from a person, not a model

An AI assistant does not tell you when it is unsure. It writes something confident, you spend a build finding out it was wrong, and you go around again.

If you are stuck and want a human to look at it, email `humans@despia.com`. A developer with five or more years of experience reads it and writes back about your actual setup, not a template. A person will sometimes tell you your approach is wrong, or that the roadblock is upstream of the thing you are debugging. That is the point of asking one.

## Get it on the stores

Take the React app you already built and ship it to iOS and Android without a CLI or a Mac. StoreKit and Play Billing arrive through RevenueCat with one JavaScript call, your web subscribers keep the access they paid for, and your existing codebase stays the single source of truth.

[See the setup docs at setup.despia.com](https://setup.despia.com)