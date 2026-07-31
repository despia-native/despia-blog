---
title: "Lovable App Store Error: Invalid URL Scheme Fix"
seoTitle: "Lovable App Store Error: Invalid URL Scheme Fix"
seoDescription: "Your Lovable app fails upload with an invalid URL scheme error. Why the project name causes it, and how to set a valid scheme without renaming."
datePublished: 2026-07-31T14:05:50.143Z
cuid: cms90m9p800000ajbamez0o58
slug: lovable-app-store-error-invalid-url-scheme-fix
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/35f50eb0-3459-413d-90b9-ac493a09dba4.png
tags: ios, ios-app-development, ai-tools, aitools, iosdevx, lovable

---

Your app works. It runs at its Lovable URL, the auth flow completes, the payments go through. You packaged it, uploaded the binary, and got a validation failure naming a URL scheme in a format Apple will not accept. You did not choose that string. Nothing in your Lovable project mentions it. Here is where it came from and how to change it.

## This is a validation failure, so nobody reviewed anything

The distinction is worth getting straight before you start worrying about your submission.

```plaintext
ERROR ITMS-90158: The following URL schemes found in your app are not in the correct format: [4week]
```

ITMS-90158 fires in Apple's delivery pipeline, before the binary is queued for anyone to look at. No reviewer opened your app, no App Store guideline was cited, and there is no appeal to write. The rule it enforces comes from RFC 1738: the value has to begin with a letter, and everything after can only be letters, digits, periods, hyphens, or plus signs.

There is a sibling error, ITMS-90155, for a scheme that is correctly formatted but claimed by another platform. `fb` is the usual one. Different cause, same outcome.

Neither costs you a review cycle. You fix the value, upload again, and the pipeline stops complaining. Google Play does not run this check at all, which is why the Android build of the same app goes through while iOS refuses it.

## Your project name is doing this

Lovable projects get named early and casually, usually as a slug: `4-week-plan`, `my_fitness_app`, `café-mode`. That name follows the app into the packaging step, which derives the scheme from it, and the derived value inherits whatever the name had in it.

| Project name | Derived value fails because | Set instead |
| --- | --- | --- |
| 4 Week Plan | Starts with a digit | `weekplan` |
| My\_Fitness\_App | Underscores are outside the allowed set | `myfitness` |
| Café Mode | Accented characters are rejected | `cafemode` |
| FB Tools | Claimed prefix, caught by ITMS-90155 | `fbtools` |

Hyphens survive the check, so a kebab-case name is not the problem on its own. Digits at the front, underscores, spaces, and anything non-ASCII are.

The instinct at this point is to rename the app so the derivation produces something legal. Do not. App Store Connect is perfectly happy with digits, spaces, and accents in a display name, and your name is a product decision that should not be overruled by a string format rule from 1994. Change the derived value and leave the name where it is.

## The advice you will find assumes an Xcode project

Search the error and every result says the same thing: open the iOS project, find Info.plist, edit the CFBundleURLTypes entry, rebuild. That works if you maintain a native project. You do not. Your source of truth is a web app in Lovable, and the native project is generated for you, which means there is no file on your machine to edit and a hand edit would be regenerated away.

So the real question is not how to edit the plist. It is whether the packaging step exposes the value to you at all. If it derives the scheme silently and offers no field, your only lever is the app name, and that is a bad lever. If it is a field, this stops being an incident.

## Setting the scheme

In Despia it is a field. Open the Despia Editor, go to App, Settings, Dynamic App Source, and edit URL Scheme. Enter the value alone, with no `://` and no path, so `weekplan` registers `weekplan://`. Trigger a build and upload that one.

The build matters more than the edit. The scheme is compiled into the native binary, so changing the field does nothing to the binary already sitting in App Store Connect and nothing to the copy on your test device. Until a fresh build exists, the old value is still the only one that exists.

Pick something reasonably distinctive while you are there. Values like `app`, `mobile`, and `auth` pass validation and then collide with whatever else on the device claimed them, at which point iOS routes to whichever app was installed first and Android shows a picker. Validation will not warn you about that one.

## Supabase login is what breaks if you get this wrong later

Most Lovable apps authenticate through Supabase, and native social login is the one place the scheme is load bearing. The provider opens in ASWebAuthenticationSession on iOS and Chrome Custom Tabs on Android. Both are isolated from your app on purpose, and the only way back in is a link that uses your scheme.

Change the scheme without updating your OAuth configuration and login fails in the most confusing way available. The Google screen loads, the user signs in, and then the sheet just sits there. No error, no redirect, no clue.

Two places hold the old value. The redirect URIs registered in your provider dashboards, and the callback page that fires the return link. The first has to be updated by hand. The second does not, if you stop hardcoding it and carry the scheme through the OAuth `state` parameter, which providers echo back untouched.

```javascript
// generating the OAuth URL
const state = `${crypto.randomUUID()}|weekplan`  // csrf token, scheme

// on /native-callback, inside the authentication session
const scheme = state.includes('|') ? state.split('|')[1] : 'weekplan'
window.location.href = `${scheme}://oauth/auth?access_token=${encodeURIComponent(token)}`
```

If the app is already live, register the new redirect URI alongside the old one before shipping. A build registers one scheme, so anyone still on the previous binary keeps answering to the old value until they update, and both have to work while that happens. Before your first submission none of this applies, which is the case for fixing the value now rather than after you have users.

## Get your Lovable app on the stores

Your Lovable app stays the source of truth. Despia ships it as a real iOS and Android binary with native device access, updates the web layer over the air without a resubmission, and keeps the settings that decide whether a build passes validation as fields rather than plist entries.

[See the setup docs at setup.despia.com](https://setup.despia.com)