---
title: "Base44 App Rejected Under Guideline 5.6: The Fix"
seoTitle: "Base44 App Rejected Under Guideline 5.6: The Fix"
seoDescription: "Similarity is one of the first things to check after a Base44 guideline 5.6 rejection. How to customize auth, prepare App Review, and clear it."
datePublished: 2026-08-19T13:17:04.596Z
cuid: cmt048r0c00000ajk6avh9er1
slug: base44-app-rejected-under-guideline-5-6-the-fix
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/a845b79f-1c68-4214-a3d0-9c6f6c0243bb.png
tags: ios, ios-app-developer, webview, webapp, ios-app-development, appstore, base44, appreview

---

If a Base44 app receives a 5.6 letter despite hiding nothing, similarity is one of the first things worth checking, and the legacy login screen is where it shows most. Base44 now ships customizable login pages. Here is what Apple checks and how to clear it.

## What does a guideline 5.6 letter actually mean?

It means Apple invoked the Developer Code of Conduct rather than a technical rule. 5.6 covers respectful communication, customer trust, ratings integrity, accurate developer identity, and app quality, and it carries account-level implications rather than applying only to one submission.

The wording is what alarms people. You get told there is a pattern of unusual behavior commonly associated with fraudulent activity, and that features appear to have been hidden during review, when you hid nothing at all.

