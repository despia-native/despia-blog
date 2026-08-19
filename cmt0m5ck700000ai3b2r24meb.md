---
title: "Base44 Bundle ID and Package Name: What to Know"
seoTitle: "Base44 Bundle ID and Package Name: What to Know"
seoDescription: "Base44 generates your bundle ID and package name, and they are public and permanent. What that costs you later and how to pick your own instead."
datePublished: 2026-08-19T21:38:18.984Z
cuid: cmt0m5ck700000ai3b2r24meb
slug: base44-bundle-id-and-package-name-what-to-know
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/1ab7764b-bdc7-4165-ac56-b8e5ac5308f9.png
tags: android-app-development, ios, android, ios-app-developer, ios-app-development, base44

---

Your bundle ID and package name are the permanent identity of your app on both stores. Base44 generates them for you in a fixed format derived from your app ID, and they cannot be changed in the generated build. That is a five second decision at submission time with consequences you live with for the life of the listing.

For the wider picture, see [how to build a Base44 mobile app you own](/blog/base44-mobile-app).

## What is my Base44 package name?

Base44 documents the format directly: your Google Play package name is `com.base[app-id].app`, where `[app-id]` is your Base44 app ID, taken from the URL of your app editor. An app whose editor URL contains `69e0c4bdd31bdu8fda51775g` gets the package name `com.base69e0c4bdd31bdu8fda51775g.app`.

There is no field to change it. Base44's documentation states that the bundle ID and signing key are configured automatically and that these values cannot be changed inside the generated IPA or AAB files.

## Why does the package name matter after launch?

Because it is public and permanent. On Google Play the package name appears in the app's own store URL, so anyone looking at the address bar reads it. It is also the immutable key both stores use for the app, so it cannot be edited later.

Baked into that identifier on both platforms:

*   The signing certificate and provisioning profile used to build the binary
    
*   The App Store Connect and Google Play Console records
    
*   Push notification credentials, App Groups, associated domains and every other entitlement granted to that identity
    

This is not a platform quirk. Apple and Google both treat the app identifier as the unique key across their entire ecosystem, which is why neither offers an in-place rename. Changing it means a new store record, and a new store record does not inherit reviews, ratings or installs from the old one.

So the question is not whether you can change it later. It is what you want printed on the app for as long as it exists.

## Does carrying a builder's name in the identifier matter?

It depends on who reads it. For an internal tool or a side project, nobody looks. For an app you are selling, raising against, or licensing to an enterprise buyer, an identifier naming the platform you built on is a detail people notice.

Three situations where it comes up:

**Enterprise and procurement.** Security reviews read package names. An identifier that resolves to a third-party app generation service invites a question about who controls the binary, and you answer it from a defensive position.

**Acquisition and diligence.** A buyer looking at your app assets finds an identifier they cannot rebrand without abandoning the listing and its review history.

**Brand consistency.** Everything else in your product carries your domain. The one string a curious user can read in the Play Store URL carries somebody else's.

None of this is fatal. It is just permanent, which is a strange property for a decision most people make without noticing they are making it.

## Can I set my own bundle ID with Despia?

Yes, at project creation. You choose the bundle ID and package name yourself, in the reverse domain format of your own domain, for example `com.yourcompany.yourapp`.

The same immutability rules apply afterwards, because they are Apple's and Google's rules rather than anyone's product decision. Once a build has been signed against an identifier, it cannot be edited from inside the editor, since changing it in place would invalidate every certificate, profile and store record tied to it.

What you get is a path that does not exist when the identifier is generated for you: if you set the wrong value before publishing, Despia support can repoint the project manually, typically within 72 hours. Email support with the old and new values and which platform it applies to. After the app is live on a store, a change starts a fresh identity with a new store record, so the practical rule is the same on any platform: get it right before you publish.

## Do I need a custom domain to submit?

Not for approval. Base44's documentation is clear that a custom domain is optional and that it can scan your app and generate the store files from the default Base44 URL. The reasons to own the domain are branding and portability, not store compliance.

Portability is the one that compounds. The mobile app has to point at some address, and whichever address you choose becomes the thing the binary is bound to. If that address is a subdomain on the platform that built the app, then leaving the platform means changing the URL the shipped app loads, which is a new build at best.

If the address is a domain you own, it is a redirection layer you control. Point it at the Base44 app today, point it at something else later, and the installed app on every user's phone follows without a resubmission. It costs about the price of a coffee per year and it is the cheapest optionality you will ever buy.

## What happens when I outgrow the platform I started on?

The identifier and the codebase move separately, and only one of them is portable by default. This is the part worth thinking about before you pick a stack, not after.

A Base44 app shipped through the built-in wrapper ties three things together: where the app is built, where it is hosted, and the identity it holds on both stores. Move the first one and you are re-entering the store as a different app, because the identifier you have cannot travel and the signing key is not yours.

Despia is framework agnostic. It ships whatever is served at a URL you control, so the arrangement is different in a useful way:

*   The **bundle ID and package name** are yours, chosen up front
    
*   The **domain** is yours, which means the app points at an address you own rather than a subdomain on someone else's platform
    
*   The **stack behind that domain** can change without touching the app record
    

Rebuild the front end in Next.js, hire developers and move to a different framework entirely, migrate the backend off the platform's managed services. As long as the same domain serves the app, the mobile app keeps its identity, its reviews, its ratings and its installs. If you eventually want out of the runtime too, the full Xcode and Android Studio projects export, so the escape hatch is a real project rather than a promise.

The general principle is the one worth taking away regardless of which tool you pick: **choose the constraint you can undo.** A framework choice is reversible. A hosting choice is reversible. A public, permanent app identifier tied to a platform you may leave is not.

## When is the generated identifier fine?

When the app is internal, experimental, or genuinely disposable, and when you have no plan to move. If you are validating an idea and the listing may not exist in six months, spending time on this is misallocated effort.

It is also worth being honest about the failure mode in the other direction: picking a beautiful bundle ID for an app that never ships is not a win. The decision only matters in proportion to how long the app lives and who inspects it. If the answer to both is "not much," take the default and move on.

Where it stops being fine is the moment the app becomes the business. At that point the identifier is infrastructure, and infrastructure you cannot change is worth choosing deliberately.

## Get it on the stores

Take the Base44 app you already built and ship it under an identifier you own, with the native capabilities the stores expect, from the same codebase. Code signing and submission run from the browser, with no CLI and no Mac required, and the full native projects export whenever you want them.

[See the setup docs at setup.despia.com](https://setup.despia.com)