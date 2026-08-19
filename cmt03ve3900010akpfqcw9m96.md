---
title: "Lovable App Rejected Under Guideline 5.6: The Fix"
seoTitle: "Lovable App Rejected Under Guideline 5.6: The Fix"
seoDescription: "Two common issues to check after a Lovable guideline 5.6 rejection: the default theme and the wrong deployment URL. Plus the steps to clear it fast."
datePublished: 2026-08-19T13:06:41.319Z
cuid: cmt03ve3900010akpfqcw9m96
slug: lovable-app-rejected-under-guideline-5-6-the-fix
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/bf6cb449-de8f-4595-ac96-2d599597e24d.png
tags: ios, ios-app-developer, webapp, ios-app-development, appstore, lovable, appreview

---

* * *

## title: "" description: "" slug: lovable-app-store-rejection keyword: lovable app store rejection

# Why Lovable apps get flagged under guideline 5.6

Two things are worth checking first on a Lovable app that got a 5.6 letter: an untouched default design, and a build pointed at the editor preview rather than the published app. Neither is concealment. Here is what Apple checks and how to clear it.

You built in Lovable, wrapped it, submitted it, and got a rejection citing guideline 5.6, a pattern of unusual behavior commonly associated with fraudulent activity, and features that appear to have been hidden during review. Nothing was hidden.

Guideline 5.6 is the Developer Code of Conduct. It covers conduct, customer trust, and accurate representation, and unlike a purely technical rejection it carries account-level implications: Apple states that a Developer Program account will be terminated for actions not in accordance with the code, and that restoring it requires a written statement of improvements Apple has to approve. That is why the wording is so much heavier than a 4.3 or a 2.1.

For the full breakdown of what guideline 5.6 means at the account level, see our [guideline 5.6 rejection guide](/blog/guideline-5-6-rejection).

Two Lovable-specific things are worth checking first. One is visual, one is procedural. References below are checked against the [App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/) as updated on June 8, 2026.

## What actually triggered the rejection?

Similarity is the first thing worth investigating. Apple's published 2025 numbers show enforcement volume sitting far more in the copycat and spam category than in apps hiding functionality, though aggregate totals cannot diagnose an individual case.

