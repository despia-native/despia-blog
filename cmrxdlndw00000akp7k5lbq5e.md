---
title: "Turn a Vue Web App into a Mobile App"
seoTitle: "Turn a Vue Web App into a Mobile App"
seoDescription: "Convert a Vue web app into a native mobile app and publish it to the App Store and Google Play, with native device features and no Swift or Kotlin."
datePublished: 2026-07-23T10:36:02.099Z
cuid: cmrxdlndw00000akp7k5lbq5e
slug: turn-a-vue-web-app-into-a-mobile-app
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/c5f852f8-1153-47f9-b3fb-d95cbd7bbd02.png
tags: vuejs, webview, vue, webapp, pwa, vue3

---

Keep building in Vue. Despia ships that app as a native mobile app on the App Store and Google Play and opens up 50+ device features through one function. Here is the path, plus the composable pattern that makes it feel native to Vue.

## Install the package

```bash
npm install despia-native
```

Import the default export where you use it. Types come with the package:

```javascript
import despia from 'despia-native'
```

Installing `despia-native` adds the JavaScript bridge. Features that need native SDKs, such as OneSignal, RevenueCat, or Stripe, still need the matching Despia integration enabled and a fresh native build.

## Wrap detection in a composable

`despia()` only resolves inside the Despia app, so gate every call. In Vue the clean home for that gate is a composable other components import:

```javascript
// composables/useDespia.js
import despia from 'despia-native'

const ua = typeof navigator !== 'undefined' ? navigator.userAgent.toLowerCase() : ''
const isDespia = ua.includes('despia')
const isDespiaIOS = isDespia && (ua.includes('iphone') || ua.includes('ipad'))
const isDespiaAndroid = isDespia && ua.includes('android')

export function useDespia() {
  return { despia, isDespia, isDespiaIOS, isDespiaAndroid }
}
```

The `typeof navigator` guard matters on Nuxt or any SSR build, where the server has no `navigator`. On a plain client-only Vite app it is simply harmless.

## Call features from onMounted

Native work belongs in `onMounted`, after the component is on the client and the runtime is present. Here it reads live entitlements from the native store, no backend:

```vue
<script setup>
import { ref, onMounted } from 'vue'
import { useDespia } from '@/composables/useDespia'

const { despia, isDespia } = useDespia()
const premium = ref(false)

onMounted(async () => {
  if (!isDespia) return
  const { restoredData } = await despia('getpurchasehistory://', ['restoredData'])
  const active = (restoredData ?? []).filter((p) => p.isActive)
  premium.value = active.some((p) => p.entitlementId === 'premium')
})
</script>

<template>
  <Premium v-if="premium" />
  <Upsell v-else />
</template>
```

Fire-and-forget calls go inline in a handler. Launch the native paywall, or register the user for push:

```vue
<button @click="isDespia && despia(`revenuecat://launchPaywall?external_id=${userId}&offering=default`)">
  Go Premium
</button>
```

The paywall covers digital goods. Physical goods and real-world services take the opposite rail: Apple and Google take no cut and require Stripe, so anything that ships or gets performed goes through the native Stripe sheet:

```javascript
async function checkoutCart() {
  const res = await fetch('/api/create-payment-intent', { method: 'POST' })
  const { publishable_key, client_secret } = await res.json()
  if (isDespia) {
    despia(`stripe://payment?publishable_key=${publishable_key}&payment_intent_client_secret=${client_secret}`)
  }
}
```

## Register callbacks once

Native events land on `window.onXxx`. Assign them in `onMounted` and clear them in `onUnmounted` so you hold a single handler, not one per mount:

```javascript
import { onMounted, onUnmounted } from 'vue'

onMounted(() => { window.onRevenueCatPurchase = () => refreshEntitlements() })
onUnmounted(() => { window.onRevenueCatPurchase = undefined })
```

## Point your AI editor at the Despia docs

If you build your Vue components with Cursor, VS Code, Windsurf, or Claude, give it the Despia MCP so it writes against the real API instead of inventing calls. The remote server is the same everywhere:

```json
{
  "mcpServers": {
    "despia": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://setup.despia.com/mcp"]
    }
  }
}
```

Cursor reads `~/.cursor/mcp.json`, VS Code reads `.vscode/mcp.json` under a `servers` key, and any MCP-capable AI builder takes the URL `https://setup.despia.com/mcp` directly. Local editors need Node 18 or newer.

MCP is the best way to keep generation accurate. If your setup has no MCP support, the second-best option is to paste the relevant feature pages from `setup.despia.com` into the chat before you prompt.

## Add this to your app

Every native call here runs from your existing Vue codebase, one function away. No native project, no Xcode, and web changes reach users over the air.

[Read the full reference at setup.despia.com](https://setup.despia.com)