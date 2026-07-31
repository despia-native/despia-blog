---
title: "Base44 App Store Error: Invalid URL Scheme Fix"
seoTitle: "Base44 App Store Error: Invalid URL Scheme Fix"
seoDescription: "A Base44 app can fail upload on an invalid URL scheme derived from its name. What the error means and how to set a valid scheme without renaming."
datePublished: 2026-07-31T14:08:47.965Z
cuid: cms90q2x800000ajghyik9yep
slug: base44-app-store-error-invalid-url-scheme-fix
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/6e34fa9d-f893-4a83-9d3d-660de1d3fc51.png
tags: ios, ios-app-development, iosdevx, base44

---

There is a specific kind of submission failure that feels much worse than it is. You upload the binary, and instead of the processing email you get a validation error naming a URL scheme, in a format Apple refuses, containing a string you have no memory of writing. It reads like a rejection. It is not one. Your app was never delivered, so nobody reviewed it, and the fix is a single field.

## ITMS-90158 is a pipeline error, not a guideline

```plaintext
ERROR ITMS-90158: The following URL schemes found in your app are not in the correct format: [4fit]
```

This comes out of Apple's delivery validation, which runs long before a human is involved. The rule it applies is old and simple, from RFC 1738: a scheme starts with a letter, and everything after it is letters, digits, periods, hyphens, or plus signs. Anything else is refused at the door.

ITMS-90155 is the adjacent error for a scheme that is formatted correctly but already claimed, which is what catches `fb` and a few other short prefixes no matter who is uploading.

The upside of both is that there is nothing to appeal and no review cycle burned. You change the value, upload again, and it processes. Google Play does not run this validation, which is why the Android build of the same app goes out fine and only iOS objects.

## The value is derived from your app name

Nobody sits down and chooses `4fit://`. The packaging step generates the scheme from the app name on the first build, and the name was chosen by whoever was thinking about the product rather than about a URL format specification. A fitness app called 4 Fit Club derives a value starting with a digit. A shop called Café Mode derives one with an accent in it. An internal tool called My\_App derives one with an underscore. All three are legal app names and none of them are legal schemes.

| App name | Derived value fails because | Set instead |
| --- | --- | --- |
| 4 Fit Club | Starts with a digit | `fitclub` |
| My\_App | Underscores are outside the allowed set | `myapp` |
| Café Mode | Accented characters are rejected | `cafemode` |
| FB Tools | Claimed prefix, caught by ITMS-90155 | `fbtools` |

The obvious move is to rename the app until the derivation produces something valid. That is the wrong trade. App Store Connect accepts digits, spaces, and accents in a display name without complaint, and your product name should not be decided by a validator. Change the scheme, keep the name.

## Whether you can fix this depends on what the packaging step exposes

Search the error code and every answer describes the same procedure: open the iOS project, edit the CFBundleURLTypes entry in Info.plist, rebuild. That is the right answer for someone with an Xcode checkout on their machine. It is not an answer available to you. Your app is a web app in Base44, the native project is generated on your behalf, and a hand edit to a generated file is overwritten the next time it regenerates.

Which reduces the question to one thing. Does the tool that builds your binary give you the scheme as a setting, or does it derive it silently from the name and leave you with nothing to change but the name? If the value is a black box, an invalid derivation is not something you can route around from inside your app. If it is a field, this is a thirty second edit and a rebuild.

## Setting the scheme

In Despia the scheme is a field. Open the Despia Editor, go to App, Settings, Dynamic App Source, and edit URL Scheme. Enter the value on its own, with no `://` and no path, so `fitclub` registers `fitclub://`. Then trigger a new build, because the scheme is compiled into the binary and the field on its own changes nothing. The build already in App Store Connect and the copy installed on your phone both still answer to the old value until you replace them.

While you are choosing, pick something distinctive. Generic values like `app`, `mobile`, and `auth` clear validation and then compete with every other app that made the same choice. On a device where two apps claim the same scheme, iOS routes to whichever was installed first and Android shows a disambiguation dialog, and neither is something your code can detect or correct.

## What breaks if you change the scheme after launch

The scheme is the return path from a native authentication session. Social login opens the provider in ASWebAuthenticationSession on iOS and Chrome Custom Tabs on Android, both deliberately isolated from your app, and a link using your scheme is the only thing that closes the session and hands the tokens back.

Change the scheme and forget the OAuth configuration, and login fails without producing an error anywhere. The provider screen loads, the user authenticates, and the session sits open with no route home.

Two places hold the old value: the redirect URIs registered with each provider, and whatever fires the return link from your callback page. The first has to be updated in each dashboard. The second only bites you if the scheme is hardcoded there, which is avoidable. Carry it through the OAuth `state` parameter instead, since providers return `state` unchanged.

```javascript
// generating the OAuth URL
const state = `${crypto.randomUUID()}|fitclub`  // csrf token, scheme

// on /native-callback, inside the authentication session
const scheme = state.includes('|') ? state.split('|')[1] : 'fitclub'
window.location.href = `${scheme}://oauth/auth?access_token=${encodeURIComponent(token)}`
```

For a live app, register the new redirect URI alongside the old one before you ship the build. One binary registers one scheme, so users who have not updated keep answering to the old value and both have to work until adoption catches up. None of this applies before your first submission, which is the reason to check the value now rather than after you have an install base.

## Get your Base44 app on the stores

Your Base44 app stays the source of truth. Despia ships it as a real iOS and Android binary with native device access, updates the web layer over the air with no resubmission, and keeps the settings that decide whether a build clears validation as fields you can edit rather than files you cannot reach.

[See the setup docs at setup.despia.com](https://setup.despia.com)