The weight is real, though. Apple states that a Developer Program account will be terminated for actions not in accordance with the code, and that restoring it means submitting a written statement of the improvements you plan to make, which Apple has to approve. For the full breakdown, see our [guideline 5.6 rejection guide](https://blog.despia.com/guideline-5-6-rejection-why-ai-built-apps-get-flagged).

References throughout are checked against the [App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/) as updated on June 8, 2026.

## What actually triggered the rejection?

Similarity is the first thing worth investigating. Apple's published 2025 numbers show enforcement volume sitting far more in the copycat and spam category than in apps hiding functionality, though aggregate totals cannot diagnose an individual case.

Apple's App Review team evaluated more than 9.1 million submissions in 2025. Over 371,000 were rejected for copying other apps, spam, or misleading users, against roughly 22,000 for hidden or undocumented features. The larger bucket is roughly 17 times the one your letter names. Source: [Apple's 2025 App Store fraud analysis](https://www.apple.com/newsroom/2026/05/the-app-store-stopped-over-2-point-2-billion-usd-in-fraudulent-transactions-in-2025/).

Apple also states that its review systems use AI to analyze app similarity and flag problematic changes in updates, and that AI development tools are driving the submission volume behind that. Guideline **4.3(b)** is the written version: do not submit apps that are indistinguishable from what is already widely available.

Base44 gets you a working app very fast. What it cannot do is make your app look like nobody else's, and that is the part review is measuring.

## Why is the Base44 login screen the problem?

Because the legacy Base44 login screen appears across submissions from unrelated developers, and it is the first thing a reviewer sees on an app they cannot otherwise verify. Apple does not disclose screen weighting, so treat it as the first similarity hotspot worth eliminating.

You know the screen: centered card, rounded icon tile, "Welcome back", Continue with Google, an OR divider, email and password fields with small leading icons, a full-width button, "Don't have an account? Create one" underneath.

It is a good screen. That is exactly the problem. Nothing on it belongs to your product, and it sits in front of everything a reviewer would otherwise use to tell your app apart.

## How do you switch to custom login pages?

Base44 rolled out custom login pages to all users on 2 June 2026. New apps ship with editable login, register, forgot password, and reset password pages. Existing apps on the old built-in pages migrate through the App Visibility setting in six clicks.

1.  Open your app editor and click **Dashboard**.
    
2.  Click **Overview**.
    
3.  Open the **App Visibility** dropdown.
    
4.  Select **Public (no login)**.
    
5.  Read the modal, then click **Enable custom auth**.
    
6.  Base44 writes a wiring prompt into the AI chat to connect the new pages to your routes. Let it run.
    

Apps already on custom login pages show a single Public option and need no change. Once migrated, the pages live inside your app, so you can restyle them, translate them, and brand them like any other screen.

## How do you redesign the auth pages?

Change the composition, not the palette. Move off the centered card, set a real typeface, rewrite every string in your product's voice, and make the pages visually continuous with your onboarding screen.

1.  **Change the composition.** A split layout with brand imagery on one side, a top-anchored form under a full-bleed header, or a two-step email-then-password flow all read as deliberate.
    
2.  **Set a typeface.** The highest-leverage single change available. The default system stack is the most recognizable tell in a generated app.
    
3.  **Rewrite every string.** "Welcome back" and "Log in to your account" appear on an enormous number of screens.
    
4.  **Match your onboarding.** Most Base44 apps have a strong branded first screen and then drop into an unbranded login. That seam is what a reviewer registers as a shell around a template.
    

Changing the accent color from purple to teal is not a redesign. The layout is the fingerprint.

Paste this into Base44's AI chat to get a redesign rather than a recolor:

```plaintext
CONTEXT
This is a Base44 app. The auth pages are close to the default and Apple
rejected the submission under guideline 5.6 for resembling other template
submissions.

TASK
1. Redesign the login, register, forgot password and reset password pages
   using a composition that is NOT a centered card with a stacked field group.
2. Apply the typeface, spacing scale, radius and background treatment from our
   onboarding screen so the whole app reads as one product.
3. Rewrite every string on these pages in our product's voice.
4. Keep all existing auth behavior, routes and redirects working exactly as
   they do now.

RULES
Never change only the accent color and call it a redesign.
Never leave a default template string on these pages.
Never break sign-in for existing accounts.

DONE WHEN
The auth pages are visually continuous with onboarding, no original template
string remains, and an existing account signs in on a real device.
```

## Is building with AI against App Review rules?

No. Apple regulates what the resulting app does and how it is submitted. Where AI-built apps run into trouble is low-effort similarity, and a default theme shipped untouched is what that looks like from outside.

Set your tokens before you build more screens: palette, radius scale, typeface, spacing rhythm, border weight, shadow treatment. Every component generated afterwards inherits your identity instead of the platform's default.

Do the same with the icon. A model produces one in 5 seconds and it looks like every other icon produced that way. Twenty minutes in a real design tool removes an entire category of similarity signal.

## Do you need a custom domain?

Not for approval. Base44 states you do not need one to submit to the App Store or Google Play, and can generate the store files from your default Base44 URL. A custom domain is a branding decision, not a review requirement.

The branding case is still worth making. A default platform subdomain turns up in your app, your emails, and your store listing, and it reads as a prototype to everyone who sees it.

What does matter at review time is that the app Apple sees is the app you meant to submit. Base44 builds against your published app, so publish before you generate store files, then open that exact URL on a phone and walk through it.

Apple does not require you to submit against production specifically, but the features you are submitting have to be accessible to App Review under guidelines **2.1(a)** and **2.3.1(a)**. Separately, guideline **2.5.2** prohibits downloading, installing, or executing code that introduces or changes features or functionality after review. Content updating remotely is normal and expected. Materially different functionality switched on after approval is not.

## How do quality problems become a conduct issue?

Through guideline 5.6.4, which names excessive customer reports, negative reviews, and refund requests as indications of failing to maintain quality, and says that inability to maintain quality may factor into whether a developer abides by the code of conduct.

A broken screen is already a 2.1 completeness problem, since Apple rejects incomplete binaries, crashes, and obvious technical issues. What 5.6.4 adds is that the same failure can also feed a conduct judgment rather than staying purely technical.

That matters because a reviewer who suspects a template spends the rest of their time looking for confirmation. Before submitting, walk the whole app on a real device: open every screen, tap every button, complete every form to the last field. Anything that dead-ends converts a suspicion into a decision.

## Are your store claims accurate?

Guideline 5.6.2 requires your representation of yourself, your business, and your offerings to be accurate, truthful, relevant, and up to date. Read your listing as a reviewer would, then delete anything you could not evidence.

This catches AI-built apps more than people expect, because generated marketing copy overstates by default. Any claim about who built the app, what qualifications sit behind it, what it is certified to do, or what results it produces has to be true and verifiable. Professional titles and credentials count.

While you are in the metadata, guideline **2.3.3** states screenshots should show the app in use, not merely title art, a login page, or a splash screen. Leading with your auth screen is both a similarity signal and a compliance problem.

## How do you make the app reviewable?

Give review working demo credentials, keep the backend live, and describe every feature and the path to it in Notes for Review. Apple states outright that generic descriptions will be rejected.

Guideline **2.1(a)** requires final versions with complete metadata, working URLs, no placeholder content, demo account details when there is a login, and your backend live during review. Guideline **2.3.1(a)** goes further, requiring features to be accessible to review and every new feature to be described with specificity.

In App Store Connect:

1.  Open your app, then the version you are submitting.
    
2.  Scroll to **App Review Information**.
    
3.  Tick **Sign-in required** and enter demo credentials.
    
4.  Verify those exact credentials on a real device the same day you submit.
    
5.  In **Notes**, name each non-obvious feature and the path to it. Name the tab, the button, the screen. Not "various improvements".
    
6.  Say how to reach anything gated behind a purchase, a role, or a region.
    
7.  Confirm your support URL and privacy policy URL both load.
    

## Does guideline 4.2.6 ban building on Base44?

No. It requires the app to be submitted directly by the provider of its content, rather than mass-submitted by a platform from its own account. Meeting that ownership condition is necessary rather than sufficient, so the customization work still applies.

The full rule says apps created from a commercialized template or app generation service will be rejected unless submitted directly by the provider of the app's content. It exists to stop agencies pushing near-identical client apps from one account.

Apple then goes further in your favor: it says those services should offer tools that let their clients create customized, innovative apps that provide unique customer experiences. That is the same standard as everything above, written by Apple rather than argued by us.

## What do you do if you already have the letter?

Do not resubmit the same binary with a bumped version. Make the changes, reply in Resolution Center before submitting the new build, and list what changed screen by screen with screenshots attached.

Apple states that review takes longer when an app is repeatedly rejected for the same guideline violation, so a second identical submission works against you.

```plaintext
We have addressed the issues as follows.

Sign-in: moved to custom login pages and redesigned with our own layout,
typography and copy. It is now continuous with our onboarding screen.

Visual identity: replaced the default theme with our own palette, radius and
typography across the app.

App icon and store screenshots: replaced. Screenshots now show the app in use.

Metadata: reviewed the listing and removed claims we cannot evidence.

Review access: demo account added under App Review Information and verified
today. Notes for Review describe each feature and where to find it.

The submitted build points at the same domain our users get. No functionality
is gated, remote-flagged or withheld from review.
```

No argument and no emotion. Evidence of change is what moves a 5.6 letter.

## Is Base44 why your app was rejected?

No. Similarity, quality, accurate representation, and verifiability are what Apple's review rules evaluate, and none of them is a build tool. A poorly differentiated app written in Swift gets flagged on the same signals.

What gets flagged is an app that arrives looking like every other app from the same starter, with claims nobody checked and a reviewer who cannot get in. Custom login pages, your own theme, a real icon, and a filled-in review notes field close that gap, and none of it takes more than an afternoon.

## Do you need more than Base44's own store files?

Only if you need native capability. Base44 can scan your app and generate the IPA and AAB you submit through your own developer accounts, which is enough for many products. Its docs list push notifications and full offline mode as unsupported in that wrapper.

Base44's own documentation describes the mobile app as a secure web view that opens your app's URL, which is why those two capabilities sit outside it.

If they are core to your product, that is where Despia fits. Same Base44 codebase, still shipped under your own developer account, with 50+ native device features available from it, push notifications and offline behavior among them, plus content updates over the air. Code signing and submission run from the browser, with no Mac and no Xcode required.

[See the setup docs at setup.despia.com](https://setup.despia.com)