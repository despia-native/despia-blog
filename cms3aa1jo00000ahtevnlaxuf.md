---
title: "Placeholder Purpose Strings: Fix the 5.1.1 Rejection"
seoTitle: "Placeholder Purpose Strings: Fix the 5.1.1 Rejection"
seoDescription: "Apple now auto-rejects apps over placeholder purpose strings. What triggers the Guideline 5.1.1 check, why WebView apps get caught, and how to fix it."
datePublished: 2026-07-27T13:49:38.802Z
cuid: cms3aa1jo00000ahtevnlaxuf
slug: placeholder-purpose-strings-fix-the-5-1-1-rejection
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/9b4a92bf-a38a-4ed6-93c2-4affffe9317a.png
tags: ios, ios-app-developer, ios-app-development, appstore, appreview

---

Since around July 20, 2026, submissions have been coming back with an automated message about placeholder or otherwise insufficient purpose strings, filed under Guideline 5.1.1. The message names no key, no resource, and no file. If you shipped a WebView app this week you probably received it, and the first read is that Apple has decided to clear the category out. That is not what happened. The reason WebView apps are getting hit at a higher rate is real, and it is worth understanding before you change anything.

## The short version

*   This is **Guideline 5.1.1 (Privacy: Data Collection and Storage)**, not Guideline 4.2. WebView apps are not banned.
    
*   An automated pass reads the `Info.plist` in your binary and flags usage description strings that are generic, or that declare a resource your app never uses.
    
*   It is hitting native iOS and native macOS apps in the same week, including apps whose strings have not changed in years.
    
*   WebView and wrapper platforms get caught more often because the whole category ships a blanket permission set in every binary.
    
*   The fix is in the binary, so it takes new strings, a new build number, and a resubmission. It cannot be pushed over the air.
    
*   In Despia this is now a field per permission under **App > Settings > Usage Descriptions**, with an empty field removing the key from iOS and Android at once.
    

## What is a purpose string?

A purpose string, also called a usage description, is the sentence iOS shows a user when your app first asks for a protected resource. It lives in `Info.plist` under a key named for the resource, such as `NSCameraUsageDescription` or `NSMicrophoneUsageDescription`, and it is compiled into the app binary at build time.

Apple's requirement is that the sentence describes what the app does with the data and gives a specific example. A string that only restates the permission dialog fails. Apple's own published examples of insufficient strings are the generic forms, along the lines of an app saying it would like access to your contacts, or that it needs microphone access.

## Why WebView apps are not banned under Guideline 5.1.1

The check does not look at your rendering layer. It reads the plist and compares your declarations against what the app appears to do.

