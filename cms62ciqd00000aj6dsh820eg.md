---
title: "How to Get Lovable In-App Purchases Approved"
seoTitle: "How to Get Lovable In-App Purchases Approved"
seoDescription: "Get your Lovable app approved with real in-app purchases. StoreKit and Play Billing through RevenueCat, without losing your Stripe subscribers."
datePublished: 2026-07-29T12:30:55.985Z
cuid: cms62ciqd00000aj6dsh820eg
slug: how-to-get-lovable-in-app-purchases-approved
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/f8c061fb-9001-446e-b7e7-de1d967c1fd5.png
tags: revenuecat, lovable, storekit

---

To get approved selling digital content, your Lovable app needs three things: purchases running through StoreKit on iOS and Google Play Billing on Android, a paywall the reviewer can actually reach, and a working restore. Lovable's payments are Stripe and Paddle checkouts on the web, which is correct for a website and rejected inside an app. RevenueCat supplies the native layer, compiled into the app binary, and your existing Stripe subscribers keep their access at the same time under Guideline 3.1.3(b).

This is the whole path: what approval requires, how to build it on either Lovable backend, and what to check before you submit.

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
| Existing Stripe subscribers honoured, with in-app purchase also offered | The 3.1.3(b) condition |
| No pricing that steers iOS users to your website | Prohibited outside the US storefront |

Everything below is how to satisfy each line.

## What Apple requires, and why Lovable payments cannot satisfy it

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

Lovable's built-in payments are powered by Stripe and Paddle, they require a paid Lovable plan, and they run on Lovable's built-in backend for webhooks and subscription data. If you connected your own Supabase project, built-in payments are not available to you at all, so you wired Stripe up yourself. Either way the result is the same shape: a hosted web checkout with no StoreKit path and no Play Billing path, because those are native frameworks that compile into an app binary and a web checkout has nowhere to put them.

One correction worth making early, because it costs people a review cycle. This is not only about subscriptions. Developers report one-time Stripe purchases getting rejected under the same guideline, and the rule is about unlocking digital content, not about whether the payment recurs. A lifetime unlock sold through Stripe fails exactly like a monthly plan does.

The two rejection notices are worth recognising, because they mean opposite things. Selling in-app through a non-Apple processor gets you the standard message about accessing paid digital content by means other than in-app purchase. Removing the purchase flow and leaving only login gets you a different one, saying your app accesses digital content purchased outside the app and that content is not available to purchase using in-app purchase. The second is not asking you to remove more. It is asking you to add the in-app option alongside what you have.

## Keep the Stripe subscribers you already have

This is permitted, and it is written into the guidelines rather than being a workaround.

Guideline 3.1.3(b), Multiplatform Services, permits apps operating across multiple platforms to let users access content, subscriptions, or features they acquired in your app on other platforms or on your website, provided those same items are also available as in-app purchases within the app.

The condition is the whole thing. Honour the Stripe subscription, and also sell the same tier through StoreKit.

| Setup | Review outcome |
| --- | --- |
| Stripe subscription honoured, StoreKit paywall also offered | Approved |
| Stripe subscription honoured, no in-app purchase option | Rejected under 3.1.1 |
| Login-only companion app, no purchase UI at all | Usually rejected |
| Digital goods sold in-app through Stripe or Paddle | Rejected under 3.1.1 |

The third row surprises people. Teams shipping companion apps with no sign-up, no pricing screens, and no purchase UI still report repeated rejections, because access to paid digital content triggers the rule rather than the presence of a buy button. Guideline 3.1.3(c) carves out services sold directly to organisations for their employees or students, but consumer, single user, and family sales must use in-app purchase.

## Know which backend you are on first

Lovable projects split two ways, and this decides where half the work below goes.

| Your setup | Where subscribers live | Where the server check runs |
| --- | --- | --- |
| Lovable Cloud with built-in payments | Lovable's built-in backend tables | A Supabase edge function |
| Your own Supabase project | Your own tables, your own Stripe wiring | A Supabase edge function |

