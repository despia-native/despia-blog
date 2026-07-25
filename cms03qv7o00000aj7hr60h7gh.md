---
title: "Despia MCP Server: Stop Burning Credits on Bad Code"
seoTitle: "Despia MCP Server: Stop Burning Credits on Bad Code"
seoDescription: "Same model, same prompt, two very different outcomes. The Despia MCP server gives your AI builder the real API surface instead of a confident guess."
datePublished: 2026-07-25T08:23:27.895Z
cuid: cms03qv7o00000aj7hr60h7gh
slug: despia-mcp-server-stop-burning-credits-on-bad-code
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/8416592e-9f36-4b74-9a7f-be3a187e2b8e.png
tags: reactjs, vue, cursor, mcp, claude-code

---

Same model. Same prompt. Two completely different outcomes. The only variable is whether it can read the docs.

Without a reference, an AI builder does not tell you it is unsure about a native API. It writes something confident, you spend a build turning it into a binary, you install it, and the feature does nothing. So you go back to the chat and describe the failure, and it apologises and writes something else that is also wrong. Four laps later you are convinced the runtime is broken.

It is not. It guessed. That is the left side of the picture.

## What the guessing actually costs

|  | Without the MCP | With the MCP |
| --- | --- | --- |
| Environment check | `window.despia`, which does not exist | User agent check that returns true on device |
| Native calls | Invented method names | Real `despia-native` calls |
| Your feedback loop | Prompt, build, install, fail, repeat | Prompt, build, ship |
| Debugging | Chasing a bug that is not in the runtime | Reading an actual error |

Every lap of that loop is a prompt you paid for, a build you spent, and an hour you are not getting back. Some of them are a store submission.

## The honest tradeoff first

MCP is not free at the message level and anyone claiming otherwise has not read the platform docs. Base44 states outright that MCP connections can increase credit usage, because the model processes more information per request. That is true.

It is also the wrong number to optimise. A few hundred extra tokens per message against four wasted rounds plus the builds attached to them is not a close call. You are not paying for context. You are paying for the loop that missing context creates.

## The three things it invents

These come through support almost every week.

`window.despia`**.** Does not exist, never has, is not a public API. An agent writes it as an environment check, the condition is always false on every device, and every native call inside that block is skipped in silence. Nothing errors. The feature is just dead. The real check is the user agent:

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

if (isDespia) despia('successhaptic://')
```

`despia()` calls go inside that gate. Global callbacks (`window.onXxx`) are assigned outside it, so they exist before the native side fires them.

**Capacitor scaffolding.** Models that have seen a lot of Capacitor generate Capacitor plugin config and a second native project next to your web app. None of it runs in Despia. You are left maintaining files that describe a build you do not have.

**Logs for calls it never wrote.** The expensive one, because it burns your time rather than just your credits. The agent adds a diagnostics panel reporting that it invoked an SDK method, got a promise back, and is awaiting a callback. Search the bundle and there is no such call anywhere. The panel reads like instrumentation and is closer to fan fiction. If your diagnostics claim a function ran, grep for that function before you believe a single line of it.

## The right side is one URL

The Despia MCP server is a documentation server. It hands your builder the actual reference for the `despia-native` package: which features exist, what the calls look like, and the correct patterns for push, biometrics, widgets, storage, GPS, camera, and the rest of the 50+ native features.

```plaintext
https://setup.despia.com/mcp
```

That is the whole migration. No restructuring, no native folder, one web codebase, same as before. Your agent stops inventing method names because it can read the real ones.

## Setup, by tool

Paths are current as of July 2026. The shape is identical everywhere: connectors screen, custom server, paste the URL, no authentication.

### Lovable

1.  Open **Connectors**, go to **Chat connectors**.
    
2.  Click **New MCP server**.
    
3.  **Server name**: Despia
    
4.  **Server URL**: `https://setup.despia.com/mcp`
    
5.  **Authentication**: **No authentication**. The docs server needs no credentials.
    
6.  Click **Add server**.
    

Chat connectors are per-user, so everyone on the project connects their own. Available on all plans.

### Base44

1.  Profile icon, top right.
    
2.  **Settings**, then **MCP connections** under Account.
    
3.  Click **Add custom MCP**.
    
4.  **Name**: Despia. **URL**: `https://setup.despia.com/mcp`. **Authentication**: Not required. Leave headers empty.
    
5.  Click **Test**, then **Test & add**.
    

Needs a Builder plan or higher, and Base44 caps you at 20 connected servers per account.

### Bolt

1.  Open the **Connectors (MCP)** page.
    
2.  Click **Custom MCP server**.
    
3.  Enter the URL and transport, no authentication.
    
4.  Click **Connect** and wait for the Connected status.
    

Tick **Auto-enable for all projects** if you build more than one Despia app. Otherwise you will enable it per project and forget on the one that matters.

### Cursor, VS Code, Windsurf, Claude Desktop

Same JSON, different file. Cursor reads `~/.cursor/mcp.json`, VS Code reads `.vscode/mcp.json` in the workspace, Windsurf takes it under Settings, Plugins, Add MCP Server, and Claude Desktop uses `claude_desktop_config.json`.

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

Node.js v18 or later. Restart the editor after saving.

### No MCP support in your tool

Paste the relevant pages from [setup.despia.com](https://setup.despia.com) into the chat before you prompt. Manual, and you redo it when context resets, but it beats letting the model improvise.

## Check that it actually took

Connecting a server and having the agent use it are two different things. Before you spend another build, ask it:

> How do I detect that my app is running inside Despia, and what package do I import for native features?

You want the user agent check and `despia-native`. If it says `window.despia`, or reaches for a Capacitor plugin, it is not reading the server. Reconnect it, and on tools where the agent picks its own tools, name it in the prompt: use the Despia MCP.

Then run one search of your project for the function you expect to be called. Not the log line describing it, the call itself. That grep takes five seconds and catches most of what would otherwise cost you a build.

## Get it on the stores

Take the app you just built and ship it to iOS and Android without a CLI or a Mac. Code signing and submission run from the browser.

[See the setup docs at setup.despia.com](https://setup.despia.com)