---
title: "Lovable Loginless Mobile App: Anonymous Login, IAP, Push"
seoTitle: "Lovable Loginless Mobile App: Anonymous Login, IAP, Push"
seoDescription: "Build a loginless Lovable app with anonymous login on iOS and Android, then add in-app purchases and push notifications with RevenueCat and Despia."
datePublished: 2026-07-11T10:30:05.229Z
cuid: cmrg83s0500000akp8g5h0dgq
slug: lovable-loginless-mobile-app-anonymous-login-iap-push
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/ee12ac3e-1f53-4b48-aa55-6aa53073f071.png
tags: mobile-app-development, login, auth, push-notifications, in-app-purchase, lovable

---

A signup screen is the single biggest drop-off point in a mobile funnel, and Apple's review guideline 5.1.1 explicitly pushes back on apps that force registration before delivering value. A loginless mobile app skips it entirely, and gives up nothing: anonymous login runs invisibly in the background, and a Lovable app shipped with Despia gets accounts, paid subscriptions, and push notifications on top of an identity the user never sees.

The post covers anonymous login first, a device-scoped ID stored in Despia Storage Vault mapped to an anonymous account in your Supabase database, then everything it unlocks: RevenueCat in-app purchases keyed to that account, push notifications targeted at it, compliant account deletion, and account switching.

## Why loginless converts better

Every field on a signup form is a place to lose someone who has not yet seen a reason to stay. A loginless mobile app moves that moment: the user opens the app and is inside it, using it, before anyone asks them for anything. For consumer apps that changes the shape of the whole funnel.

**More paying users.** The paywall is the first thing you ask for, not the second. A user who hits your subscription prompt has already experienced the app, which is a fundamentally better moment to sell than a registration wall in front of a product they have not touched. And because the purchase belongs to the store account, buying takes one Face ID confirmation, no account required first.

**Fewer day-one uninstalls.** The most common reason a fresh install dies is a gate at first open. Remove the gate and the app gets its shot at forming a habit, which is what actually keeps it on the phone.

**More real signups, later.** Loginless does not mean no accounts ever. It means the signup happens when the user has something worth keeping, saved data, an active subscription, a streak. At that point creating a login is not friction, it is protection, and conversion on that prompt reflects it. The users who never link were the ones who would have bounced off the signup wall anyway, except now some of them paid you first.

**Linking is optional, and it unlocks the web.** The anonymous account is a full account row, so when the user wants to sign in on their laptop, keep access across a platform switch, or just attach an email, you offer a link step: credentials get written onto the existing row, the account ID stays the same, and their subscription, push linkage, and data come with them. Nothing about going loginless first closes the door on accounts later.

## The architecture in one paragraph

On first launch, the app generates a UUID and stores it in Storage Vault. A Supabase function creates an anonymous account row for that UUID and returns an account ID. That account ID becomes the RevenueCat `external_id` for purchases and the external user ID for push. The user gets a persistent account, a working subscription, and targeted notifications, and they never typed an email address.

The reason Storage Vault is the right anchor and not `localStorage`: vault data survives app uninstall and reinstall. On iOS it is backed by iCloud Key Value Store, which means the ID is actually scoped to the Apple ID and syncs across the user's devices. That is exactly how App Store purchases are scoped too, so your identity anchor and Apple's purchase anchor travel together. On Android, Key/Value Backup restores the vault through the user's Google account, which is where Play purchases live. The alignment is not a coincidence you got lucky on, it is the reason this architecture is safe.

## Tell Lovable the real API, or it will invent one

Before any code: Lovable will happily generate calls to APIs that do not exist, `window.despia` objects, made-up purchase methods, invented callbacks. The Despia SDK is one function, `despia()`, installed with `npm install despia-native`, and the only correct environment check is the user agent. Put the patterns from this post directly into your Lovable prompt so the generated code matches the real runtime, and if the output contains an API you cannot find at setup.despia.com, it is hallucinated.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')
```

## Set up anonymous login

`readvault://` throws when the key does not exist, so a try/catch is the branch between a returning user and a first launch. Two details make this production-grade rather than demo-grade. The bridge calls on the boot path get a short timeout, so a dead WebView bridge degrades to a fresh ID instead of hanging app boot on a promise that never settles. And the vault write happens before the backend call, so a crash between the two can never strand an account the device has no way to find again.

