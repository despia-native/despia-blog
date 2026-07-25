---
title: "Building a Complete iOS App with Apple Sign-In, In-App Purchases, and Push Notifications in Under 2 Hours"
datePublished: 2025-09-10T20:03:38.008Z
cuid: cmfeeqefr000202jjbnruaycl
slug: building-a-complete-ios-app-with-apple-sign-in-in-app-purchases-and-push-notifications-in-under-2-hours
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/G3q7mxXkP-M/upload/cb2752dc78dd4e138dbca9ddc39940a4.jpeg
tags: native, lovable, despia

---

%[https://www.youtube.com/watch?v=lvtdUK7ucFo&t=4210s] 

Building a production-ready mobile app with authentication, payments, and real-time features typically takes weeks. But what if you could build it in an afternoon? In this tutorial, we'll walk through creating a complete iOS application with native Apple Sign-In, in-app purchases, and push notifications using Lovable, Supabase, and Despia.

## What We'll Build

By the end of this tutorial, you'll have a fully functional iOS app featuring:

* **Native Apple Sign-In** with the beautiful Apple JS modal (no ugly redirects)
    
* **In-app purchases** with RevenueCat integration
    
* **Real-time UI updates** via WebSockets with fallback polling
    
* **Push notifications** for purchase confirmations
    
* **Secure backend** with proper row-level security
    
* **Cross-device sync** for user accounts and purchases
    

## Why This Approach Matters

Most tutorials show you basic OAuth flows that redirect to external URLs, breaking the native app experience. We'll implement Apple JS directly to maintain that polished, native feel users expect from iOS apps.

## Setting Up the Foundation

### 1\. Create Your Lovable Project

Start with a blank Lovable project and connect it to Supabase. Lovable will automatically detect that you haven't created any tables yet and guide you through the setup process.

### 2\. Configure Apple Developer Account

Head to your Apple Developer Console and create the necessary identifiers:

**Create a Service ID:**

1. Go to Identifiers → Services IDs → Create new
    
2. Description: "My App Apple Auth"
    
3. Identifier: `com.yourcompany.myapp.appleauth`
    
4. Configure Sign in with Apple
    
5. Add your domains and return URLs:
    
    * Domains: Your Lovable app URLs (both published and preview)
        
    * Return URL: [`https://yourapp.com/oauth/apple`](https://yourapp.com/oauth/apple)
        

**Enable Apple Sign-In for Your App ID:**

1. Find your main App ID in the console
    
2. Enable "Sign in with Apple" capability
    
3. Configure as primary App ID
    

**Create an Authentication Key:**

1. Go to Keys → Create new key
    
2. Name: "My App Apple Auth"
    
3. Enable "Sign in with Apple"
    
4. Download the key file (you can only do this once!)
    

### 3\. Implement Apple Sign-In with Lovable

The key insight here is that we need to use Apple JS instead of Supabase's built-in OAuth to maintain the native experience:

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">Set up login with Apple using Apple JS. I need to use Apple JS because it's mandatory for my use case to avoid redirects that break the native workflow.</div>
</div>

Lovable will analyze your requirements and create a comprehensive implementation plan:

* Configure Apple Developer credentials
    
* Implement Apple JS SDK integration
    
* Create custom authentication flow
    
* Set up database schemas
    
* Handle user profile management
    

## Implementing the Payment System

### 4\. Set Up RevenueCat Integration

With authentication working, we can add in-app purchases. Use the Despia documentation prompt for RevenueCat:

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">The purchase gives users 10 coins. Set up the webhooks to add coins to the users' logged-in account in Supabase via RevenueCat webhooks.</div>
</div>

This creates:

* A `user_coins` table linked to authenticated users
    
* RevenueCat webhook endpoint for purchase events
    
* Automatic coin crediting system
    
* Real-time UI updates when purchases complete
    

### 5\. Configure RevenueCat Webhooks

In your RevenueCat dashboard:

1. Go to Integrations → Webhooks
    
2. Add your Lovable webhook URL
    
3. Set up authentication headers for security
    
4. Select "All Events" to capture purchases, renewals, and refunds
    

## Adding Real-Time Features

### 6\. Implement WebSocket Reliability

The challenge with real-time features is handling network interruptions. We implement a robust system:

* **Primary**: Supabase WebSocket subscriptions for instant updates
    
* **Fallback**: Polling every 30 seconds if WebSockets fail
    
* **Aggressive mode**: Enhanced polling for 5 minutes after purchase attempts
    
* **Smart reconnection**: Exponential backoff with automatic retry logic
    

### 7\. Add Push Notifications with OneSignal

For the ultimate user experience, we add native push notifications:

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">Implement OneSignal Player ID binding on app load for logged-in users. When the user does a RevenueCat payment event, send a push notification confirming their purchase.</div>
</div>

This creates:

* Device registration system linking OneSignal Player IDs to users
    
* Server-side push notification triggers
    
* Purchase confirmation notifications
    
* Cross-device notification support
    

## Security Best Practices

### 8\. Implement Row-Level Security

Lovable's security scanner will identify potential vulnerabilities:

* **Issue**: Edge functions with excessive database access
    
* **Solution**: Implement proper RLS policies
    
* **Result**: Users can only access their own data
    

### 9\. Handle Edge Cases

Real-world apps need to handle complex scenarios:

* **Email changes**: Users changing their iCloud email but keeping the same Apple account
    
* **Device switching**: Maintaining access across iPhone, iPad, Mac
    
* **Network issues**: Graceful degradation when connectivity is poor
    
* **Purchase failures**: Proper error handling and retry mechanisms
    

## The Results

What you get is remarkable:

* **Native experience**: Beautiful Apple Sign-In modal, no external redirects
    
* **Instant feedback**: Real-time coin updates and push notifications
    
* **Cross-platform sync**: Works seamlessly across all Apple devices
    
* **Production ready**: Proper security, error handling, and performance optimization
    
* **Development speed**: Built in under 2 hours instead of weeks
    

## Key Takeaways

1. **Use Apple JS over standard OAuth** for native iOS apps to maintain the polished user experience
    
2. **Implement robust fallback systems** - WebSockets are great when they work, but you need polling as backup
    
3. **Security scanning is essential** - AI can build fast, but you need to verify security practices
    
4. **Real-time updates require multiple strategies** - combine WebSockets, polling, and push notifications
    
5. **Testing edge cases matters** - plan for network issues, device changes, and user behavior variations
    

## What's Next?

This foundation gives you everything needed for a production iOS app. You could extend it with:

* Subscription management
    
* Advanced analytics
    
* Social features
    
* Content delivery
    
* Advanced notification targeting
    

The combination of Lovable's AI development, Supabase's backend infrastructure, and Despia's native capabilities makes it possible to build sophisticated mobile apps in record time without sacrificing quality or security.

**The future of app development is here, and it's faster than you think.**