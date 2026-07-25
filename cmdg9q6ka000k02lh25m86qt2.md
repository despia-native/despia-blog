---
title: "What is Despia?"
datePublished: 2025-07-23T17:59:37.402Z
cuid: cmdg9q6ka000k02lh25m86qt2
slug: what-is-despia
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1753293232219/7332682f-3c25-47f3-ae05-1a97c9d25be0.png
tags: apple, ios, mobile-app-development, native, android

---

%[https://www.youtube.com/watch?v=OnsNQektzmg] 

In the world of mobile app development, there's always been a stark divide: build native apps with platform-specific languages, or settle for web-based solutions with limited device access. Despia is changing this narrative by introducing a sophisticated native runtime that transforms any web application into a genuine native mobile app – without sacrificing performance or functionality.

## What is Despia Really?

### Beyond the Webview Stigma

Let's address the elephant in the room: Yes, Despia uses a webview at its core. But before you close this tab, understand that Despia is to webviews what a Formula 1 car is to a go-kart – they might share some basic principles, but the engineering, performance, and capabilities are worlds apart.

Despia is a comprehensive native framework that:

* Provides a GPU-powered rendering engine with hardware acceleration
    
* Includes pre-built native SDKs for extensive device functionality
    
* Offers modular native targets for advanced features like widgets and app clips
    
* Handles the entire build and distribution pipeline automatically
    

## How Despia Actually Works

### The Four Pillars of Native Power

Despia's architecture consists of several sophisticated components working in harmony:

#### 1\. GPU-Powered Rendering Engine

At its heart, Despia uses Apple's WKWebView and Android's WebView, but with extensive hardware acceleration tuning. This isn't your typical webview wrapper – it's a highly optimized rendering engine that achieves smooth 60fps performance through GPU acceleration.

#### 2\. Pre-Built Native SDKs

Unlike traditional webview apps, Despia packages extensive native functionality directly into the compiled application. From biometric authentication to background location tracking, these aren't plugins you need to install – they're part of the runtime itself.

#### 3\. The Core Framework

This defines build phases, configurations, and native behaviors like status bar appearance and navigation gestures. It's the foundation that makes your web app feel truly native.

#### 4\. Modular Native Targets

Want widgets? App clips? Share extensions? These are conditionally compiled based on your needs, ensuring your app only includes what it uses.

### The Window.Despia Bridge: Zero Dependencies, Maximum Power

The magic happens through the `window.despia` interface – but this isn't just another JavaScript API. It's a sophisticated protocol handler system that bridges your web code directly to native functionality.

#### No NPM, No Problem

**Here's the beauty of it: The Despia SDK doesn't require any npm packages or JavaScript dependencies.** There's nothing to install in your web codebase. The SDK is compiled directly into the native Swift and Java/Kotlin code that Despia generates.

#### Behind the Magic

When you write:

```javascript
window.despia = "feature://parameters"
```

Here's what happens:

1. Protocol intercepted by Despia's runtime
    
2. Securely routed to native SDK
    
3. Permissions handled automatically
    
4. Results returned to JavaScript
    

#### Real Native Features, Real Simple Code

Real examples:

```javascript
// Create a home screen widget with live data
const svg = "https://your-backend.com/api/widget"
const refresh_time = 10  // in minutes
window.despia = `widget://${svg}?refresh=${refresh_time}`

// Request push notification permission
window.despia = "registerpush://"

// Install an eSIM card
window.despia = `esim://?carddata=${cardData}`
```

This elegantly simple approach means:

* No complex libraries to import
    
* No dependency management headaches
    
* No version conflicts
    
* Works with any framework or no-code platform
    

## Universal Platform Support

### Build Once, Run Everywhere

Despia works with any web stack:

* **WeWeb** - Build visually and deploy natively
    
* **Webflow** - Turn your designs into apps
    
* **Wized** - Add complex logic and interactions
    
* **Nordcraft (Toddle)** - Advanced no-code development
    
* **Any JavaScript Framework** - React, Vue, Angular, Svelte
    
* **Any hosting** - Vercel, Netlify, Cloudflare
    

If it has a URL, Despia can turn it into a native app.

## Despia V3: Where Performance Meets Simplicity

### The Game-Changing Features

The latest version introduces game-changing enhancements to the runtime:

### 1\. GPU-Accelerated Everything

V3's rendering engine delivers consistent 60fps performance through advanced hardware acceleration. Animations, transitions, and scrolling feel indistinguishable from apps built with Swift or Kotlin.

### 2\. Smart Caching That Learns

The V3 runtime intelligently manages:

* Asset preloading and storage
    
* State persistence across sessions
    
* Predictive resource caching
    

Result: Instant load times that feel native because they are native.

### 3\. Native Processing Power

V3's most innovative feature: offload compute-intensive tasks from JavaScript to native code. Currently supporting biometric verification and haptic feedback, with AI processing capabilities planned.

### 4\. Platform-Perfect UX

V3 automatically enhances your web app with:

* Native touch gesture refinement
    
* Platform-specific scroll physics
    
* Contextual haptic feedback
    
* OS-matched swipe navigation
    

### 5\. Beyond the App

V3 enables features most native developers struggle with:

* **WidgetKit Widgets**: Live home screen widgets
    
* **App Clips**: Instant experiences without download
    
* **Share Extensions**: Receive content from other apps
    
* **3D Touch Shortcuts**: Quick actions from home screen
    

### 6\. One-Click Everything

Despia runs on M2 Mac minis in the cloud:

* Automatic iOS and Android builds
    
* Code signing with your certificates
    
* Direct store submission
    
* Over-the-air updates
    
* Full source code export
    

## Why This Approach Solves the Native Gap

### The Problem Web Developers Face

Web applications have always struggled to access device-level features that native apps take for granted:

* Background processes and location tracking
    
* Hardware access like biometrics and haptics
    
* System integration (widgets, app clips, share extensions)
    

This creates a capability gap between what users expect from mobile apps and what web technologies can traditionally deliver.

### Despia's Solution: Instant Native Powers

Despia closes this gap without the traditional complexity:

* **One line of code** unlocks each native feature
    
* **Zero dependencies** - no npm packages or libraries to manage
    
* **Cross-platform by default** - iOS and Android from the same code
    
* **No-code compatible** - works perfectly with WeWeb, Wized, Webflow, Nordcraft and others
    
* **Future-proof** - as Despia adds SDKs, your implementation stays the same
    

### Compared to React Native

React Native uses "real" native UI elements but suffers from the JavaScript bridge bottleneck. Ironically, Flutter (which powers FlutterFlow) uses zero native UI elements – it draws everything using the Skia engine via GPU, similar to how Despia's GPU-accelerated webview renders HTML/CSS.

### Compared to Flutter/FlutterFlow

Both Flutter and Despia use GPU rendering for UI elements. The key differences:

* Flutter can achieve 120fps vs Despia's 60fps (though this is rarely noticeable)
    
* Despia offers instant over-the-air updates without app review
    
* Despia provides web development flexibility and ecosystem access
    

## Real-Time Updates: The Game Changer

One of Despia's killer features is instant updates. Change your paywall, fix a bug, or roll out new features – your changes go live immediately without App Store approval. This combines the best of Expo and CodePush but with fewer limitations since it's built on web technologies.

## The Bottom Line

Despia isn't trying to hide what it is – it's a sophisticated native runtime built on webview technology, engineered to deliver native performance and functionality. Just as modern cars still use wheels like horse carriages did, the underlying technology matters less than the implementation.

With thousands of native files, GPU acceleration, extensive native SDKs, and modular architecture, Despia delivers:

* True native performance (60fps)
    
* Full device feature access
    
* Instant updates without app review
    
* Single codebase for all platforms
    
* No native development knowledge required
    

## Getting Started with V3

Ready to experience the future of mobile development? Despia V3 is now available at [v3.despia.com](https://v3.despia.com/). Whether you're building with WeWeb, Webflow, or vanilla JavaScript, you can have a native app running in under an hour – no Mac required, no native code to write.

*As the creator says: "I wanted the freedom of building a web application with the competitive edge of native features." That's exactly what Despia delivers.*