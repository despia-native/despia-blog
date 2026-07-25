---
title: "Base44 Loginless Mobile App: Anonymous Login"
seoTitle: "Base44 Loginless Mobile App: Anonymous Login"
seoDescription: "Build a loginless Base44 app with anonymous login on iOS and Android, then add in-app purchases and push notifications with RevenueCat and Despia."
datePublished: 2026-07-11T10:27:55.042Z
cuid: cmrg80zkf000209ky33wqdln1
slug: base44-loginless-mobile-app-anonymous-login
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/01318ac2-960b-47e8-99d0-4d1cc77ec9d3.png
tags: mobile-app-development, login, auth, push-notifications, in-app-purchase, base44

---

A signup screen is the single biggest drop-off point in a mobile funnel, and Apple's review guideline 5.1.1 explicitly pushes back on apps that force registration before delivering value. A loginless mobile app skips it entirely, and gives up nothing: anonymous login runs invisibly in the background, and a Base44 app shipped with Despia gets accounts, paid subscriptions, and push notifications on top of an identity the user never sees.

The post covers anonymous login first, a device-scoped ID stored in Despia Storage Vault mapped to an anonymous account in your Base44 backend, then everything it unlocks: RevenueCat in-app purchases keyed to that account, push notifications targeted at it, compliant account deletion, and account switching.

## Why loginless converts better

Every field on a signup form is a place to lose someone who has not yet seen a reason to stay. A loginless mobile app moves that moment: the user opens the app and is inside it, using it, before anyone asks them for anything. For consumer apps that changes the shape of the whole funnel.

**More paying users.** The paywall is the first thing you ask for, not the second. A user who hits your subscription prompt has already experienced the app, which is a fundamentally better moment to sell than a registration wall in front of a product they have not touched. And because the purchase belongs to the store account, buying takes one Face ID confirmation, no account required first.

**Fewer day-one uninstalls.** The most common reason a fresh install dies is a gate at first open. Remove the gate and the app gets its shot at forming a habit, which is what actually keeps it on the phone.

**More real signups, later.** Loginless does not mean no accounts ever. It means the signup happens when the user has something worth keeping, saved data, an active subscription, a streak. At that point creating a login is not friction, it is protection, and conversion on that prompt reflects it. The users who never link were the ones who would have bounced off the signup wall anyway, except now some of them paid you first.

**Linking is optional, and it unlocks the web.** The anonymous account is a full `Account` record, so when the user wants to sign in on their laptop, keep access across a platform switch, or just attach an email, the template's `authLinkAccount` writes the credentials onto the existing record, the account ID stays the same, and their subscription, push linkage, and data come with them. Nothing about going loginless first closes the door on accounts later.

## The architecture in one paragraph

On first launch, the app generates a UUID and stores it in Storage Vault. A Base44 backend function upserts an anonymous `Account` entity for that UUID and returns a JWT you mint yourself. That JWT authenticates every protected function call, and the account ID inside it becomes the RevenueCat `external_id` for purchases and the external user ID for push. The user gets a persistent account, a working subscription, and targeted notifications, and they never typed an email address.

One Base44-specific reality shapes this design. Base44's built-in auth cannot mint a session in code: `User.create` returns 405 even from service-role functions, there is no token-exchange or session-minting API, and sessions only fall out of the hosted browser flow, which an anonymous user never enters. So anonymous accounts on Base44 run on your own auth: an `Account` entity you manage, deny-all RLS, and a hand-rolled JWT, the same custom auth system that native Google sign-in on Base44 requires. You do not have to build it. The template at [github.com/despia-native/base44-native-boilerplate](https://github.com/despia-native/base44-native-boilerplate) is a working Base44 project with the whole system already in place: the `deviceSignIn` function for anonymous accounts, Google, Apple, and password login, push, and compliant account deletion, all on one JWT. Clone it, run `base44 dev`, and pushed changes sync back into the Base44 Builder. Every snippet in this post is a condensed version of code that ships in that repo.

