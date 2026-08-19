---
title: "Guideline 5.6 Rejection: Why AI-Built Apps Get Flagged"
seoTitle: "Guideline 5.6 Rejection: Why AI-Built Apps Get Flagged"
seoDescription: "Apple's guideline 5.6 letter reads like a fraud accusation. What the Developer Code of Conduct actually says, and the steps to clear it."
datePublished: 2026-08-19T08:40:36.926Z
cuid: cmszud7vr000009jb0sef0a1u
slug: guideline-5-6-rejection-why-ai-built-apps-get-flagged
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/2c696231-a0d1-46d3-9edd-23f6ab748609.png
tags: ios, ios-app-developer, ios-app-development, appstore, appreview

---

The letter reads like an accusation. Apple identified a pattern of unusual behavior commonly associated with fraudulent activity, and your app appears to contain features that were intentionally hidden during review. You did not hide anything. There is no kill switch, no screen that unlocks after approval, no remote flag waiting to flip.

Apple does not publish how it decides, or how it weights any individual signal. What it does publish is that its systems use AI to analyze app similarity, identify malicious patterns, and flag potentially problematic changes in updates. For a legitimate developer holding this letter, similarity and reviewability are the two concrete things worth investigating, and neither of them is an engineering problem. Everything below is checked against the [App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/) as last updated on June 8, 2026.

## What guideline 5.6 actually says

Section 5.6 is the Developer Code of Conduct, and reading it changes how you respond to the letter.

It is about conduct rather than features. It asks developers to treat people with respect in App Store review responses, customer support requests, and communication with Apple, and it states that repeated manipulative or misleading behavior or other fraudulent conduct leads to removal from the Apple Developer Program.

It is about customer trust. Apps should not prey on users, rip off customers, trick them into unwanted purchases, force them to share unnecessary data, raise prices in a tricky manner, charge for things that are not delivered, or use other manipulative practices.

And it escalates to the account. Apple states that a Developer Program account will be terminated for actions not in accordance with the code of conduct, and that restoring it means submitting a written statement detailing the improvements you plan to make, which Apple has to approve and confirm before the account comes back.

That is why the letter feels so heavy. A 2.1 rejection wants a demo account. A 4.3 rejection usually starts with the submission, though Apple notes that repeated low-effort submissions under it can also lead to removal from the Developer Program. A 5.6 letter starts at the level 4.3 can escalate to, because it is the code of conduct itself.

Four subsections sit under it:

*   **5.6.1 Reviews and Ratings** covers how you collect ratings, and requires the API Apple provides rather than gathering them by other means.
    
*   **5.6.2 Developer Identity** requires that your representation of yourself, your business, and your offerings on the App Store is accurate, truthful, relevant, and up to date, so customers understand who they are engaging with.
    
*   **5.6.3 Discovery Fraud** states that manipulating any element of the App Store customer experience, such as charts, search, reviews, or referrals to your app, is not permitted.
    
*   **5.6.4 App Quality** states that indications of failing to maintain quality include excessive customer reports, negative reviews, and excessive refund requests, and that an inability to maintain high quality may be a factor in deciding whether a developer is abiding by the code of conduct.
    

Note what 5.6.4 does. Quality problems can therefore become part of a conduct judgment rather than remaining only a technical issue.

## The rejection numbers explain the wording

Apple published its App Store fraud analysis for 2025 in May 2026. App Review evaluated more than 9.1 million submissions and rejected over 2 million, including more than 1.2 million new apps and nearly 800,000 updates. By category:

| Reason for rejection | Submissions in 2025 |
| --- | --- |
| Privacy violations | 443,000+ |
| Copying other apps, spam, or misleading users | 371,000+ |
| Hidden or undocumented features | 22,000+ |