```javascript
// resolve with a fallback after 2s if the native bridge never answers
const withTimeout = (promise, ms = 2000, fallback = null) =>
    Promise.race([promise, new Promise(r => setTimeout(() => r(fallback), ms))])

async function getAccountId() {
    if (!isDespia) {
        // web fallback: cookie or localStorage id, weaker but fine for browser users
        return resolveAccount(getWebDeviceId())
    }

    let deviceId
    try {
        const data = await withTimeout(despia('readvault://?key=deviceId', ['deviceId']))
        deviceId = data?.deviceId
    } catch {
        // key not found on first launch, fall through and create one
    }

    if (!deviceId) {
        deviceId = crypto.randomUUID()
        await withTimeout(despia(`setvault://?key=deviceId&value=${deviceId}&locked=false`))
    }

    // one idempotent upsert serves first launches and returning users alike
    return resolveAccount(deviceId)
}
```

On the Supabase side this is a single table mapping `device_id` to `account_id` with a unique constraint on `device_id`, plus whatever app data hangs off the account. `resolveAccount` is an Edge Function or RPC doing an insert-or-return upsert against that constraint, which is what makes it safe to call from both branches, from a restored vault on a new phone, and from two devices racing their first launch on the same Apple ID. Lock the table down with RLS so the anon key can only reach it through the function.

## Wire purchases to the anonymous account

Pass the account ID as `external_id` on every RevenueCat call. That makes it the `app_user_id` RevenueCat reports on, so your backend can grant and revoke access against a real row even though the user never registered.

```javascript
if (isDespia) {
    despia(`revenuecat://launchPaywall?external_id=${encodeURIComponent(accountId)}&offering=default`)
}
```

Check entitlements by querying the native store and filtering for active.

```javascript
async function checkEntitlements() {
    if (!isDespia) return
    const data   = await despia('getpurchasehistory://', ['restoredData'])
    const active = (data?.restoredData ?? []).filter(p => p.isActive)
    // set state both ways, an expired subscription must re-lock the UI
    setPremium(active.some(p => p.entitlementId === 'premium'))
}

checkEntitlements()
window.onRevenueCatPurchase = checkEntitlements
```

Here is the fact that makes the whole loginless model trustworthy: the purchase does not actually belong to your account ID. It belongs to the Apple ID or Google account at the store level. Your account ID is a label RevenueCat attaches for reporting and webhooks. `getpurchasehistory://` queries the native store directly, so even if your database burned down, an entitlement check on the device would still return the truth. Your account row is for data and push targeting. The store is the source of truth for access.

## Protect Edge Functions without a session

The client-side check is right for showing and hiding UI, and it is not sufficient for anything a manipulated client could walk away with. Any Edge Function that costs you money or serves protected content needs a server-side check at request time. No login is required for this either, because the check is keyed on the app user ID, not on a credential. The client sends its account ID, and a Supabase Edge Function asks RevenueCat directly whether that ID is entitled. The RevenueCat secret key lives in your Edge Function secrets and never reaches the client.

```javascript
// inside a Supabase Edge Function, the secret key never ships to the client
async function hasActiveEntitlement(accountId, entitlementId) {
    const id  = encodeURIComponent(accountId)
    const res = await fetch(
        `https://api.revenuecat.com/v2/projects/${PROJECT_ID}/customers/${id}/active_entitlements`,
        { headers: { Authorization: `Bearer ${RC_SECRET_KEY}` } }
    )
    if (res.status === 404) return false    // RevenueCat has never seen this customer
    if (!res.ok) throw new Error(`RevenueCat ${res.status}`)
    const { items = [] } = await res.json()
    return items.some(e => e.entitlement_id === entitlementId)
}