The reason Storage Vault is the right anchor and not `localStorage`: vault data survives app uninstall and reinstall. On iOS it is backed by iCloud Key Value Store, which means the ID is actually scoped to the Apple ID and syncs across the user's devices. That is exactly how App Store purchases are scoped too, so your identity anchor and Apple's purchase anchor travel together. On Android, Key/Value Backup restores the vault through the user's Google account, which is where Play purchases live. The alignment is not a coincidence you got lucky on, it is the reason this architecture is safe.

## Tell Base44 the real API, or it will invent one

Before any code: Base44 will happily generate calls to APIs that do not exist, `window.despia` objects, made-up purchase methods, invented callbacks. The Despia SDK is one function, `despia()`, installed with `npm install despia-native`, and the only correct environment check is the user agent. Put the patterns from this post directly into your Base44 prompt so the generated code matches the real runtime, and if the output contains an API you cannot find at setup.despia.com, it is hallucinated.

```javascript
import despia from 'despia-native'

const isDespia = navigator.userAgent.toLowerCase().includes('despia')
```

## Set up anonymous login: the Account entity and deviceSignIn

`Account` is your user table. Anonymous accounts live in the same entity as real logins: the vault UUID goes in `device_id`, `is_anonymous` is true, and `email` holds a synthetic `device-<uuid>@anon.local` address so the unique-email model stays intact for when the user later upgrades. The deny-all `rls` block is not optional: your users hold your JWT, not a platform session, so they are anonymous to Base44's data layer, and any permissive rule would expose the entity to the whole internet.

```json
{
  "name": "Account",
  "type": "object",
  "properties": {
    "email":          { "type": "string", "description": "Unique identity. Synthetic device-<uuid>@anon.local for anonymous accounts" },
    "full_name":      { "type": "string" },
    "password_hash":  { "type": "string", "description": "Empty for Google, Apple, and anonymous accounts" },
    "google_id":      { "type": "string" },
    "apple_id":       { "type": "string" },
    "device_id":      { "type": "string", "description": "Stable UUID from the Storage Vault, the anonymous identity" },
    "is_anonymous":   { "type": "boolean", "default": false },
    "email_verified": { "type": "boolean", "default": false },
    "role":           { "type": "string", "enum": ["user", "admin"], "default": "user" },
    "last_login_at":  { "type": "string", "format": "date-time" }
  },
  "required": ["email"],
  "rls": {
    "create": { "user_condition": { "role": "__service_only__" } },
    "read":   { "user_condition": { "role": "__service_only__" } },
    "update": { "user_condition": { "role": "__service_only__" } },
    "delete": { "user_condition": { "role": "__service_only__" } }
  }
}
```

Every operation is gated on a role no user can ever have, which denies all of them for everyone. Backend functions bypass RLS through `base44.asServiceRole`, and that is the single sanctioned data path: frontend, to a function, which verifies your JWT, then touches the database.

The `deviceSignIn` function finds or creates the anonymous account and mints your JWT. `signJwt` and `verifyJwt` are hand-rolled HS256 helpers signed with a `JWT_SECRET` env var, and both live in the template so you do not have to type them.

```javascript
// Base44 function: deviceSignIn, condensed from the template
import { createClientFromRequest } from 'npm:@base44/sdk'

const { device_id } = await req.json()
if (!/^[A-Za-z0-9._-]{8,128}$/.test(device_id ?? '')) {
    return Response.json({ error: 'A valid device_id is required' }, { status: 400 })
}

const base44 = createClientFromRequest(req)
// the device id IS the identity, the synthetic email keeps the unique-email model intact
const email = `device-${device_id.toLowerCase()}@anon.local`

let [account] = await base44.asServiceRole.entities.Account.filter({ device_id })
if (!account) {
    account = await base44.asServiceRole.entities.Account.create({
        email, full_name: 'Guest', device_id, is_anonymous: true,
        role: 'user', last_login_at: new Date().toISOString(),
    })
}

const token = await signJwt(
    { sub: account.id, email: account.email, role: account.role },
    Deno.env.get('JWT_SECRET')
)
return Response.json({ token, account_id: account.id })
```

The client side ties it to the vault. `readvault://` throws when the key does not exist, so a try/catch is the branch between a returning user and a first launch.