Lovable Cloud is Supabase underneath, so the edge function is the same place in both cases. What differs is only the record you read for existing subscribers. Built-in payments are unavailable on projects connected to your own Supabase, so if you are on Supabase you already own the Stripe integration and the subscription table and you know exactly where to look.

Nothing about the store side changes either way.

## The setup, start to finish

Four stages. The code is the fastest of them.

### 1\. Create the RevenueCat project

At app.revenuecat.com, create a project, then add your apps.

For iOS you need your bundle ID, an App Store Connect API key with App Manager access, and an in-app purchase key. For Android you need the same package name as your build plus a Google Play service account JSON with permissions granted in Play Console. Permissions can take time to propagate, so do this before you need it.

Then create your products in App Store Connect and Play Console, import them into RevenueCat, attach each to an entitlement, group them into packages, and put those packages in an offering. Mark one offering as default. An offering with no packages returns an empty list rather than an error, which is a confusing hour if you skip it.

### 2\. Connect the docs server before you prompt

AI builders invent native APIs constantly. The most common invention is `window.despia` as an environment check, which does not exist, so the condition is false on every device and every native call inside it is skipped in silence. Nothing errors. The feature is just dead, and you spend four builds debugging a runtime that is fine.

The Despia MCP server is a documentation server that hands your builder the real reference for the `despia-native` package. In Lovable: open Connectors, go to Chat connectors, click New MCP server, name it Despia, URL `https://setup.despia.com/mcp`, authentication set to none. Chat connectors are per-user, so everyone on the project connects their own.

### 3\. Point your agent at the reference

Ask Lovable for subscriptions on its own and it scaffolds a `revenuecat.ts` and a webhook handler, because Lovable's native path runs on Capacitor. Those files describe a setup you are not shipping. Point the agent at the guide instead and have it implement what is actually there.

```text
Read this page and implement it in this app exactly as written:
https://blog.despia.com/lovable-in-app-purchases-without-webhooks-revenuecat.md

Rules while you implement it:
- This app ships to iOS and Android with Despia. Native features come from the
  despia-native npm package.
- Detect the native runtime with the user agent. Never use window.despia, it
  does not exist and the check will be false on every device.
- Follow the code in the guide. Do not substitute your own RevenueCat pattern.
- Delete any revenuecat.ts and any RevenueCat webhook handler already scaffolded
  in this project. They describe a Capacitor setup we are not using.
- Do not add Capacitor, Cordova, or any second native project.
- Gate on the entitlement the store reports, never on a tier flag in our own
  database that our UI set.
- No webhooks and no new subscriptions table.

Two additions that are not in the guide:
- Before granting access, check the user's existing Stripe subscription record
  first, and only ask the store if that returns nothing. A customer who already
  subscribed on our website must never be shown a paywall.
- Pass our signed-in user id to RevenueCat as external_id, not an anonymous id.

When you are done, list every value I have to supply myself, and tell me which
files you changed.
```

The `.md` on the end of that URL matters. It serves the guide as plain markdown, which is what you want an agent reading instead of a rendered page full of navigation.

The deletion line earns its place. If the Capacitor scaffolding stays, you end up with two RevenueCat integrations in one project, one of which never runs, and the next agent that reads the codebase will happily extend the wrong one.

Asking which files changed gives you something to check, because an agent reporting success and an agent having written the call are not the same event. Search the project for the function it claims it called before you spend a build on it.

### 4\. Add the keys, then rebuild

In the Despia Editor open App, Integrations, RevenueCat, paste the iOS and Android public SDK keys, and trigger a new build.

The RevenueCat SDK compiles into the binary, so it cannot arrive over the air the way your Lovable web code does. Until you rebuild and install that build, the integration stays dormant even with the keys saved. Purchases fail quietly and entitlement checks return nothing while every dashboard screen looks correct. This is the single largest source of support tickets on this integration.

## Show the right thing to the right user

Keep the entitlement concept singular. A user is premium or not. Where the money came from belongs in one function, not scattered through your components.

