---
title: "How to Get Vue In-App Purchases Approved"
seoTitle: "How to Get Vue In-App Purchases Approved"
seoDescription: "Get your Vue app approved with real in-app purchases. StoreKit and Play Billing through RevenueCat, without a second native codebase."
datePublished: 2026-07-29T12:48:09.123Z
cuid: cms62ynwt00000age8iiv0avr
slug: how-to-get-vue-in-app-purchases-approved
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/bbd56151-95eb-49d8-9a48-0055dc7c01ea.png
tags: vuejs, revenuecat, storekit

---

To get approved selling digital content, your Vue app needs three things: purchases running through StoreKit on iOS and Google Play Billing on Android, a paywall the reviewer can actually reach, and a working restore. Stripe is correct for your website and rejected inside the app. RevenueCat supplies the native layer, compiled into the binary, and your existing web subscribers keep their access under Guideline 3.1.3(b) at the same time.

You keep your components, your router, your store, and your API. What changes is where the money comes from on mobile, and how one composable answers who has access.

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

Stripe has no StoreKit path and no Play Billing path, because those are native frameworks that compile into an app binary and a hosted checkout has nowhere to put them. Stripe Elements mounted in a Vue component is still a web checkout, however native it looks.

This is not only about subscriptions. Developers report one-time Stripe purchases getting rejected under the same guideline, because the rule is about unlocking digital content rather than whether the charge recurs. A lifetime unlock fails exactly like a monthly plan does.

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

If you are pairing with an AI assistant, do not let it write this from its own idea of how RevenueCat works. Models reach for Capacitor plugins by default, because that is what most Vue-to-mobile content describes.

```text
Read this page and implement it in this app exactly as written:
https://blog.despia.com/vue-js-in-app-purchases-without-webhooks-revenuecat.md

Rules while you implement it:
- This is a Vue web app shipped to iOS and Android with Despia. Native features
  come from the despia-native npm package.
- Detect the native runtime with the user agent. Never use window.despia, it
  does not exist and the check will be false on every device.
- Do not install Capacitor, Cordova, or any RevenueCat plugin. None applies here.
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

## One store, one answer

Entitlement is global state, so put it in a Pinia store rather than a composable each component calls independently. Otherwise every mounted component runs its own restore and you get several definitions of premium that disagree.

```javascript
import { defineStore } from 'pinia'
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

export const useAccess = defineStore('access', {
  state: () => ({ premium: null }), // null means not resolved yet

  actions: {
    // display only: what to render, never what work to run
    async resolve(user) {
      if (!user) return (this.premium = false)

      // web subscribers keep their access on every platform
      if (user.subscription_status === 'active') return (this.premium = true)

      if (!isDespia) return (this.premium = false)

      const { restoredData } = await despia('getpurchasehistory://', ['restoredData'])
      this.premium = (restoredData ?? [])
        .some(p => p.entitlementId === 'premium' && p.isActive)
    },
  },
})
```

Register the purchase callback once, at app setup, not inside a component:

```javascript
// main.js, after the Pinia instance exists
const access = useAccess()
window.onRevenueCatPurchase = () => access.resolve(currentUser())
```

Four Vue-specific things that cause real bugs here.

**Three states, not two.** `null` while unresolved, then `true` or `false`. A `v-if="!access.premium"` on a store initialised to `false` renders the paywall for a split second on every launch, including for people who already pay you. That flash is what produces the review saying they were charged twice. Guard on `access.premium === null` and render a skeleton.

**Resolve after auth, not in** `onMounted`**.** A mount hook races your auth initialisation and resolves against a null user. Watch the user instead, so the check re-runs when auth lands.

**Register the callback once, globally.** `window.onRevenueCatPurchase` is a single global slot. Assign it in a component and the next component to mount overwrites it, silently. In `main.js` it exists for the life of the app and nothing competes for it.

**Route guards read the store.** A `beforeEach` guard that performs its own restore call will disagree with the store eventually. One source, consumed everywhere.

If you are on Nuxt or anything rendering on the server, resolve access on the client only. There is no device user agent and no native bridge during a server render, so treat server-rendered output as unentitled until the client says otherwise.

## Block the expensive calls on the server

The store reflects what the client claims. That is fine for hiding a screen and useless for protecting anything that costs you money, because anyone who opens Vue DevTools can flip that value.

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

Environment variable naming matters. Anything prefixed `VITE_` is compiled into the bundle and readable by anyone who opens the network tab, and Vue projects are almost always Vite projects. The RevenueCat secret key must not carry that prefix and must never be imported into a component or a store.

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
| Access resolves, then stops updating after a purchase | Callback assigned in a component that unmounted |
| Paywall flashes then disappears | Store initialised to `false` instead of `null` |
| Existing web subscriber sees a paywall | Access check only reads the store |

Error code 23, `CONFIGURATION_ERROR`, says none of the products registered in RevenueCat could be fetched from the store. RevenueCat is relaying a refusal rather than failing. Work these in order: the Paid Applications Agreement must be signed and active in App Store Connect, products must not be sitting in READY\_TO\_SUBMIT, tax and banking must be complete, the bundle ID must match across Despia, App Store Connect and RevenueCat, and product identifiers must match exactly including case. On Android, the app must have had an active release on a testing track.

There is no console in TestFlight, so render the user agent, whether the environment check passed, and the raw entitlement response into a debug panel you can toggle.

## Common questions

### Do I need to rebuild in a native framework?

No. Your Vue app ships as a real iOS and Android binary with the RevenueCat SDK compiled in, reached from JavaScript. A rebuild replaces your entire product to acquire one capability.

### Can I use the RevenueCat web SDK instead?

No. The web SDK bills through Stripe, which is the thing Apple rejects for digital goods. The native SDK compiled into the binary is what you want, called through the bridge.

### Does this apply to one-time purchases, or only subscriptions?

Both. Developers regularly report one-time Stripe purchases being rejected under the same guideline. The rule is about unlocking digital content, not about whether the charge repeats.

### Will my existing subscribers have to pay twice?

Not if you build the two-source access check. Guideline 3.1.3(b) exists so they do not, on condition that the same subscription is also purchasable in-app.

### Does this work with Nuxt?

Yes, with one rule: resolve access on the client. There is no user agent from the device and no native bridge during a server render, so server-rendered output is unentitled until the client resolves.

### Do I need Pinia for this?

Not strictly, but you need one owner of the value. If you are on Vuex or a plain reactive singleton, the same rule applies: one place resolves it, everything else reads it.

### Do I need webhooks?

No. Restoring entitlements on the device covers what the app displays, and one direct API call covers anything the server has to protect. Add a webhook later if you need real-time cancellation handling.

## When you want an answer from a person, not a model

An AI assistant does not tell you when it is unsure. It writes something confident, you spend a build finding out it was wrong, and you go around again.

If you are stuck and want a human to look at it, email `humans@despia.com`. A developer with five or more years of experience reads it and writes back about your actual setup, not a template. A person will sometimes tell you your approach is wrong, or that the roadblock is upstream of the thing you are debugging. That is the point of asking one.

## Get it on the stores

Take the Vue app you already built and ship it to iOS and Android without a CLI or a Mac. StoreKit and Play Billing arrive through RevenueCat with one JavaScript call, your web subscribers keep the access they paid for, and your existing codebase stays the single source of truth.

[See the setup docs at setup.despia.com](https://setup.despia.com)