```javascript
const DEVICE_KEY = 'app_device_id'

// read the device id from the vault, or create and persist one on first run
async function getOrCreateDeviceId() {
    try {
        const data = await despia(`readvault://?key=${DEVICE_KEY}`, [DEVICE_KEY])
        if (data?.[DEVICE_KEY]) return data[DEVICE_KEY]
    } catch {
        // key not found on first run, fall through to create
    }
    const id = crypto.randomUUID()
    await despia(`setvault://?key=${DEVICE_KEY}&value=${id}&locked=false`)
    return id
}

async function signInWithDevice() {
    // web fallback: cookie or localStorage id, weaker but fine for browser users
    const deviceId = isDespia ? await getOrCreateDeviceId() : getWebDeviceId()
    const { data } = await base44.functions.invoke('deviceSignIn', { device_id: deviceId })
    localStorage.setItem('app_auth_token', data.token)
    return data.account_id
}
```

Run this on every launch. The JWT in `localStorage` is disposable, the vault UUID can always mint a new one, so a purged WebView costs nothing, and a restored vault on a new phone hits the same `device_id` filter and lands on the same account instead of minting a duplicate. The template goes one step further and mirrors the session token itself into the vault under a second key, which means even a linked login survives reinstall without the user signing in again. Two options worth knowing from the template: set `locked=true` on the vault write and restoring the session requires Face ID, and wrap the vault calls in a short timeout so a dead bridge falls through to a fresh sign-in instead of hanging the launch spinner, the pattern the template documents in ANTI\_FREEZE.md.

## Wire purchases to the anonymous account

Pass the account ID returned by `signInWithDevice` as `external_id` on every RevenueCat call. That makes it the `app_user_id` RevenueCat reports on, so your backend can grant and revoke access against a real entity even though the user never registered.

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

The template ships this whole client layer ready-made: `src/lib/revenuecat.js` has `checkEntitlements`, `launchPaywall` with the Web Purchase Link fallback for browser users, and `openCustomerCenter`, a `PremiumProvider` keeps app-wide premium state in sync with the store and handles the `onRevenueCatPurchase` and `onRevenueCatCenter` events, and the entitlement ID, offering, and web purchase URL are all set in one config file, `src/config/app-config.js`.

Here is the fact that makes the whole loginless model trustworthy: the purchase does not actually belong to your account ID. It belongs to the Apple ID or Google account at the store level. Your account ID is a label RevenueCat attaches for reporting and webhooks. `getpurchasehistory://` queries the native store directly, so even if your database burned down, an entitlement check on the device would still return the truth. Your account entity is for data and push targeting. The store is the source of truth for access.

## Protect backend functions without a session

The client-side check is right for showing and hiding UI, and it is not sufficient for anything a manipulated client could walk away with. Any backend function that costs you money or serves protected content needs a server-side check at request time. On Base44 the guard has two layers: verify your JWT to know which account is calling, then ask RevenueCat whether that account is entitled. The account ID comes out of the verified token, never out of the request body, so a caller cannot claim someone else's subscription. The RevenueCat secret key lives in your function secrets and never reaches the client.