This function decides what the interface shows. It is not security, and the next section is.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// display only: what screens to show, never what work to run
async function resolveAccess(user) {
  // Stripe subscribers keep their access on every platform
  if (user.subscription_status === 'active') return true

  // in the app, ask the store what this user actually owns
  if (isDespia) {
    const { restoredData } = await despia('getpurchasehistory://', ['restoredData'])
    return (restoredData ?? []).some(p => p.entitlementId === 'premium' && p.isActive)
  }

  return false
}

// outside the gate, so it exists before native fires it
window.onRevenueCatPurchase = () => resolveAccess(currentUser).then(setPremium)
```

Three things about this matter more than the code.

**It runs client-side, always.** If your project was created after Lovable moved to TanStack Start, parts of your app render on the server, where there is no user agent from the device and no native bridge. Resolve access in the browser and treat any server-rendered view as unauthenticated until it does.

**Order matters for the user, not the logic.** Checking the existing subscription first means a paying Stripe customer opening the app never sees a paywall flash before access resolves. That flash is what produces the review saying they were charged twice.

**Use your own user id.** Pass your Supabase auth user id, or your Lovable Cloud user id, to RevenueCat as the app user id. If RevenueCat generates anonymous ids instead, a user who subscribes in the app and later signs in on the web becomes two customers, and you merge them by hand.

## Block the expensive calls on the server

The function above reflects what the client claims. That is fine for hiding a screen and useless for protecting anything that costs you money, because a client can be modified. Anyone who opens the developer tools can flip a boolean and start calling your paid endpoints.

Lovable apps usually have something worth protecting: an AI generation, a paid export, a third-party API you are billed per call for. Check those when the request arrives.

```javascript
// Supabase Edge Function or Lovable backend function
// secret key stays server-side, never reaches the browser
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