// gate the protected route
if (!(await hasActiveEntitlement(accountId, 'premium'))) {
    return { error: 'subscription required', status: 402 }
}
```

The account ID in this check must be the same value you passed as `external_id` at paywall launch. If they diverge, RevenueCat resolves a different customer and the check correctly returns no entitlement. No webhook pipeline, no subscription table to keep in sync. RevenueCat already knows who is subscribed, so ask it at the moment the answer matters, and cache the result per user for a short window if the function is hot.

Without a session, the account ID is the entire credential, the same model as Firebase anonymous auth. That is fine for a consumer app as long as you treat it like one. `crypto.randomUUID()` gives you 122 random bits, so the ID is unguessable, and it must stay that way: never put it in URLs, logs, error messages, or anything user-visible. Send it in a header or body over HTTPS only. If someone exfiltrates another user's ID they can use that entitlement, which is the same exposure a stolen session token creates in a logged-in app, so you are not weaker in kind, you are just missing the log-in-again recovery path. For a step up, store the ID with `locked=true` so reading the credential from the vault requires Face ID.

## Push notifications without an email address

The same account ID keys your push setup. Despia compiles the OneSignal SDK into the binary and registers the device at launch, so the only thing your web layer does is tell OneSignal which account this device belongs to. Link on every launch, right after the identity bootstrap resolves.

```javascript
function linkPush(accountId) {
    if (!isDespia) return
    despia(`setonesignalplayerid://?user_id=${encodeURIComponent(accountId)}`)
}
```

The `user_id` becomes the `external_id` in OneSignal, and your backend targets it with the same identifier it already stores for RevenueCat and your database. One ID, three systems.

```javascript
// inside a Supabase Edge Function, the REST API key never reaches the client
async function sendPush(accountId, title, message) {
    await fetch('https://onesignal.com/api/v1/notifications', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            Authorization: `Basic ${Deno.env.get('ONESIGNAL_REST_API_KEY')}`,
        },
        body: JSON.stringify({
            app_id: Deno.env.get('ONESIGNAL_APP_ID'),
            include_external_user_ids: [accountId],
            headings: { en: title },
            contents: { en: message },
        }),
    })
}
```

Notice what is absent: no permission-gated signup, no email capture to build an audience. The anonymous account is the audience. You can message a user who has never told you anything about themselves, which for most consumer apps is the entire point of push.

## Account deletion that does not orphan purchases

Apple requires apps that create accounts to offer account deletion, and anonymous accounts do not exempt you if you store data server-side. The naive implementation deletes the row and mints a fresh identity, and it has a nasty consequence: RevenueCat entitlements are keyed to the account ID you passed as `external_id`, so a new ID orphans the purchase until a restore syncs through, and in the gap a paying user sees locked features on every server-side check, which is how refund requests and one-star reviews get written.

The pattern that avoids the gap entirely: anonymize instead of delete. An Edge Function wipes every personal column on the account row and every row hanging off it, revert the account to a pristine anonymous state, and keep the `id` and `device_id` unchanged. RevenueCat, push, and your data model all keep pointing at a valid account, so purchases survive with zero transfer choreography. This satisfies the store rule, which requires deleting the account and its associated personal data, and an anonymous row holds none: a random device UUID with no identifying fields attached to it is not personal data. Reserve hard deletion for web-only accounts that were never bound to a device, where there is no purchase identity to preserve.

```javascript
async function deleteAccountData() {
    // Edge Function wipes all personal columns and attached rows,
    // keeps id and device_id so external_id stays valid
    await supabase.functions.invoke('anonymize-account')

    // same account, now a pristine guest, access was never at risk
    await checkEntitlements()
}
```

Deletion is destructive, so confirm it with something stronger than a tap: on native, a `locked=true` vault key makes Face ID or Touch ID the confirmation step, and on web, type-to-confirm does the job. On the next launch the normal anonymous login finds the same account by device ID and the user continues as a fresh guest with their purchases intact. Push needs no re-link either, the external user ID never changed. And if anything ever does go wrong with your database, the store is still the fallback: `getpurchasehistory://` answers from the Apple ID or Google account, which owns the purchase no matter what your rows say.

## Account switching, and why you should not fight the store

Sooner or later you add optional login for cross-device sync, and the question arrives: what happens when someone logs out and logs into a different account on the same phone? The answer starts with a constraint that no app architecture can change. The store sells to the Apple ID or Google account, not to your app account, and Apple does not allow one Apple ID to hold two active subscriptions to the same product. Two app accounts on one device can never each own their own copy of your subscription. If the second account tries to buy while the device's store account already has it, the store reports the existing purchase instead of charging again, and RevenueCat applies its configured transfer behavior. This applies to new purchases, not just restores.

RevenueCat has a setting for that moment, restore behavior, and the guidance is one line: leave it on the default, Transfer to new App User ID. In this architecture it is a safety net, not a mechanism you build on. Despia passes your account ID as `external_id` on every call and the ID stays stable for the life of the device identity, so in steady state the receipt and the account never diverge and no transfer ever fires. The default earns its keep in exactly two moments. If the vault is ever lost and a fresh account ID gets minted, the user taps Restore Purchases, the receipt posts under the new ID, and RevenueCat re-points the entitlement to it, recovery with no support ticket. And on account switching, the entitlement follows whoever last synced on the device that owns the purchase, which is the store's own rule. In Despia terms, the moment a transfer can happen is any receipt sync under the new ID: a purchase, a paywall checkout, or a Customer Center restore. `getpurchasehistory://` never triggers one, it reads the local store and does not talk to RevenueCat, which is why it stays instant and offline-capable. Your backend sees a `TRANSFER` webhook when a transfer does happen, and there is nothing to build for it. The one configuration to never pair with anonymous accounts is Keep with original App User ID: it blocks the restore-recovery path, so a user who reinstalls after losing the vault is locked out of what they paid for.

Two rules make optional login safe on top of this:

**Signup is an upgrade, never a new account.** When the anonymous user creates credentials, attach them to the account row they already have. The account ID does not change, so nothing transfers, the subscription and push linkage stay exactly where they were. Creating a second parallel account at signup is the mistake that causes most "my subscription disappeared" tickets. Watch for this specifically in Lovable-generated auth flows, the default Supabase Auth signup creates a brand new user rather than upgrading the anonymous one, so prompt for the upgrade-in-place behavior explicitly.