```javascript
// Base44 function: guard a paid endpoint
const token   = req.headers.get('x-app-token')
const payload = await verifyJwt(token, Deno.env.get('JWT_SECRET'))
if (!payload) return Response.json({ error: 'unauthorized' }, { status: 401 })

if (!(await hasActiveEntitlement(payload.sub, 'premium'))) {
    return Response.json({ error: 'subscription required' }, { status: 402 })
}
// entitled, run the protected work

async function hasActiveEntitlement(accountId, entitlementId) {
    const id  = encodeURIComponent(accountId)
    const res = await fetch(
        `https://api.revenuecat.com/v2/projects/${PROJECT_ID}/customers/${id}/active_entitlements`,
        { headers: { Authorization: `Bearer ${Deno.env.get('RC_SECRET_KEY')}` } }
    )
    if (res.status === 404) return false    // RevenueCat has never seen this customer
    if (!res.ok) throw new Error(`RevenueCat ${res.status}`)
    const { items = [] } = await res.json()
    return items.some(e => e.entitlement_id === entitlementId)
}
```

The `payload.sub` in this check must be the same value you passed as `external_id` at paywall launch. If they diverge, RevenueCat resolves a different customer and the check correctly returns no entitlement. No webhook pipeline, no subscription entity to keep in sync. RevenueCat already knows who is subscribed, so ask it at the moment the answer matters, and cache the result per account for a short window if the function is hot.

Be clear about what the JWT buys you here. It stops one user from invoking your functions as another user, and it is what makes the deny-all RLS model work, since your functions are the only data path and they need to know who is calling. It does not change the root of trust: the vault UUID can mint a JWT for its account, so the UUID is still the underlying credential, the same model as Firebase anonymous auth. `crypto.randomUUID()` gives you 122 random bits, so it is unguessable, and it must stay that way: never put the device ID or the token in URLs, logs, error messages, or anything user-visible. For a step up, store the device ID with `locked=true` so reading the credential from the vault requires Face ID.

## Push notifications without an email address

The same account ID keys your push setup. Despia compiles the OneSignal SDK into the binary and registers the device at launch, so the only thing your web layer does is tell OneSignal which account this device belongs to. Link on every launch, right after the identity bootstrap resolves.

```javascript
function linkPush(accountId) {
    if (!isDespia) return
    despia(`setonesignalplayerid://?user_id=${encodeURIComponent(accountId)}`)
}
```

The `user_id` becomes the `external_id` in OneSignal, and your backend targets it with the same identifier it already stores for RevenueCat and your entities. One ID, three systems.

```javascript
// inside a Base44 backend function, the REST API key never reaches the client
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

That snippet is the minimal version. The template generalizes it as a JWT-verified `sendPush` function with targets for self, one user, several users, a tag-based segment, or every device, plus scheduling by absolute time or each user's local time and iOS badge control, and a `pushTags` function for building segments. On the client, `linkPushUser` runs on every authenticated load from the auth context, so the device-to-account link never goes stale.

Notice what is absent: no permission-gated signup, no email capture to build an audience. The anonymous account is the audience. You can message a user who has never told you anything about themselves, which for most consumer apps is the entire point of push.

## Account deletion that does not orphan purchases

Apple requires apps that create accounts to offer account deletion, and anonymous accounts do not exempt you if you store data server-side. The naive implementation deletes the row and mints a fresh identity, and it has a nasty consequence: RevenueCat entitlements are keyed to the account ID you passed as `external_id`, so a new ID orphans the purchase until a transfer syncs through, and in the gap a paying user sees locked features, which is how refund requests and one-star reviews get written.

The template's `authDeleteAccount` avoids the gap entirely with two modes:

**Anonymize, when the account has a** `device_id`**.** The record is kept but reverted to a pristine guest: `email` back to the synthetic `device-<uuid>@anon.local`, name to Guest, `password_hash`, `google_id`, `apple_id`, and avatar cleared, `is_anonymous` back to true, admin rights revoked, and everything hanging off the account wiped. The `id` and `device_id` stay unchanged on purpose. RevenueCat, OneSignal, and your data model all keep pointing at a valid account, so in-app purchases survive with zero transfer choreography. This satisfies the store rule, which requires deleting the account and its associated personal data, and an anonymous shell holds none.

**Hard delete, when there is no** `device_id`**.** A web-only account was never bound to a device or a native purchase, so the record is removed entirely.

The client flow is short: confirm with the user, invoke the function, then run the normal guest sign-in again.

```javascript
async function deleteAccountData() {
    await base44.functions.invoke('authDeleteAccount')

    // same record, now a pristine guest, so mint a fresh session for it
    localStorage.removeItem('app_auth_token')
    await signInWithDevice()

    // access was never at risk, the store owns the purchase
    // and external_id never changed
    await checkEntitlements()
}
```

Before the function runs, the template confirms the deletion with the account's original sign-in method: password accounts re-enter the password, Google and Apple accounts re-authenticate and the returned identity must match this exact account, and guests confirm with Face ID through a locked vault key on native or by typing DELETE on web. And one maintenance contract to honor as your app grows: `authDeleteAccount` is the single place user data is erased, so every new entity keyed to an account must be wiped there, or deletion leaves orphaned personal data behind, which is a store-review and GDPR violation.

