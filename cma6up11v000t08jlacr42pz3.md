---
title: "How to Avoid Apple's 30% Fee After the Court Ruling using Despia"
seoTitle: "How to Avoid Apple's 30% Fee After the Court Ruling using Despia"
seoDescription: "A step-by-step guide for Despia app developers to implement external payment processing"
datePublished: 2025-05-02T13:49:46.771Z
cuid: cma6up11v000t08jlacr42pz3
slug: how-to-avoid-apples-30-fee-after-the-court-ruling-using-despia
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/f-3mUXFLY2o/upload/1fefebda8559170d09f17056339b945d.jpeg
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1746194962662/7d762b95-eb5f-4c51-8aae-65417415198b.png
tags: apple, app-store

---

The recent court ruling has opened new possibilities for app developers to bypass Apple's 30% commission by implementing external payment processing. This guide explains how Despia developers can take advantage of this opportunity while maintaining a seamless user experience.

## What You'll Need

* A Despia app
    
* A third-party payment processor account (like Stripe, PayPal, or Polar)
    
* Access to your website domain + hosting
    

## Setting Up App Links

First, you need to set up App Links to ensure your app can handle specific URLs, creating a smooth transition between web and app experiences:

1. Create a .json file named `apple-app-site-association` with the following content:
    

```json
{
  "applinks": {
    "details": [
      {
        "appIDs": [
          "{TEAM-ID}.{BUNDLE-ID}"
        ],
        "components": [
          {
            "/": "/*",
            "comment": "Matches any URL path, so all links open the app."
          }
        ]
      }
    ]
  }
}
```

2. Host this file at `/.well-known/apple-app-site-association` on your website's root directory. This special path is recognized by iOS to handle app links properly.
    

## Configuring Your Despia App

Next, configure your Despia app to handle links correctly:

1. In the Despia Editor, navigate to "Link Handling"
    
2. Add your payment processor URL into the "Always Open in Browser" field to meet Apple's external policy compliance requirements
    
3. Enable Associated Domains in Despia (for more details on associated domains, refer to [this informative video about app links](https://www.youtube.com/watch?v=ENEwGyJPvPo))
    

## Implementing the Payment Flow

Now, let's implement the payment flow that bypasses Apple's 30% commission:

1. Generate a checkout link in your app that includes necessary parameters like the user account ID (refer to your payment processor's documentation for specifics)
    
2. Open this link using `window.location.href = "https://your-payment-processor.com/pay/123"`
    
3. The link opens in Safari, allowing the user to complete payment outside of your app, avoiding Apple's commission
    
4. Set your success URL to be under the same host your Despia app runs on
    
5. After payment completion, the success page will open directly in your app, creating a smooth app → web → app flow
    

This approach feels almost 100% native, especially since users can also utilize Apple Pay via your third-party processor.

## Important Notes

* This workaround currently only applies to apps in the United States due to regional differences in the court ruling
    
* Your success URL must be under the same domain as your app to maintain the seamless transition back to the app
    
* Be sure to thoroughly test this implementation across different iOS versions
    

By following this guide, you can offer your users alternative payment options while avoiding Apple's 30% commission. The approach creates a professional user experience that integrates web payments seamlessly with your Despia app.

*Note: Always review the latest App Store guidelines and legal requirements as they continue to evolve following the court ruling.*