Apple's App Review team evaluated more than 9.1 million submissions in 2025. Over 371,000 were rejected for copying other apps, spam, or misleading users, against roughly 22,000 for hidden or undocumented features. Apple also states that its review systems analyze app similarity as an input, and that AI development tools are driving the volume behind it. Source: [Apple's 2025 App Store fraud analysis](https://www.apple.com/newsroom/2026/05/the-app-store-stopped-over-2-point-2-billion-usd-in-fraudulent-transactions-in-2025/).

Those are global category totals, so they do not tell you why any single app was flagged. What they do tell you is where the enforcement volume sits: the bucket about apps resembling other apps is roughly 17 times the one the letter names. Guideline **4.3(b)** says it plainly: do not submit apps that are indistinguishable from what is already widely available.

## Is the shadcn default theme the problem?

The theme is, not the library. shadcn/ui lands in your repo as source you own, with the visual identity in CSS variables you are meant to set. Leaving those at their defaults is what makes a thousand projects look alike.

Lovable uses Tailwind for styling, and many Lovable projects use shadcn/ui components. The underlying frontend stack varies by project age: newer apps default to TanStack Start with server-side rendering, while older ones use React with Vite. shadcn/ui is well built and accessible, and it is explicitly designed to be customized: the components land in your repo as source you own, and the visual identity lives in CSS variables you are meant to set.

What produces the generic result is the agent scaffolding it and stopping there. A thousand projects ship the same neutral palette, the same default radius, the same system font stack, the same card-and-border rhythm. The library handed you a solid vanilla base. Leaving it vanilla is the choice that gets noticed.

Being built with AI is not itself an App Review violation. Apple regulates what the resulting app does and how it is submitted. Where AI-built apps run into trouble is low-effort similarity, and an untouched default theme is what that looks like from outside.

**Set your tokens before you build any more screens.** In Lovable this lives in your global stylesheet and Tailwind config, and doing it first means every component the agent generates afterwards inherits your identity instead of the default one.

1.  **Palette.** Define your own foreground, background, primary, muted, border and destructive values. Not one hue swap on the default.
    
2.  **Radius.** One of the most recognizable tells. Change it, and change it consistently.
    
3.  **Typeface.** The highest-leverage single change in the list. Load one real display face for headings and check it renders on device, not just in preview.
    
4.  **Density and border weight.** Card padding, gap scale, border thickness, shadow treatment. Together these read as a designed product where a color change does not.
    

Then the auth screens, which are what a reviewer sees first on an app they cannot otherwise verify:

```plaintext
CONTEXT
This project is on the default shadcn theme and Apple rejected the submission
under guideline 5.6 for resembling other template submissions.

TASK
1. Update our CSS variables and Tailwind config with the palette, radius scale,
   typeface and spacing rhythm below, then make every existing component read
   from those tokens rather than hardcoded values.
2. Redesign the sign-in, sign-up and forgot-password screens using a
   composition that is NOT a centered card with a stacked field group.
3. Rewrite every string on the auth screens in our product's voice. No
   "Welcome back", no "Log in to your account".
4. Keep all auth logic, routes and redirects working exactly as they do now.

MY TOKENS
Palette: <fill in>
Radius: <fill in>
Heading typeface: <fill in>
Body typeface: <fill in>

RULES
Never leave a hardcoded default color or radius in a component.
Never change only the accent color and call it a redesign.
Never break sign-in for existing accounts.

DONE WHEN
No screen still shows the default theme, the auth screens are visually
continuous with the rest of the app, and an existing account signs in on a
real device.
```

Give the agent a spec, not a vibe. "Build a sign-in screen" returns the default. "Split layout, brand image left, form right, our tokens, no centered card" returns your app.

## Which URL should the build point at?

The published app or your custom domain, never the editor preview. Publishing is a distinct action from editing, so unpublished changes stay unpublished and the preview you work in is not the app your users get.

This is a particularly important Lovable-specific trap, because it can leave App Review seeing a different state from the one you intended to submit.

Every Lovable project gets a subdomain like `yourapp.lovable.app` on any plan, with custom domains on paid plans. Publishing is a distinct action from editing, and unpublished changes stay unpublished. So the preview you work in and the app your users get are not the same thing.

Apple does not require production specifically, but the features you are submitting have to be accessible to App Review under guidelines **2.1(a)** and **2.3.1(a)**, and an editor preview that diverged from your published app does not expose them. Separately, guideline **2.5.2** prohibits downloading, installing, or executing code that introduces or changes features or functionality after review. Content and data updating remotely is normal and expected.

**Before every submission:**

1.  Publish in Lovable so the live URL reflects your current build.
    
2.  Connect your custom domain if you have one and confirm it serves the published app over HTTPS.
    
3.  Set your app's start URL to that domain, or the published `.lovable.app` URL if you have not connected one. Never the editor preview URL.
    
4.  Open that exact URL in a phone browser and walk the app before you build the binary.
    

Lovable's reference for the publish flow and subdomain settings is at [docs.lovable.dev/features/publish](https://docs.lovable.dev/features/publish).

## Can your Lovable credits run out during review?

Yes, and Lovable documents the failure itself. One credit balance now covers building, hosting and running the app on Cloud, and AI features in deployed apps. When it hits zero, Lovable's backend and AI services may stop serving the deployed app.

This is easy to miss, because everything works the day you submit. Cloud and AI gateway usage show up as Run credits under Settings, Plans and credit usage, Usage details, and they are consumed by your live app rather than by your prompting. Lovable is rolling the unified balance out gradually, so some workspaces still show the older separate Cloud and AI balances.

Guideline **2.1(a)** requires your backend services to be live and accessible during review. If the balance empties while your app sits in the queue, the services behind it may stop serving the deployed app, a reviewer opens it, nothing answers, and you are rejected for something that worked when you hit submit. Your data is safe while services are paused, but you cannot reach it either.

Two things to do before submitting. Check the balance, and enable auto top-up with a monthly spend limit you are comfortable with, which is available on Pro and Business plans. It is the cheapest rejection you will ever avoid.

## How do you make the app reviewable?

Give review working demo credentials, keep the backend live, and describe every feature and its path in Notes for Review. Apple states outright that generic descriptions will be rejected.

The same section requires demo account details when the app has a login and final metadata with no placeholder content or dead URLs. Guideline **2.3.1(a)** goes further: features must be accessible to review, and every new feature must be described with specificity in Notes for Review, with Apple stating outright that generic descriptions will be rejected.

In App Store Connect:

1.  Open your app, then the version you are submitting.
    
2.  Scroll to **App Review Information**.
    
3.  Tick **Sign-in required** and enter demo credentials.
    
4.  Verify those exact credentials on a real device the same day you submit.
    
5.  In **Notes**, name each non-obvious feature and the path to it. Name the tab, the button, the screen.
    
6.  Say how to reach anything behind a purchase, a role, or a region.
    
7.  Confirm your support URL and privacy policy URL both load.
    

Also check **2.3.3**: screenshots should show the app in use, not merely title art, a login page, or a splash screen.

## How do quality problems become a conduct issue?

Through guideline 5.6.4, which names excessive customer reports, negative reviews, and refund requests as indications of failing to maintain quality, and says that inability to maintain quality may factor into whether a developer abides by the code of conduct.

Guideline **5.6.4** states that indications of failing to maintain quality include excessive customer reports, negative reviews, and excessive refund requests, and that an inability to maintain high quality may be a factor in deciding whether a developer is abiding by the Developer Code of Conduct.

That is why a broken screen can matter here in addition to being a 2.1 completeness problem, since Apple rejects incomplete binaries, crashes, and obvious technical issues under App Completeness. A reviewer who suspects a template spends the rest of their time looking for confirmation. Walk the whole app on a real device before submitting: every screen, every button, every form to the last field. Anything that dead-ends converts a suspicion into a decision.

## Are your store claims accurate?

Guideline 5.6.2 requires your representation of yourself, your business, and your offerings to be accurate, truthful, relevant, and up to date. Read your listing as a reviewer would and delete anything you could not evidence.

Guideline **5.6.2** requires your representation of yourself, your business, and your offerings to be accurate, truthful, relevant, and up to date. Generated marketing copy overstates by default, so read your name, subtitle, description, and in-app claims as a reviewer would. Any claim about who built the app, what qualifications sit behind it, what it is certified to do, or what results it produces has to be true and verifiable. Delete anything you could not evidence.

## Does guideline 4.2.6 ban building on Lovable?

No. It requires the app to be submitted directly by the provider of its content, rather than mass-submitted by a platform from its own account. Meeting that ownership condition is necessary rather than sufficient, so the customization work still applies.

Guideline **4.2.6** says apps created from a commercialized template or app generation service will be rejected unless submitted directly by the provider of the app's content. In practice: submit under your own Apple Developer account, as a customized product rather than a rethemed instance. Building in Lovable does not put you on the wrong side of this. Submitting through someone else's shared account, or shipping something indistinguishable from the next generated app, does. Apple also says those services should offer tools that let their clients create customized, innovative apps that provide unique customer experiences, which is the same standard as everything above, stated in Apple's own words.

## What do you do if you already have the letter?

Do not resubmit the same binary with a bumped version. Make the changes, reply in Resolution Center before submitting the new build, and list what changed screen by screen with screenshots attached.

Apple states that review takes longer when an app is repeatedly rejected for the same guideline violation.

```plaintext
We have addressed the issues as follows.

Visual identity: replaced the default component theme with our own palette,
radius, typography and spacing across the app.

Sign-in: redesigned with our own layout and copy, now continuous with
onboarding.

App icon and store screenshots: replaced. Screenshots now show the app in use.

Metadata: reviewed the listing and removed claims we cannot evidence.

Review access: demo account added under App Review Information and verified
today. Notes for Review describe each feature and where to find it.

The submitted build points at our published production domain and the backend
is provisioned for the review period. No functionality is gated, remote-flagged
or withheld from review.
```

If you still believe the rejection is wrong after that, the [appeal path](https://developer.apple.com/contact/app-store/?topic=appeal) exists, but it is slower than fixing and resubmitting, and evidence of change is what actually moves a 5.6 letter.

## Is Lovable why your app was rejected?

No. Similarity, quality, accurate representation, and verifiability are what Apple's review rules evaluate, and none of them is a build tool.

Lovable is not why your app was rejected. Apple's stated inputs are app similarity, quality, accurate representation, and whether the app can be verified. None of those is a build tool.

What gets flagged is an app that arrives on the default theme, pointed at a preview URL, with a backend that may not be answering by the time a reviewer opens it. Fix the tokens, publish properly, top up the credits, and fill in the review notes.

## Get it on the stores

Once the app itself is review-ready, the remaining problem is getting the build signed and onto the stores.

Take the Lovable app you already built and ship it to iOS and Android as a real native binary, under your own developer account, with 50+ native device features available from your existing codebase and content updates over the air. Code signing and submission run from the browser, with no Mac and no Xcode required.

[See the setup docs at setup.despia.com](https://setup.despia.com)