On the next launch the automatic guest sign-in finds the same record by `device_id` and the user continues as a fresh guest with their purchases intact. And if anything ever does go wrong with your database, the store is still the fallback: `getpurchasehistory://` answers from the Apple ID or Google account, which owns the purchase no matter what your entities say.

## Account switching, and why you should not fight the store

Sooner or later you add optional login for cross-device sync, and the question arrives: what happens when someone logs out and logs into a different account on the same phone? The answer starts with a constraint that no app architecture can change. The store sells to the Apple ID or Google account, not to your app account, and Apple does not allow one Apple ID to hold two active subscriptions to the same product. Two app accounts on one device can never each own their own copy of your subscription. If the second account tries to buy while the device's store account already has it, the store reports the existing purchase instead of charging again, and RevenueCat applies its configured transfer behavior. This applies to new purchases, not just restores.

RevenueCat has a setting for that moment, restore behavior, and the guidance is one line: leave it on the default, Transfer to new App User ID. In this architecture it is a safety net, not a mechanism you build on. Despia passes your account ID as `external_id` on every call and the ID stays stable for the life of the device identity, so in steady state the receipt and the account never diverge and no transfer ever fires. The default earns its keep in exactly two moments. If the vault is ever lost and a fresh account ID gets minted, the user taps Restore Purchases, the receipt posts under the new ID, and RevenueCat re-points the entitlement to it, recovery with no support ticket. And on account switching, the entitlement follows whoever last synced on the device that owns the purchase, which is the store's own rule. In Despia terms, the moment a transfer can happen is any receipt sync under the new ID: a purchase, a paywall checkout, or a Customer Center restore. `getpurchasehistory://` never triggers one, it reads the local store and does not talk to RevenueCat, which is why it stays instant and offline-capable. Your backend sees a `TRANSFER` webhook when a transfer does happen, and there is nothing to build for it. The one configuration to never pair with anonymous accounts is Keep with original App User ID: it blocks the restore-recovery path, so a user who reinstalls after losing the vault is locked out of what they paid for.

Two rules make optional login safe on top of this:

**Signup is an upgrade, never a new account.** Because you own the auth system, this is one function, and the template ships it as `authLinkAccount`: it verifies the current anonymous JWT, then attaches either email and password credentials or a Google identity to the same `Account` record and flips `is_anonymous` to false. The account ID does not change, so nothing transfers, the subscription and push linkage stay exactly where they were, and everything the user did as a guest is preserved. Creating a second parallel account at signup is the mistake that causes most "my subscription disappeared" tickets, and it is exactly what a naive register function does, so route signed-in guests through the link function, not through register.

**On a second device, gate on the account, not only the local store.** `getpurchasehistory://` answers from the device's own store, so on a user's new phone, and always across platforms, it comes back empty even though their account is entitled. When a login is active, the client rule is local store check or account check via your backend, whichever grants. This is also why the post-purchase link prompt earns its place: a signed-in user who restores on a new device posts the receipt under the account that already owns it, so nothing transfers and nothing can go wrong. The transfer machinery only exists for the user who never linked, and the prompt exists to make that user rare.

**Switching moves the entitlement, and that is the correct behavior.** If a user signs into a different existing account on a device whose store account owns the purchase, the entitlement transfers to that account on the next sync, and transfers back when the original account syncs again. Your backend sees `TRANSFER` events bounce between the two IDs. Do not treat that as a bug to suppress, it is the store enforcing one subscription per store account. If your product genuinely cannot tolerate an entitlement following the device, that is the signal you have outgrown loginless: switch to mandatory login before purchase and the Keep with original behavior, which is a different app than the one this post describes.

The template already implements switching the right way: an iOS-style account picker remembers up to five accounts per device, guest included, mirrored into the vault so the list survives reinstall, and it runs on the default transfer behavior rather than fighting it.