**On a second device, gate on the account, not only the local store.** `getpurchasehistory://` answers from the device's own store, so on a user's new phone, and always across platforms, it comes back empty even though their account is entitled. When a login is active, the client rule is local store check or account check via your backend, whichever grants. This is also why the post-purchase link prompt earns its place: a signed-in user who restores on a new device posts the receipt under the account that already owns it, so nothing transfers and nothing can go wrong. The transfer machinery only exists for the user who never linked, and the prompt exists to make that user rare.

**Switching moves the entitlement, and that is the correct behavior.** If a user signs into a different existing account on a device whose store account owns the purchase, the entitlement transfers to that account on the next sync, and transfers back when the original account syncs again. Your backend sees `TRANSFER` events bounce between the two IDs. Do not treat that as a bug to suppress, it is the store enforcing one subscription per store account. If your product genuinely cannot tolerate an entitlement following the device, that is the signal you have outgrown loginless: switch to mandatory login before purchase and the Keep with original behavior, which is a different app than the one this post describes.

**What a restore changes, and what it must not.** When a restore lands while a different account is signed in, exactly one thing moves: the receipt's owner inside RevenueCat. Do not swap the device ID in the vault, it is identity plumbing for the device's anonymous account and has nothing to do with purchase ownership. Do not touch the session, a restore is not a login event. And never merge account data because a receipt moved, the two accounts may be two people sharing a device, and a receipt cannot tell you which. On the backend the `TRANSFER` webhook carries `transferred_from` and `transferred_to`, arrays of account IDs identifying who lost the purchases and who now owns them, with no product or entitlement fields, since a transfer moves everything between the two customers. The loser also gets no follow-up EXPIRATION event, the TRANSFER is the revocation signal. So the entire correct response is entitlement bookkeeping: mark every account in `transferred_from` unentitled, re-query RevenueCat for the `transferred_to` ID rather than computing a delta from the event, or do nothing at all if you query RevenueCat live. Merging accounts is a separate, explicit, user-initiated feature, or it is a bug.

**The consent step polished apps add.** The `getpurchasehistory://` response includes `externalUserId`, the account ID that made the purchase. Compare it to the current signed-in account before offering a restore: if they differ, show the choice, this purchase belongs to another profile on this device, move it to this account or switch to that one. The silent transfer is correct behavior, the consented transfer is better UX, and it prevents the where-did-my-subscription-go ticket from the other account.

The rule that summarizes all of it: your app accounts are a layer on top of store identity, not a replacement for it. Design account switching around what the store will actually do, not around what your database schema implies.

## The failure modes to design for

**The vault can be unavailable.** iCloud can be signed out, and Android Key/Value Backup is best-effort across OEMs. Treat the vault as an identity cache, not as the entitlement store. If the read throws on a device that has actually used the app before, you generate a new ID, and the user recovers access through restore purchases because the store remembers even when the vault does not. Never gate premium features on the vault alone.

**Two devices, one Apple ID, one vault.** iCloud KVS syncs, so a user's iPhone and iPad resolve to the same account. That is correct behavior, one person is one account, and it matches how their subscription works anyway. The upsert against the unique `device_id` constraint already handles the race when both launch at once. One honest edge remains: iCloud KVS sync is not instant, so a brand-new second device can beat the sync and mint its own ID before the first device's key arrives. That occasionally leaves one human with two account rows, which is harmless for access, the store owns the purchase and Restore Purchases recovers it under either ID, but expect it in your data rather than treating it as a bug.

**Do not reach for device fingerprinting.** A self-generated UUID stored in the vault is a random identifier you created, which is fine. Deriving an ID from hardware signals to track the user is exactly what Apple's fingerprinting rules prohibit, and you do not need it.

**Trial abuse is mostly solved for free.** Because the vault survives uninstall and reinstall, a `hasUsedTrial` flag next to the device ID closes the delete-and-redownload loophole without any server work.

## Where this fits

This is the right default for consumer apps built in Lovable where signup friction is the enemy: utilities, content apps, games, anything where the purchase is the account. You keep the conversion of a zero-friction first open, and you keep the option of full accounts and web access whenever a user wants them. When users eventually want email login for cross-device sync, add it as an upgrade to the existing account row, follow the two switching rules above, and the account ID stays the spine of the system from anonymous first launch to full accounts.

## Get it on the stores

Take the Lovable app you already built and ship it to iOS and Android with native purchases, push, and persistent storage wired in through one JavaScript call. Code signing and submission run from the browser.

[See the setup docs at setup.despia.com](https://setup.despia.com)