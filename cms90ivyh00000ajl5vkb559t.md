---
title: "Invalid URL Scheme: Fix the ITMS-90158 Upload Error"
seoTitle: "Invalid URL Scheme: Fix the ITMS-90158 Upload Error"
seoDescription: "Your build fails with an invalid URL scheme before review ever sees it. What ITMS-90158 means, what triggers it, and how to fix it in a minute."
datePublished: 2026-07-31T14:03:12.351Z
cuid: cms90ivyh00000ajl5vkb559t
slug: invalid-url-scheme-fix-the-itms-90158-upload-error
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/e5a73643-bdd6-4d7e-97f3-ae3938922892.png
tags: ios-apps, ios, ios-app-development

---

You uploaded the binary, waited for the processing email, and got a failure notice instead. Somewhere in it is a line about URL schemes not being in the correct format, naming a string you may not remember choosing. This is not an App Review rejection. No reviewer opened your app, no guideline was cited, and there is nothing to appeal. Apple's delivery pipeline read one value out of your build, decided it was malformed, and refused the upload. The fix is usually a single character.

## The error comes from validation, not from App Review

Two codes cover almost every case of this.

```plaintext
ERROR ITMS-90158: The following URL schemes found in your app are not in the correct format: [4fit]
```

ITMS-90158 means the scheme your app registers does not match the format Apple accepts. The accompanying text points at RFC 1738 and spells out the rule: the value has to start with a letter, and the rest can only be letters, digits, periods, hyphens, or plus signs.

ITMS-90155 is the neighbouring error and it means something different. That one fires when the scheme is well formed but claimed by someone else, which is why `fb` and a handful of other short prefixes get bounced regardless of who is uploading.

Both are better news than they look. A validation failure costs you a re-upload. A review rejection costs you days and a reply thread with someone who is not permitted to tell you which line of code they meant.

Google Play does not check this at upload time, which is why the same app sails through on Android and stops dead on iOS. The invalid value still does not work on Android. It fails quietly at runtime instead of loudly at submission.

## What actually makes a scheme invalid

The rule is narrow enough that most violations come down to one of four characters.

| Value | Why it fails | Use instead |
| --- | --- | --- |
| `4fit` | Starts with a digit | `fitapp` |
| `my_app` | Underscores are not in the allowed set | `myapp` |
| `my app` | Spaces are never valid | `myapp` |
| `café` | Accented and non-ASCII characters are rejected | `cafeapp` |
| `fb` | Well formed, but claimed by another platform | `myapp` |

Uppercase is not a validation failure, but it is not meaningful either. The operating system normalises the value, so treat it as case-insensitive and write it lowercase to avoid confusing yourself later. Hyphens pass validation, so a kebab-case value is legal even though it reads oddly next to `://`.

The other rule that catches people is uniqueness, and validation will not save you from it. Values like `app`, `mobile`, `auth`, and your framework's name pass the format check and then collide with something else on a real device. When two installed apps claim the same value, iOS hands the link to whichever was installed first and Android shows a disambiguation dialog. Neither outcome is recoverable from your code.

## Nobody types `4fit://` on purpose

If you never picked a scheme, you are not alone, and that is exactly how this error finds people. Tooling that packages a web app for the stores generates the value from your app name on the first build. The name was a marketing decision made by someone who was not thinking about RFC 1738, so a gym app called 4 Fit Club produces `4fit`, a shop called Café Mode produces an accented string, and an internal tool called My\_App produces an underscore. Every one of those uploads and fails.

The name itself is fine. App Store Connect is happy with digits, spaces, and accents in a display name. The problem is only the derived value, which means renaming your app to satisfy a validator is the wrong trade. Change the scheme and leave the name alone.

## Every search result tells you to edit a file you do not have

Search the error code and the answers are unanimous: open the iOS project, find Info.plist, edit the CFBundleURLTypes entry, rebuild. That is correct advice for someone maintaining an Xcode project. It is useless if your app is a web app that something else packages for you, because there is no checkout on your machine to edit, and any change you did make by hand would be overwritten the next time the pipeline generated the project.

The version of this question that matters is not how do I edit the plist. It is whether the value is exposed to you at all. If the packaging step derives it from your app name and gives you no field, your only lever is the app name itself, which is a bad lever. If it is a field, this is a thirty second fix.

## The fix

In Despia the scheme is a field. Open the Despia Editor, go to App, Settings, Dynamic App Source, and edit URL Scheme. Enter the value on its own, with no `://` suffix and no path, so `myapp` registers `myapp://`. Trigger a new build and upload it.

That last part is the step people skip. The scheme lives in the native binary, so editing the field changes nothing about the build already sitting in App Store Connect, and nothing about the copy installed on your phone. You need a fresh build for the new value to exist anywhere.

## The one thing that breaks when you change it

The scheme is how a native authentication session hands control back to your app. Social login opens the provider in ASWebAuthenticationSession on iOS and Chrome Custom Tabs on Android, both isolated from your app by design, and the only route back is a link using your scheme. Change the scheme without touching your OAuth configuration and login breaks in the least helpful way possible: the provider screen loads, the user signs in, and then nothing happens.

Two things need updating. Every redirect URI registered in your provider dashboards, and any callback page with the old value written into it.

The second one is avoidable. Rather than hardcoding the scheme in a page that runs inside the authentication session, carry it through the OAuth `state` parameter, which providers echo back unchanged.

```javascript
// generating the OAuth URL
const state = `${crypto.randomUUID()}|myapp`  // csrf token, scheme

// on /native-callback, inside the authentication session
const scheme = state.includes('|') ? state.split('|')[1] : 'myapp'
window.location.href = `${scheme}://oauth/auth?access_token=${encodeURIComponent(token)}`
```

If your app is already live, register the new redirect URI alongside the old one before you ship. A build registers one scheme, so users on the previous binary keep answering to the old value until they update, and both have to work while adoption catches up. Before your first submission none of this applies, which is the argument for checking the value now rather than after launch.

## Get it on the stores

Take the app you already built and ship it to iOS and Android without a CLI or a Mac. Code signing and submission run from the browser, and the settings that decide whether a build passes validation are fields rather than plist entries.

[See the setup docs at setup.despia.com](https://setup.despia.com)