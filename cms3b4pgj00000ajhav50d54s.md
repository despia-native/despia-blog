---
title: "Lovable App Permissions: Fix the 5.1.1 Rejection"
seoTitle: "Lovable App Permissions: Fix the 5.1.1 Rejection"
seoDescription: "Apple is auto-rejecting Lovable apps over purpose strings. Where Info.plist lives on each shipping route, what to clear, and how to resubmit."
datePublished: 2026-07-27T14:13:29.469Z
cuid: cms3b4pgj00000ajhav50d54s
slug: lovable-app-permissions-fix-the-5-1-1-rejection
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/05f51a08-7406-4566-a658-61487dd6c2de.png
tags: ios, ios-app-developer, ios-app-development, appstore, lovable, appreview

---

An automated message under Guideline 5.1.1 says your submission includes placeholder or otherwise insufficient purpose strings. It names no key, no resource, and no file, which is why the first hour goes into working out what it is talking about. These started landing around July 20, 2026, and the natural reading is that Apple has decided apps built this way do not belong on the store. That is not what is happening, and nothing in your Lovable project caused it.

## The short version

*   This is **Guideline 5.1.1 (Privacy: Data Collection and Storage)**, not Guideline 4.2. Lovable apps are not banned.
    
*   An automated pass reads the `Info.plist` in your binary and flags usage description strings that are generic, or that declare a resource your app never uses.
    
*   Native iOS and macOS apps are getting the same message in the same week, including apps whose strings have not changed in years.
    
*   Apps built on a hosted platform are caught more often, because the category compiles a blanket permission set into every binary.
    
*   The fix lives in the binary. New strings, a new build number, a resubmission. It cannot be pushed over the air, and there is nothing to edit in Lovable.
    
*   In Despia each permission is now a field under **App > Settings > Usage Descriptions**, and an empty field removes the key from iOS and Android together.
    

## What is a purpose string?

A purpose string, also called a usage description, is the sentence iOS shows the first time your app asks for a protected resource. It sits in `Info.plist` under a key named for that resource, such as `NSCameraUsageDescription` or `NSMicrophoneUsageDescription`, and it is compiled into the binary when the app is built.

Apple asks that the sentence describe what the app does with the data and give a specific example. Restating the permission dialog is not enough. The examples Apple publishes of strings that fail are the generic ones, a sentence saying the app would like access to your contacts, or that it needs microphone access.

## Why Lovable apps are not banned under Guideline 5.1.1

The check has no opinion about how your app was built or what renders it. It reads the plist and compares your declarations against what the app appears to do.