The clearest evidence that this is not a category purge comes from outside the WebView world entirely. Indie Mac developer Matthias Gansrigler-Hrad [wrote up his week](https://blog.eternalstorms.at/2026/07/21/tales-from-app-review/): ScreenFloat was submitted to the Mac App Store on July 16, 2026 and approved the same day. He then found a crash, fixed it with a one-line change, resubmitted, and got back the identical automated purpose-string rejection. His next attempt was rejected again, this time by a human, over an address book entitlement that App Review said had no matching functionality in the app. That is a native AppKit screenshot utility with hand-written purpose strings that had shipped unchanged for years.

So: a new or newly tightened automated pass, applied to every submission on both stores, native and WebView alike. Guideline 4.2 minimum functionality has not moved, and it remains the rule a WebView app actually has to clear.

## Why wrapper apps get caught more often than native ones

Here is the part my own category does not like saying out loud.

Every WebView platform, Despia included until this week, declared a broad permission set in every binary it produced. Camera, microphone, photo library, location, contacts, calendar, Face ID, the lot. Not because your app used them, but because the vendor cannot know which web APIs you will reach for six months from now, and the entire promise of the model is that you ship web changes over the air without cutting a new binary. If the string is not compiled in, the API is not reachable, and no over-the-air update can add it. So the template declared everything, every customer got the same plist, and nobody had to think about it.

That worked for over a decade. It was also, viewed honestly, over-declaration at scale, and an automated check that flags declared-but-unused resources reads a stock wrapper plist as a long list of things to reject.

Native developers hand-write strings for the four or five resources they actually use, so their worst case is wording. For a wrapper app the worst case is a fitness app declaring `NSContactsUsageDescription`, `NSFaceIDUsageDescription`, and `NSCalendarsUsageDescription` for features it has never had. Apple's message covers both failures and both are in play at once:

*   **Insufficient wording.** The string names the resource without describing the use.
    
*   **Declared but unused.** The key is present for a resource the app never reaches. This is what produced the second, human rejection in the ScreenFloat case, and it is what puts wrapper apps in the net.
    

The automated message does not tell you which key tripped it, so assume the audit covers all of them.

## What changed in Despia on July 26, 2026

We rebuilt the permission system in under 24 hours. Every usage description in a Despia build is now yours to edit or delete, under **Despia > App > Settings > Usage Descriptions**.

1.  **Edit any string.** Write your own copy, or hit Revert to Default to drop our wording back in and edit from there.
    
2.  **Delete a permission by clearing the field.** An empty field removes the key from `Info.plist`, and it also strips the corresponding feature, native code, and any dependencies or libraries it pulled in. Your exported Xcode and Android Studio projects come out clean rather than carrying dead capability.
    
3.  **Android follows iOS.** The plist keys are mapped to the Android manifest. Fill in an iOS usage description and the matching Android feature and permission are written into the manifest. Clear it and both come out. One field, two platforms, no Xcode and no Android Studio.
    

One warning that matters more than it should: the field has to be genuinely empty. A single space counts as a value and the key stays in the build. Check it twice before you submit.

### Every usage description key you can now control

| Despia field | iOS Info.plist key | Prompt appears when |
| --- | --- | --- |
| Camera | `NSCameraUsageDescription` | The app opens the camera for photo or video capture |
| Microphone | `NSMicrophoneUsageDescription` | The app records audio |
| Photo Library | `NSPhotoLibraryUsageDescription` | The app reads from the photo library, such as an image picker |
| Save to Photos | `NSPhotoLibraryAddUsageDescription` | The app only writes images to the library and never reads it |
| Location While Using | `NSLocationWhenInUseUsageDescription` | The app first requests location with the app open |
| Background Location | `NSLocationAlwaysAndWhenInUseUsageDescription` | The app asks for always-on location while it already has when-in-use |
| Always-On Location | `NSLocationAlwaysUsageDescription` | Legacy always-on prompt for iOS 10 and earlier |
| Bluetooth | `NSBluetoothAlwaysUsageDescription` | The app connects to Bluetooth accessories |
| Contacts | `NSContactsUsageDescription` | The app reads the address book |
| Calendar | `NSCalendarsUsageDescription` | The app reads or creates calendar events |
| Face ID | `NSFaceIDUsageDescription` | The app first authenticates with Face ID |
| Health Read | `NSHealthShareUsageDescription` | The app reads Health data such as steps or workouts |
| Health Write | `NSHealthUpdateUsageDescription` | The app writes samples back to the Health app |
| Speech Recognition | `NSSpeechRecognitionUsageDescription` | Audio is sent for speech-to-text transcription |
| NFC Scanning | `NFCReaderUsageDescription` | The scan sheet opens to read an NFC tag |
| Tracking (ATT) | `NSUserTrackingUsageDescription` | The App Tracking Transparency prompt appears, before the app tracks you across other apps and websites |
| Push Notifications | None | Your own priming screen, shown ahead of the system notification prompt |

Two rows behave differently from the rest.

**Tracking** is the App Tracking Transparency string, and it is required if your app collects an advertising identifier. Ship an ad identifier without it and you fail review on a different front, so this is one to check rather than reflexively clear.

**Push Notifications** has no `Info.plist` key at all. iOS does not allow custom text in the system notification alert, so this string is yours to use on a priming screen shown before the system prompt. Clearing it does not remove a declaration, because there is no declaration to remove.

Most of these have a matching Android permission that Despia writes into the manifest when the field is filled and removes when it is empty. Camera maps to the camera permission, Microphone and Speech Recognition to record audio, Contacts to read contacts, Calendar to the calendar read and write pair, Location to fine, coarse, and background location, Face ID to biometrics, Bluetooth to the Bluetooth permissions, and NFC to the NFC permission. Three have no Android side: the Health keys, because HealthKit has no Android counterpart in a Despia build, Tracking, because App Tracking Transparency is an iOS framework, and Push Notifications, because that string is your own priming copy rather than a system declaration.

## How to write a purpose string that passes review

Look at the shape of the strings Despia ships as defaults, because it is the shape Apple asks for. Each one is a verb, the data, and the outcome the user gets:

*   Camera: `Opens the camera so you can take photos and videos in the app.`
    
*   Microphone: `Records audio so you can capture voice notes in the app.`
    
*   Photo Library: `Opens your photo library so you can pick images to upload.`
    
*   Location While Using: `Reads your location so you can see yourself on the map and find nearby results.`
    
*   Health Read: `Reads your steps and workouts so you can see your activity in the app.`
    

The formula is what the app touches, and what the user gets because of it. A string that only names the resource fails, because it tells the user nothing the permission dialog did not already say.

Rewrite the defaults if your app does something more specific. A generic string is not automatically a rejection, but a specific one is what the guideline asks for, and it converts better on the prompt:

*   Weak: `Opens the camera so you can take photos and videos in the app.`
    
*   Strong: `Opens the camera so you can photograph a receipt and attach it to an expense claim.`
    

Write these by hand or hand the field to a model. The point of the change is that the copy is yours now, not ours.

## The trap: deleting a permission you will need later

This is where people are about to hurt themselves, so read it before you start clearing fields.

Usage descriptions live in the binary. Your web code ships over the air. Those two things update on completely different clocks, and the permission system is where that asymmetry bites.

Say you clear the microphone field to pass review, cut a new build, get approved. Two weeks later you push a web update that calls `getUserMedia` for a voice note feature. On iOS, touching a protected resource with no matching usage description is not a denied promise, it is a terminated process. The user taps the button and your app disappears. There is no runtime workaround and no OTA fix, because the fix is in the binary. Android is more forgiving and will usually fail the permission request instead of killing the app, but iOS is the platform that decides whether you ship.

There are two honest ways to handle this.

**The old way, which the industry has run on for a decade.** Keep the declaration for anything you plan to ship inside this binary's lifetime, and write the string as if the feature is already there. No engineer is going to congratulate you for it. It is also, right now, exactly the pattern the automated check was built to find, and the second ScreenFloat rejection shows a human reviewer following up on declared-but-unused capability after the automated pass cleared. This route is getting less viable by the month.

**The way that survives the new check.** Delete what you do not use, and gate the feature on the app version when you add it. Despia hands your web code the version number you set in the dashboard, the human-readable one like `1.2.0`, so the web layer can tell which native capabilities are compiled into the shell it is running inside:

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

// "1.2.0" -> 10200, so versions compare as plain numbers
const versionValue = (v) => v.split('.').reduce((acc, part) => acc * 100 + Number(part), 0)

let micAvailable = false

if (isDespia) {
  const { versionNumber } = await despia('getappversion://', ['versionNumber'])
  micAvailable = versionValue(versionNumber) >= versionValue('1.2.0')
}

if (micAvailable) {
  showVoiceNoteButton()
}
```

Keep the two numbers straight while you do this. The version is the one you set in Despia and the one your web code gates on, and it changes when the native layer gains a capability. The build number is what App Store Connect wants incremented on every upload, including a resubmission that carries no capability change at all.

Bump the version in Despia when you add the usage description, ship that binary, then push the web update that turns the feature on. Users still on `1.1.0` never see the button, so they never reach the API that would kill their app, and users on `1.2.0` get the feature the moment it lands. The version gate does the job the blanket declaration used to do by brute force.

The rule that falls out of this: your plist should describe what your app does in this binary, and adding a native capability is one of the few things that legitimately costs a rebuild. Everything else still ships over the air.

## How to fix a purpose string rejection, step by step

1.  Open **Despia > App > Settings > Usage Descriptions** and go through every key. Anything your app does not use, clear the field. Confirm it is empty and not a space.
    
2.  Rewrite what remains so each string names the data and the outcome. Do not leave a default in place for a resource that is central to your app.
    
3.  Cut a new build with a new build number.
    
4.  Upload it and resubmit. Replying in Resolution Center without a new build will not clear this, because the plist is compiled into the binary. There is nothing on your web app or your hosting platform to change.
    
5.  If you plan to add a native capability soon, add its string now, in this build, and gate the feature on the app version rather than shipping the declaration and hoping nobody looks.
    

## FAQ

**Are WebView apps banned from the App Store?** No. This is Guideline 5.1.1, a privacy rule that applies to every app on the store, and it is hitting native iOS and macOS apps in the same week. Guideline 4.2 minimum functionality is unchanged, and it remains the rule a WebView app has to clear.

**What does "placeholder or otherwise insufficient purpose strings" mean?** Either the string is too generic to describe the actual use, or the key is declared for a resource your app never touches. Both fail the same automated check.

**Which purpose string did Apple flag?** Apple does not say. The automated message names no key and no resource, so you have to audit all of them.

**Can I fix a 5.1.1 purpose string rejection over the air?** No. Usage descriptions are compiled into the binary, so it takes a new build and a resubmission. Everything on the web side of your app still updates over the air.

**Do I need to upload a new build if my last submission was rejected?** Yes. A rejected build cannot be edited. Change the strings, increment the build number, upload, and resubmit.

**How do I remove a permission from Info.plist in a WebView app?** In Despia, clear the field under App > Settings > Usage Descriptions and rebuild. The key leaves `Info.plist`, the matching Android permission leaves the manifest, and the feature code is stripped from the exported native projects.

**Does removing a permission break my app?** It breaks any feature that reaches the removed resource. On iOS the app terminates rather than failing gracefully, which is why you gate new native capabilities on the app version instead of assuming everyone is on the latest binary.

**Does this affect Android too?** Google does not run this check, but over-declared permissions are a Data Safety liability there and they cost you installs at the permission screen. Despia maps the iOS usage descriptions to the Android manifest, so cleaning up one cleans up both.

**Will a well-written string still get rejected if the app does not use the resource?** Yes, and that is the second failure mode. Wording does not save a declaration with no matching functionality behind it.

## Add this to your app

Every permission your app declares is now a field you control, on both platforms, from one place. No Xcode, no Android Studio, and no native project to keep in sync.

[Read the full reference at setup.despia.com](https://setup.despia.com)