Source: [Apple, The App Store stopped over $2.2 billion in potentially fraudulent transactions in 2025](https://www.apple.com/newsroom/2026/05/the-app-store-stopped-over-2-point-2-billion-usd-in-fraudulent-transactions-in-2025/).

The copycat and spam bucket is roughly 17 times the hidden-features bucket. Those are global category totals, so they do not tell you why any single app was flagged. What they do tell you is where the enforcement volume sits, and it is not in apps concealing functionality.

Apple is explicit about the mechanism in that same report. Its systems use AI to identify malicious patterns, **analyze app similarity**, and flag problematic changes in updates, so human reviewers can focus their attention. It is equally explicit about the cause: powerful AI development tools are driving a surge in app submissions.

App similarity is not a side detail. It is a stated input to review, and it can turn "my app came from the same starter as thousands of others" into a flag. Guideline **4.3(b)** says the same thing in plain language: do not submit apps that are indistinguishable from what is already widely available.

## AI is not the problem. The untouched default theme is

The wrong lesson gets drawn from these rejections constantly, so it is worth saying directly. Being built with AI is not itself an App Review violation. Apple regulates what the resulting app does and how it is submitted, and 4.2.6 explicitly contemplates app-generation services. Where AI-built apps run into trouble is low-effort similarity, and an untouched default theme is what that looks like from the outside.

The component libraries are not the problem either. shadcn/ui is the good example. It looks excellent, it is accessible, and it is deliberately built to be customized: the components land in your repo as source you own, and the visual identity lives in CSS variables you are meant to set. What produces the generic result is an agent scaffolding it and stopping there, so a thousand projects ship the same neutral palette, the same default radius, the same system font stack, the same card-and-border rhythm. The library gave you a solid vanilla base. Leaving it vanilla is the choice that gets noticed.

Same with generated assets. A model produces an icon in 5 seconds and it looks like every other icon produced that way. Taking that into a real design tool and making decisions about mark, weight, contrast, and crop is 20 minutes that removes an entire category of similarity signal.

**Set your tokens before you build more screens.** Palette, radius scale, typeface, spacing rhythm, border weight, shadow treatment. Do it first and every component generated afterwards inherits your identity instead of the default one. Changing the accent color from purple to teal is not a redesign.

## The login screen is where it gets caught

Apple has not said which screens weigh most, so treat this as the first place to look rather than a ranking. It is the screen most likely to be identical across unrelated submissions, and it is the one a reviewer sees first.

Most AI builders ship the same authentication screen. Centered card, rounded icon tile at the top, a bold "Welcome back", a grey subtitle, a Continue with Google button, an OR divider, email and password fields with small leading icons, a full-width primary button, and "Don't have an account? Create one" underneath. It is a decent screen. That is the problem. Every project scaffolded from that builder gets it, and it is the first thing a reviewer sees, in front of an app they cannot otherwise verify.

**Steps:**

1.  **Change the composition.** Move off the centered card entirely. A split layout with brand imagery on one side, a top-anchored form under a full-bleed header, or a two-step email-then-password flow all read as deliberate.
    
2.  **Set a typeface.** Highest-leverage single change on the list. The default system stack is the most recognizable tell in a generated app.
    
3.  **Rewrite every string.** "Welcome back" and "Log in to your account" appear on an enormous number of screens.
    
4.  **Restyle the social button.** Follow Google's branding rules for the mark, but height, radius, border treatment, and placement are yours.
    
5.  **Make it continuous with your onboarding.** Most AI-built apps have a strong branded first screen and then drop into an unbranded login. That seam reads as a shell wrapped around a template.
    
6.  **Compare it against three other apps built with the same tool.** If you cannot tell them apart, neither can a similarity check.
    

A prompt that produces a redesign rather than a recolor:

```plaintext
CONTEXT
This app was scaffolded from a template and the auth screens are still the
default. Apple rejected the submission under guideline 5.6 for resembling
other template submissions.

TASK
1. Redesign sign-in, sign-up and forgot-password using a composition that is
   NOT a centered card with a stacked field group.
2. Apply the typeface, spacing scale, radius and background treatment from our
   onboarding screen so the app reads as one product.
3. Rewrite every string on these screens in our product's voice.
4. Keep all existing auth logic, routes and error handling unchanged.

RULES
Never change only the accent color and call it a redesign.
Never leave a default template string on these screens.
Never break sign-in for existing accounts.

DONE WHEN
The auth screens share the onboarding screen's visual identity, no original
template string remains, and an existing account signs in on a real device.
```

## The rest of the shell counts too

Review looks at the whole submission.

1.  **Replace the generated icon.** A gradient square with a letterform is the most common AI-generated icon in existence. Ship a flat 1024x1024 PNG with no alpha channel.
    
2.  **Fix the screenshots.** Guideline **2.3.3** states screenshots should show the app in use, not merely title art, a login page, or a splash screen. If your first screenshot is the auth wall, you led with the generic part and you are out of compliance at the same time.
    
3.  **Scrub placeholder content.** Guideline **2.1(a)** requires final versions with no placeholder text and no dead URLs. Search for lorem, TODO, and example.com.
    
4.  **Rewrite the store description.** Not a violation on its own, but another similarity signal in a submission that already has several.
    

## Quality is part of the conduct judgment

A broken screen is already a 2.1 completeness problem, since Apple rejects incomplete binaries, crashes, and obvious technical issues. 5.6.4 means it can also matter here. A reviewer who suspects a template will spend the rest of their time looking for confirmation, and an app that does not work gives it to them.

Before submitting, walk the app on a real device. Open every screen. Tap every button and confirm none of them dead-ends. Complete every form to the last field. Anything a reviewer cannot finish is a reason for them to stop, and it converts a suspicion into a decision.

The forward-looking half of 5.6.4 matters too. Excessive customer reports, negative reviews, and high refund rates are named as indications, which means quality problems that survive approval can come back as a code of conduct question later.

## Represent the app accurately

5.6.2 is short and it catches more AI-built apps than people expect, because generated marketing copy overstates by default.

Your app name, subtitle, description, screenshots, and in-app claims all have to be accurate and current. Any claim about who built the app, what qualifications sit behind it, what it is certified to do, or what results it produces has to be true and verifiable. Professional titles and credentials are part of this. So is the pricing you advertise, inside the App Store and outside it.

If the model wrote your listing, read it as a reviewer would and delete anything you could not evidence.

## Do not do the things 5.6.1 and 5.6.3 actually ban

Worth stating plainly, because growth advice on the internet routinely recommends these.

*   Do not buy, incentivize, trade, or otherwise arrange ratings and reviews.
    
*   Do not gather ratings outside the API Apple provides for it.
    
*   Do not stuff keywords, use competitor names in your metadata, or manipulate search and charts.
    
*   Do not run install campaigns that misrepresent your price.
    

These are the behaviors 5.6 is genuinely written about. If any of them are in your growth plan, remove them before you resubmit, because an account already carrying a 5.6 letter has no margin.

## Apple's own rule says to customize it

Guideline **4.2.6** is worth reading if you built on any app-generation platform, because it says more than people quote.

It states that apps created from a commercialized template or app generation service will be rejected unless they are submitted directly by the provider of the app's content, that those services should not submit apps on behalf of their clients, and that they should offer tools letting their clients create customized, innovative apps that provide unique customer experiences. It also allows template providers a single binary hosting all client content in an aggregated or picker model.

Two things follow. First, the app has to go up under the Apple Developer account of the person or company whose content and business it is. Building on a platform does not put you on the wrong side of 4.2.6. Submitting through a platform's shared account does.

Second, Apple has written down the standard in its own words: customized, innovative, unique customer experiences. That is the same instruction as the section above, coming from the rulebook rather than from us.

## Make the app reviewable

This is where a legitimate app accidentally lands in the hidden-features bucket.

**Point the submitted build at a complete, representative environment.** Apple does not require you to submit against production specifically, but the features you are submitting have to be accessible to App Review, which is guideline **2.1(a)** and **2.3.1(a)** below. Separately, guideline **2.5.2** prohibits downloading, installing, or executing code that introduces or changes features or functionality after review. Content and data updating remotely is normal and expected. Materially different functionality switched on after approval is not.

For most people the simplest way to satisfy that is to point at production, because a staging URL or a preview deployment usually lags the live app and nobody notices until a reviewer does. Check the start URL before every submission.

**Give review a working way in.** Guideline **2.1(a)** requires demo account details when the app has a login, and requires your backend live during review. Guideline **2.3.1(a)** goes further: features must be accessible for review, and every new feature and product change must be described with specificity in Notes for Review, with Apple stating outright that generic descriptions will be rejected.

In App Store Connect:

1.  Open your app, then the version you are submitting.
    
2.  Scroll to **App Review Information**.
    
3.  Tick **Sign-in required** and enter demo credentials.
    
4.  Verify those exact credentials on a real device the same day you submit.
    
5.  In **Notes**, name each non-obvious feature and the path to it. Name the tab, the button, the screen. Not "various improvements".
    
6.  Say how to reach anything gated behind a purchase, a role, or a region.
    
7.  Confirm your support URL and privacy policy URL both load.
    

## If the letter already arrived

Do not resubmit the same binary with a version bump. Apple states that review takes longer when an app is repeatedly rejected for the same guideline violation or when a developer has attempted to manipulate the process. A second identical submission is exactly the behavior an account-level guideline is written about.

Instead: make the changes, reply in Resolution Center **before** submitting the new build, list what changed screen by screen, attach screenshots, then submit.

```plaintext
We have addressed the issues as follows.

Visual identity: replaced the default component theme with our own palette,
radius, typography and spacing across the app.

Sign-in: redesigned with our own layout and copy, now continuous with
onboarding. Screenshots attached.

App icon and store screenshots: replaced. Screenshots now show the app in use
rather than the sign-in screen.

Metadata: reviewed the listing and removed claims we cannot evidence.

Review access: demo account added under App Review Information and verified
today. Notes for Review now describe each feature and where to find it.

The submitted build points at the same environment our users get. No
functionality is gated, remote-flagged or withheld from review.
```

No argument and no emotion. Evidence of change is what moves a 5.6 letter. If your account was actually terminated rather than the app rejected, the path back is the written improvement statement described in 5.6, and it should read like the note above with more detail. The [appeal path](https://developer.apple.com/contact/app-store/?topic=appeal) exists, but it is slower than fixing and resubmitting.

## The pre-submission pass

1.  Your app does not use the default theme of the tool that built it.
    
2.  Your sign-in screen is distinguishable from other apps built with the same tool.
    
3.  The icon was made deliberately, not generated and shipped.
    
4.  Screenshots show the app in use, not the login screen.
    
5.  Every button does something and every form can be completed.
    
6.  No placeholder text, no dead links.
    
7.  Every claim in your listing is true and evidenced.
    
8.  No incentivized reviews, no keyword manipulation, anywhere in your growth plan.
    
9.  The build points at an environment that exposes everything you are asking Apple to approve.
    
10.  Demo credentials verified today, backend live.
     
11.  Notes for Review name each feature and the path to reach it.
     

If you only have time for three, do 1, 2 and 11.

## This is not about how your app is built

It is easy to read a 5.6 rejection as a verdict on your stack. It is not. A poorly differentiated app written in Swift gets flagged on the same signals. Apple's stated inputs are app similarity, quality, accurate representation, and whether the app can be verified. None of those is a runtime.

The takeaway for anyone building with AI tools is that the tool gets you a working app quickly, and the last 10 percent, where the app stops looking like its scaffolding, is work you have to ask for. If you ship generic output and it gets rejected, the AI was not the problem. The 20 minutes nobody spent on the theme was.

## Get it on the stores

Once the app itself is review-ready, the remaining problem is getting the build signed and onto the stores.

Take the app you already built and ship it to iOS and Android as a real native binary, under your own developer account, with 50+ native device features available from your existing web codebase and content updates over the air. Code signing and submission run from the browser, with no Mac and no Xcode required.

[See the setup docs at setup.despia.com](https://setup.despia.com)