**What a restore changes, and what it must not.** When a restore lands while a different account is signed in, exactly one thing moves: the receipt's owner inside RevenueCat. Do not swap the device ID in the vault, it is identity plumbing for the device's anonymous account and has nothing to do with purchase ownership. Do not touch the session, a restore is not a login event. And never merge account data because a receipt moved, the two accounts may be two people sharing a device, and a receipt cannot tell you which. On the backend the `TRANSFER` webhook carries `transferred_from` and `transferred_to`, arrays of account IDs identifying who lost the purchases and who now owns them, with no product or entitlement fields, since a transfer moves everything between the two customers. The loser also gets no follow-up EXPIRATION event, the TRANSFER is the revocation signal. So the entire correct response is entitlement bookkeeping: mark every account in `transferred_from` unentitled, re-query RevenueCat for the `transferred_to` ID rather than computing a delta from the event, or do nothing at all if you query RevenueCat live. Merging accounts is a separate, explicit, user-initiated feature, or it is a bug.

**The consent step polished apps add.** The `getpurchasehistory://` response includes `externalUserId`, the account ID that made the purchase. Compare it to the current signed-in account before offering a restore: if they differ, show the choice, this purchase belongs to another profile on this device, move it to this account or switch to that one, and the template's saved-account picker makes switching a one-tap. The silent transfer is correct behavior, the consented transfer is better UX, and it prevents the where-did-my-subscription-go ticket from the other account.

The rule that summarizes all of it: your app accounts are a layer on top of store identity, not a replacement for it. Design account switching around what the store will actually do, not around what your database schema implies.

## The failure modes to design for

**The vault can be unavailable.** iCloud can be signed out, and Android Key/Value Backup is best-effort across OEMs. Treat the vault as an identity cache, not as the entitlement store. If the read throws on a device that has actually used the app before, you generate a new ID, and the user recovers access through restore purchases because the store remembers even when the vault does not. Never gate premium features on the vault alone.

**Two devices, one Apple ID, one vault.** iCloud KVS syncs, so a user's iPhone and iPad resolve to the same account. That is correct behavior, one person is one account, and it matches how their subscription works anyway. The `deviceSignIn` filter-then-create on `device_id` already handles it, just keep that pair as the only path that mints accounts. One honest edge remains: iCloud KVS sync is not instant, so a brand-new second device can beat the sync and mint its own ID before the first device's key arrives. That occasionally leaves one human with two account records, which is harmless for access, the store owns the purchase and Restore Purchases recovers it under either ID, but expect it in your data rather than treating it as a bug.

**Do not reach for device fingerprinting.** A self-generated UUID stored in the vault is a random identifier you created, which is fine. Deriving an ID from hardware signals to track the user is exactly what Apple's fingerprinting rules prohibit, and you do not need it.

**Trial abuse is mostly solved for free.** Because the vault survives uninstall and reinstall, a `hasUsedTrial` flag next to the device ID closes the delete-and-redownload loophole without any server work.

## Where this fits

This is the right default for consumer apps built in Base44 where signup friction is the enemy: utilities, content apps, games, anything where the purchase is the account. You keep the conversion of a zero-friction first open, and you keep the option of full accounts and web access whenever a user wants them. When users eventually want email, Google, or Apple login, the custom auth system is already in place, link into the existing account entity, follow the two switching rules above, and the account ID stays the spine of the system from anonymous first launch to full accounts. The template at [github.com/despia-native/base44-native-boilerplate](https://github.com/despia-native/base44-native-boilerplate) carries the whole surface: `deviceSignIn` for anonymous accounts, `authLinkAccount` for upgrading guests, `authDeleteAccount` with the anonymize logic, Google, Apple, and password auth, `sendPush` and `pushTags` for targeted notifications, the RevenueCat client layer with paywall, Customer Center, and premium state, and the saved-account switcher. The docs inside explain each decision, JWT\_AUTH.md, ACCOUNT\_DELETION.md, DB\_SECURITY.md, PUSH\_NOTIFICATIONS.md, and ANTI\_FREEZE.md, and TEMPLATE\_SETUP.md is the checklist that makes the clone yours.

## Get it on the stores

Take the Base44 app you already built and ship it to iOS and Android with native purchases, push, and persistent storage wired in through one JavaScript call. Code signing and submission run from the browser.

[See the setup docs at setup.despia.com](https://setup.despia.com)