The clearest evidence that this is not a purge of hosted or AI-built apps comes from well outside that world. Indie Mac developer Matthias Gansrigler-Hrad [wrote up his week](https://blog.eternalstorms.at/2026/07/21/tales-from-app-review/). ScreenFloat was submitted to the Mac App Store on July 16, 2026 and approved the same day. A crash turned up, he fixed it with a one-line change, resubmitted, and the identical automated purpose-string rejection came back. The attempt after that was rejected by a human, over an address book entitlement App Review said had no matching functionality in the app. That is a native AppKit utility whose hand-written strings had shipped unchanged for years.

So this is a new or newly tightened automated pass running against every submission on both stores. Guideline 4.2 minimum functionality has not moved, and that remains the rule your app has to clear on its own merits.

## Why your Lovable app declared things it has never done

Here is the part my own category does not enjoy saying out loud.

Every platform that ships web apps as native binaries, ours included until this week, compiled a broad permission set into every build. Camera, microphone, photo library, location, contacts, calendar, Face ID, all of it. Not because your app used them, but because the vendor cannot know what you will build next, and the whole promise of the model is that web changes reach users without cutting a new binary. A string that is not compiled in makes the API unreachable, and no over-the-air update can put it back. So the template declared everything, every customer got the same plist, and it never had to be a decision.

That held up for over a decade. It was also over-declaration at scale, and an automated check hunting declared-but-unused resources reads a stock plist as a long list of violations.

A native developer writes strings by hand for the four or five resources they use, so wording is their worst case. Yours is a Lovable app declaring `NSContactsUsageDescription`, `NSFaceIDUsageDescription`, and `NSCalendarsUsageDescription` for features it has never had. Apple's message covers both failures, and both are live at once:

*   **Insufficient wording.** The string names the resource without describing the use.
    
*   **Declared but unused.** The key is present for a resource the app never reaches. This is what produced the second, human rejection in the ScreenFloat case, and it is what pulls hosted apps into the net.
    

The message will not tell you which key tripped it, so audit all of them.

## What changed in Despia on July 26, 2026

We rebuilt the permission system in under 24 hours. Every usage description in a Despia build is yours to edit or delete, under **Despia > App > Settings > Usage Descriptions**.

1.  **Edit any string.** Write your own, or hit Revert to Default to bring our wording back and edit from there.
    
2.  **Delete a permission by clearing the field.** An empty field removes the key from `Info.plist`, and it strips the corresponding feature, native code, and any dependencies or libraries it pulled in. Your exported Xcode and Android Studio projects come out clean instead of carrying dead capability.
    
3.  **Android follows iOS.** The plist keys are mapped to the Android manifest. Fill in an iOS usage description and the matching Android feature and permission are written into the manifest. Clear it and both come out. One field, two platforms.
    

One warning that matters more than it should: the field has to be genuinely empty. A single space counts as a value and the key stays in the build. Check it twice before you submit.

### Every usage description key you can now control

| Despia field | iOS Info.plist key | Prompt appears when |
| --- | --- | --- |
| Camera | `NSCameraUsageDescription` | The app opens the camera for photo or video capture |
| Microphone | `NSMicrophoneUsageDescription` | The app records audio |
| Photo Library | `NSPhotoLibraryUsageDescription` | The app reads the photo library, such as an image picker |
| Save to Photos | `NSPhotoLibraryAddUsageDescription` | The app only writes images to the library and never reads it |
| Location While Using | `NSLocationWhenInUseUsageDescription` | The app first requests location with the app open |
| Background Location | `NSLocationAlwaysAndWhenInUseUsageDescription` | The app asks for always-on location while it already has when-in-use |
| Always-On Location | `NSLocationAlwaysUsageDescription` | Legacy always-on prompt for iOS 10 and earlier |
| Contacts | `NSContactsUsageDescription` | The app reads the address book |
| Calendar | `NSCalendarsUsageDescription` | The app reads or creates calendar events |
| Face ID | `NSFaceIDUsageDescription` | The app first authenticates with Face ID |
| Health Read | `NSHealthShareUsageDescription` | The app reads Health data such as steps or workouts |
| Health Write | `NSHealthUpdateUsageDescription` | The app writes samples back to the Health app |
| Speech Recognition | `NSSpeechRecognitionUsageDescription` | Audio is sent for speech-to-text transcription |
| NFC Scanning | `NFCReaderUsageDescription` | The app reads an NFC tag |

Most of these have a matching Android permission that Despia writes into the manifest when the field is filled and removes when it is empty. Camera maps to the camera permission, Microphone and Speech Recognition to record audio, Contacts to read contacts, Calendar to the calendar read and write pair, Location to fine, coarse, and background location, Face ID to biometrics, and NFC to the NFC permission. The Health keys are iOS only, since HealthKit has no Android counterpart in a Despia build.

Take that table and go down it against the app you have shipped, not the one you plan to ship. Anything with no feature behind it goes.

## How to write a purpose string that passes review

The strings Despia ships as defaults are built in the shape Apple asks for: a verb, the data, and the outcome the user gets.

*   Camera: `Opens the camera so you can take photos and videos in the app.`
    
*   Microphone: `Records audio so you can capture voice notes in the app.`
    
*   Photo Library: `Opens your photo library so you can pick images to upload.`
    
*   Location While Using: `Reads your location so you can see yourself on the map and find nearby results.`
    
*   Health Read: `Reads your steps and workouts so you can see your activity in the app.`
    

What the app touches, and what the user gets because of it. A string that only names the resource fails, since it tells the user nothing the permission dialog did not already say.

Where your app does something more specific, say the specific thing. A generic string is not an automatic rejection, but the specific version is what the guideline asks for and it converts better on the prompt:

*   Weak: `Opens the camera so you can take photos and videos in the app.`
    
*   Strong: `Opens the camera so you can photograph a receipt and attach it to an expense claim.`
    

Write them yourself or hand the field to a model. The copy is yours now, not ours.

## The trap: deleting a permission you will need later

Read this before you start clearing fields, because it is the part that catches people who ship quickly.

Usage descriptions live in the binary. Your Lovable app ships over the air. The two update on completely different clocks, and permissions are where that asymmetry bites.

Clear the microphone field to pass review, cut a new build, get approved. Two weeks later you build a voice note feature in Lovable and it reaches users within minutes. On iOS, reaching a protected resource with no matching usage description does not reject the promise, it terminates the process. The user taps the button and the app closes. There is no runtime workaround and no over-the-air fix, because the fix is in the binary. Android is more forgiving and will usually deny the permission rather than kill the app, but iOS is the platform that decides whether you ship.

Two honest ways to handle it.

**The old way, which the industry has run on for a decade.** Keep the declaration for anything you plan to ship inside this binary's lifetime, and write the string as if the feature is already there. No engineer is going to congratulate you for it. It is also, right now, precisely the pattern the automated check was built to find, and the second ScreenFloat rejection shows a human reviewer following up on declared-but-unused capability after the automated pass cleared. This route gets less viable by the month.

**The way that survives the new check.** Delete what you do not use, and gate the feature on the app version when you add it. Despia hands your web code the version number you set in the dashboard, the human-readable one like `1.2.0`, so your Lovable app can tell which native capabilities are compiled into the shell it is running inside:

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

Keep the two numbers straight. The version is the one you set in Despia and the one your code gates on, and it moves when the native layer gains a capability. The build number is what App Store Connect wants incremented on every upload, including a resubmission that changes nothing else.

Bump the version when you add the usage description, ship that binary, then push the Lovable update that turns the feature on. Users still on `1.1.0` never see the button, so they never reach the API that would kill their app, and users on `1.2.0` get the feature the moment it lands. The version gate does the job the blanket declaration used to do by brute force.

The rule that falls out of it: your plist should describe what your app does in this binary, and adding a native capability is one of the few things that legitimately costs a rebuild. Everything else still ships over the air.

## How to fix the rejection, step by step

1.  Open **Despia > App > Settings > Usage Descriptions** and work through every key. Anything your app does not use, clear the field. Confirm it is empty and not a space.
    
2.  Rewrite what remains so each string names the data and the outcome. Do not leave a default sitting on a resource that is central to your app.
    
3.  Cut a new build with a new build number.
    
4.  Upload it and resubmit. Replying in Resolution Center without a new build will not clear this, because the plist is compiled into the binary. There is nothing to change in Lovable and nothing to change on your hosting.
    
5.  If a native capability is coming soon, add its string now, in this build, and gate the feature on the app version rather than shipping the declaration and hoping nobody looks.
    

## FAQ

**Are Lovable apps banned from the App Store?** No. This is Guideline 5.1.1, a privacy rule that applies to every app on the store, and native iOS and macOS apps are getting it in the same week. Guideline 4.2 minimum functionality is unchanged, and it remains the rule your app has to clear.

**Do I need to change anything in my Lovable project?** No. Purpose strings are part of the native build. There is nothing on your web app or your hosting to change.

**What does "placeholder or otherwise insufficient purpose strings" mean?** Either the string is too generic to describe the actual use, or the key is declared for a resource your app never touches. Both fail the same automated check.

**Which purpose string did Apple flag?** Apple does not say. The message names no key and no resource, so you have to audit all of them.

**Can I fix a 5.1.1 rejection over the air?** No. Usage descriptions are compiled into the binary, so it takes a new build and a resubmission. Everything on the web side of your app still updates over the air.

**Do I need to upload a new build if my last submission was rejected?** Yes. A rejected build cannot be edited. Change the strings, increment the build number, upload, and resubmit.

**How do I remove a permission from Info.plist?** Clear the field under Despia > App > Settings > Usage Descriptions and rebuild. The key leaves `Info.plist`, the matching Android permission leaves the manifest, and the feature code is stripped from the exported native projects.

**Does removing a permission break my app?** It breaks any feature that reaches the removed resource. On iOS the app terminates rather than failing gracefully, which is why you gate new native capabilities on the app version instead of assuming everyone is on the latest binary.

**Does this affect Android too?** Google does not run this check, but over-declared permissions are a Data Safety liability there and they cost you installs at the permission screen. The usage description fields map to the Android manifest, so cleaning up one cleans up both.

**Will a well-written string still get rejected if the app does not use the resource?** Yes, and that is the second failure mode. Wording does not save a declaration with no matching functionality behind it.

## Add this to your app

Every permission your Lovable app declares is now a field you control, on both platforms, from one place. No Xcode, no Android Studio, and no native project to keep in sync.

[Read the full reference at setup.despia.com](https://setup.despia.com)