async function runExpensiveThing(userId) {
  // Stripe subscriber, or a real store entitlement, and nothing else
  const entitled =
    (await getSubscription(userId))?.status === 'active' ||
    (await hasActiveEntitlement(userId, 'premium'))

  if (!entitled) return new Response('Forbidden', { status: 403 })
  return doTheWork(userId)
}
```

Four practicals worth getting right. The secret key stays server-side, which on Supabase means an Edge Function secret and never a `VITE_` prefixed variable, since anything prefixed that way is compiled into the bundle and readable by anyone. The `appUserId` must be the same id you passed as `external_id` when launching the paywall, or RevenueCat resolves a different customer and correctly reports no entitlement. URL-encode the id, because anonymous ids carry a prefix with a colon in it. And in sandbox this endpoint can read empty while the entitlement is genuinely live, so cross-check against RevenueCat's older subscriber lookup while testing.

Verify the active-entitlements route against RevenueCat's current documentation before you ship, since the API differs across endpoints and versions.

## Make sure the reviewer can find your paywall

This is where teams who did everything else right still get bounced, with a notice saying the subscription is not discoverable in review.

The reviewer has to reach a working purchase screen from a fresh install, with no context and no account. Four things stop them.

*   **Sign-in blocks the paywall.** If your paywall lives behind login, supply working credentials in the App Review Information section, and check them on a device that has never opened your app. Magic-link and email-verification sign-in makes the credentials useless, so give the reviewer a password account.
    
*   **Onboarding breaks before the paywall.** A quiz or setup flow that stalls on a slow network leaves the reviewer stuck upstream of the thing they are meant to test. Test the whole path on cellular from a clean install.
    
*   **The paywall is region-gated.** Review commonly runs from outside your main market. If your offering only returns products in some regions, they see an empty screen.
    
*   **Offerings fail to load for the reviewer specifically.** This happens, which is the reason to render a real error state rather than an empty view. A blank screen reads as a broken app.
    

Say where the paywall is in your review notes. One sentence naming the screen and how to reach it removes the entire category.

## Use offer codes, not your own promo codes

A common Lovable pattern that fails review: a field where users enter a code to unlock premium.

Apple's notice names it directly, saying the app unlocks or enables additional functionality with mechanisms other than in-app purchase, specifically that the app uses promo codes to unlock digital content. The instruction is to use an Apple-supported offer code instead.

So implement discounts as offer codes and promotional offers inside App Store Connect, attached to your subscription. Your Stripe coupon codes stay on the web where they always worked.

## What you can show in the app

| In-app element | Allowed? |
| --- | --- |
| Sign-in for existing Stripe subscribers | Yes, and it should be easy to find |
| Restore purchases | Yes, and a missing one is its own rejection |
| StoreKit paywall at your chosen price | Yes |
| Managing an existing subscription | Yes, without pricing or purchase calls to action |
| "Subscribe on our website and save" | No, outside the US storefront |
| A Stripe customer portal link that can upgrade | No, outside the US storefront |
| A link to your web checkout | US storefront only, and gated |

The customer portal row catches people. A portal that only cancels or updates a card is account management. A portal that can start or upgrade a paid plan is a purchase mechanism, and linking to it from inside the app is the violation.

## You can charge a different price in the app

Apple does not require price parity across platforms. If your Stripe tier is 9.99 and the store takes 15%, pricing the in-app tier at 12.99 is permitted, and many large apps do exactly that.

What you cannot do is explain why inside the app. The price is the price on each platform, and users who care will find the cheaper one without your help.

## Does the Epic ruling change any of this?

Partly, in one country, and it does not remove the need for native billing.

Since the 2025 Epic v. Apple injunction, apps on the United States storefront may include buttons and external links pointing at outside checkout, with no entitlement required. Stripe's own documentation now describes exactly this: for US digital goods on iOS you may redirect customers to an external Stripe Checkout page, and on Android in the US you may process payments in-app with a third-party processor. Both are marked US only, and that qualifier is the whole story.

Four things keep it from being the approval route:

*   It applies to the United States storefront only. Everywhere else the old restrictions stand, so a global build needs storefront gating and two purchase paths regardless.
    
*   On iOS you still cannot process the payment inside the app. The allowance covers linking out to a browser.
    
*   A browser handoff converts worse than a native purchase sheet with Face ID and a stored card, and that gap is often larger than the commission.
    
*   The current zero commission on US external purchases exists because a court has not set a rate, not because a rate was decided.
    

Rules current as of July 2026 and actively moving. Check Apple's and Google's own policy pages before building around them.

## What to do with subscribers who are already paying you

**You cannot migrate them automatically.** There is no mechanism to move a Stripe subscription into StoreKit on a user's behalf. A subscription is a contract with a processor, and switching means the user cancels one and starts the other themselves.

**You should not push them to switch.** Stripe subscribers at roughly 3% are your best-margin customers. Moving them to a store tier costs you money and gains nothing, because 3.1.3(b) already lets them use the app.

**Cancel-and-rebuy creates refund work.** If you prompt someone to move, you own the overlap, the refund request, and the confusion while both are briefly active.

## Before you submit

Run these on a physical device, from a clean install, on cellular.

*   Buy the subscription in sandbox and confirm premium unlocks.
    
*   Delete the app, reinstall, sign in, and confirm restore returns your entitlement.
    
*   Sign in as an existing Stripe subscriber and confirm no paywall appears.
    
*   Open the paywall with the demo account you are giving Apple.
    
*   Confirm your in-app purchase products are attached to the version you are submitting, not merely created in App Store Connect.
    
*   Check the paywall shows price, billing period, terms, privacy, and restore.
    
*   Confirm no screen inside the app links to Stripe Checkout or a portal that can upgrade.
    
*   Write one line in the review notes saying where the paywall is.
    

Products have to be submitted together with the app version. Creating them is not enough, and this is the most common reason a technically correct billing setup still gets bounced.

## When something does not work

| What you see | Most likely cause |
| --- | --- |
| Entitlement check empty, keys are saved | No rebuild since the keys were added |
| Paywall button does nothing, no error | Environment check false, so the call never ran |
| `window.despia is not a function` | That API does not exist, use the user agent check |
| `Error fetching offerings`, code 23 | Store cannot return the products yet |
| Offerings load but contain no packages | Products not attached to an offering |
| Purchase completes, access does not unlock | Callback assigned inside the gate instead of at module scope |
| Existing Stripe subscriber sees a paywall | Access check only reads the store |
| Works in the Lovable preview, not in the app | Preview is not the deployed build the app loads |

Error code 23, `CONFIGURATION_ERROR`, says that none of the products registered in the RevenueCat dashboard could be fetched from App Store Connect or the Play Store. RevenueCat is relaying a refusal rather than failing. Work these in order: the Paid Applications Agreement must be signed and active in App Store Connect, products must not be sitting in READY\_TO\_SUBMIT, tax and banking must be complete, the bundle ID must match across Despia, App Store Connect and RevenueCat, and product identifiers must match exactly including case. On Android, the app must have had an active release on a testing track, which is the most common Android-only cause.

Debug on a real device rather than a simulator, and remember there is no console in TestFlight. Render the user agent, whether the environment check passed, and the raw entitlement response into a debug panel you can toggle.

## Common questions

### Can I keep using Stripe at all?

Yes, on the web, and for physical goods or real-world services anywhere. What you cannot do is sell digital content through it inside the iOS or Android app. Most approved apps run both: Stripe on the web, StoreKit and Play Billing in the app.

### Does this apply to one-time purchases, or only subscriptions?

Both. Developers regularly report one-time Stripe purchases being rejected under the same guideline. The rule is about unlocking digital content, not about whether the charge repeats.

### Will my existing subscribers have to pay twice?

Not if you build the two-source access check. Guideline 3.1.3(b) exists so they do not, on condition that the same subscription is also purchasable in-app.

### Does Lovable support StoreKit or Google Play Billing?

No. Lovable's payments are web checkouts through Stripe and Paddle, and its built-in payments run on Lovable's own backend. Native billing has to come from the native layer around your app.

### Does this work with my own Supabase project?

Yes, and it is arguably simpler, because you already own the subscription table and know exactly what to read. Lovable's built-in payments are unavailable on projects connected to your own Supabase, so if you are on Supabase you wired Stripe yourself and that record is yours to query.

### Do I have to rebuild my app in React Native or Expo?

No. The missing piece is native billing, not your application. A rebuild replaces the whole product to acquire one capability, and it takes your Stripe subscribers through a migration for no reason.

### Do I need a subscription group?

Yes, for auto-renewable subscriptions. A subscription group holds tiers and durations of the same service, a user can hold only one active subscription per group, and a free trial can be redeemed only once within a group. Putting monthly and annual in separate groups lets one person subscribe to both and claim two free trials.

### Can I offer a free trial?

Yes, through an introductory offer on the subscription in App Store Connect. It is configured on the store side rather than in your code, and RevenueCat reports the trial state back through the entitlement. Play has an equivalent.

### Do I need webhooks for this?

No. Restoring entitlements on the device covers what the app displays, and one direct API call covers anything the server has to protect. A webhook plus a second subscription table is another source of truth you then maintain forever. Add one later if you need real-time cancellation handling.

### How long does this take?

The code is an afternoon. The slow parts are the RevenueCat project, store product configuration, and the build. Budget a day, and expect the store side to take longer than the integration.

## When you want an answer from a person, not a model

Shipping your first app is stressful, and an AI builder makes it worse in a specific way: it does not tell you when it is unsure. It tells you what you asked to hear, confidently, and you find out four builds later.

If you are stuck on something with Despia and Lovable and you want a human to look at it, email `humans@despia.com`. A developer with five or more years of experience reads it and writes back about your actual setup, not a template.

Worth knowing what you are asking for. A person will sometimes tell you your approach is wrong, that the roadblock is upstream of the thing you are debugging, or that the feature you want is not the one that gets you approved. That is the point of asking a human.

## Get it on the stores

Take the Lovable app you already built and ship it to iOS and Android without a CLI or a Mac. StoreKit and Play Billing arrive through RevenueCat with one JavaScript call, your Stripe subscribers keep the access they paid for, and your Lovable app stays the single source of truth.

[See the setup docs at setup.despia.com](https://setup.despia.com)