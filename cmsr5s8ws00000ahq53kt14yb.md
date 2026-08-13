---
title: "Migrate from Lovable Cloud to Supabase"
seoTitle: "Migrate from Lovable Cloud to Supabase"
seoDescription: "Move a Lovable Cloud app onto a Supabase project you own: schema, data, users, storage and Stripe webhooks, with copy-paste prompts for every step."
datePublished: 2026-08-13T06:50:18.295Z
cuid: cmsr5s8ws00000ahq53kt14yb
slug: migrate-from-lovable-cloud-to-supabase
cover: https://cdn.hashnode.com/uploads/covers/67f0239ff4f5a0b7b4a852c4/d9f56071-48a1-4916-8bbc-6eabad40eeab.png
tags: supabase, lovable

---

Lovable Cloud runs on Supabase, but the project is not yours. This guide moves your app onto a Supabase account you own: schema, data, users with working passwords, files and Stripe webhooks, with no lost events. You paste 9 prompts, your AI tool does the work, and the same app ships to the App Store afterwards.

Lovable Cloud is the fastest way to get a working backend behind an app you described in a chat window. It is also a backend you do not hold the keys to. At some point you want the database, the auth users, the storage bucket and the Stripe webhook in a project on your own Supabase account, with your own billing and your own region.

This post walks that migration end to end, and you do not need to be a developer to follow it. You will not type a single command. Every step is a prompt you paste into one of 2 windows: Lovable chat, or an AI coding tool running on your computer. The tool does the work, you move the messages between them.

The end state: the same app, running against your own Supabase project, with every user ID unchanged, every password still working, every file in place, and not one Stripe event lost along the way.

Everything here is packaged as an open source agent skill: [github.com/despia-native/lovable-to-supabase](https://github.com/despia-native/lovable-to-supabase). It installs a runbook plus 7 scripts into your project, and your AI tool runs them for you.

## What you are actually moving

A Lovable Cloud project is Supabase underneath, which is the good news. The pieces that need to move:

| Piece | How it moves |
| --- | --- |
| Schema (tables, functions, triggers, security policies) | Dumped from the source project and restored into yours |
| Data | Same dump, with `auth.users` loaded first so user IDs stay stable |
| Auth users | Password hashes travel, so everyone's existing password keeps working |
| Storage buckets and files | Copied object by object |
| Edge functions | Redeployed from your repo |
| Cron jobs | Recreated with the new project URL |
| Stripe webhooks | A second endpoint runs in parallel, then the old one is retired |

Stable user IDs are the load-bearing detail. Every row in your database that belongs to a user points at that user's ID, and every security policy compares against it. Preserve those IDs and the whole app keeps working after the move. Regenerate them and you own a data-repair project instead of a migration.

## 2 windows, and you are the wire

Migrations like this go wrong when people try to do all of it in one place. Lovable's sandbox cannot reach your new project's database port, and it must never hold your service-role key or your database password. So every step belongs to exactly one of 2 windows:

**Lovable chat** handles code changes: repointing the app, adding a table, guarding the webhook handler.

**Your AI coding tool** handles everything that touches real credentials: the dumps, the restore, the file copy, the Stripe replay. This is Cursor, Claude Code or Codex, running on your machine with your project open. If you have never used one, Cursor is the easiest starting point, and installing it is the only setup this migration needs.

Lovable and your coding tool cannot talk to each other. You are the wire between them. Each prompt below tells you which window it goes into, and most of them end with a strict reply format, so you can paste the answer straight back into the other window without having to understand it.

Lovable never sees the skill, and does not need to. The skill and the runbook are installed only in your coding tool. Every block you paste into Lovable spells out the whole change inline, down to the exact table columns, so Lovable only ever does the 1 code change in front of it. Nothing to look up, nothing to fetch, no credentials involved.

What Lovable does get is 1 line of background at the top of each block: that the backend is moving to a Supabase project you own, and that the migration itself runs somewhere else. That is enough to stop it inventing its own migration plan or asking you for a database password, without inviting it into work it cannot do from a sandbox.

One rule holds the security model together: the only credentials that ever go into a Lovable prompt are your new project URL and publishable key, because both ship inside your app anyway. Service-role keys, database passwords and Stripe secret keys go into 1 file on your computer and nowhere else. Not into a chat window, not into a screenshot.

| Prompt | Paste it into |
| --- | --- |
| 1\. Set up and look inside | Your AI coding tool |
| 2\. Make webhooks exactly-once | Lovable chat |
| 3\. Add the reconciliation endpoint | Lovable chat |
| 4\. Copy the database and users | Your AI coding tool |
| 5\. Copy files, functions and cron jobs | Your AI coding tool |
| 6\. Run the cutover | Your AI coding tool |
| 7\. Flip the app to the new backend | Lovable chat |
| 8\. Verify everything | Your AI coding tool |
| 9\. Fix whatever failed | Lovable chat |

## Before you start: 3 things to open

**1\. Your project on your own computer.** Download your code from Lovable (GitHub export or the download option), open the folder in your AI coding tool, and leave it open. Everything in the coding-tool lane happens there.

**2\. A new Supabase project.** Go to [supabase.com/dashboard](https://supabase.com/dashboard), create a project, and pick the same region as your Lovable project. Matching regions keeps the file copy fast and your app's latency identical.

**3\. Your Lovable backend credentials.** In Lovable, open the Cloud tab and its advanced settings, where the data export and the direct database connection string live. The service-role key sits in the same area. That UI moves around, so if a value is not exposed, ask Lovable support for it. It is your data.

You will paste those values into 1 file later, in your editor, not into any chat.

## Prompt 1: set up and look inside your backend

This one is long because it teaches your coding tool the rules for the whole migration. The rest are short.

**Paste this into your AI coding tool running locally** (Cursor, Claude Code, Codex)

```plaintext
CONTEXT
This repo is my app, exported from Lovable. Its backend is Lovable Cloud
(Supabase under the hood) and I am moving it onto a Supabase project I own.
I am not a developer. You have a terminal on my machine, so you run every
command yourself. Never hand me a command to type unless there is genuinely
no alternative, and if there is not, give me the exact line and tell me where
to run it.

SOURCE OF TRUTH
Read https://github.com/despia-native/lovable-to-supabase before doing
anything. Install its skills/lovable-to-supabase/MIGRATION.md and
skills/lovable-to-supabase/scripts/migration/ into this repo and follow that
runbook exactly. Use its numbered handoff blocks as written. Do not improvise
steps it does not have, and ask me rather than guessing.

MY SETUP
My new Supabase project ref: <PASTE IT HERE>
Payments: <my own Stripe account | Lovable-managed | none>

TASK, THIS SESSION ONLY
1. Install the runbook and the scripts into this repo and make them runnable.
2. Add .env.migration and scripts/migration/dumps/ to .gitignore BEFORE
   creating any file that will hold credentials.
3. Create scripts/migration/.env.migration from the example, fill in only the
   values you can read from this repo, then STOP. List for me, line by line,
   every value I still need to enter and exactly where in the Supabase or
   Lovable dashboard to find each one.
4. After I confirm the file is filled in, run the inventory script and explain
   the result in plain English: how many users and tables I have, which tables
   have no security policies, what scheduled jobs exist, and what settings my
   edge functions expect.

RULES
Never print, open or echo .env.migration, the dump files, or anything under
scripts/migration/dumps/.
Never ask me to paste a password, a secret key or a connection string into
this chat.
Never continue past a step the runbook marks as a gate.
Stop and tell me when something looks wrong instead of working around it.
```

When it stops and asks you to fill in the credentials file, open `scripts/migration/.env.migration` in your editor, paste each value after its equals sign, and save. That file is the only place your secrets live, and it is already excluded from your code history.

The inventory is worth reading rather than skimming. 2 things matter most. A table with no security policy is a hole that lets any signed-in user read everyone's rows, and it is better fixed before the move than after. And a scheduled job with a service-role key written into it is a leaked key, which should be rotated.

## Prompts 2 and 3: prepare the app while it is still on Lovable

These 2 changes happen on your current backend, before anything is copied, so they travel across with the schema. They are what makes payments safe during the move: your webhook stops being able to process the same Stripe event twice, and you gain an endpoint that can re-check every subscription against Stripe at any time.

**Paste this into Lovable chat** (make webhook processing exactly-once)

```plaintext
You are working on my app which currently uses Lovable Cloud (Supabase under
the hood). Background, for context only: I am moving this app's backend to a
Supabase project I own. The database copy and the Stripe cutover run on my
own machine, not here, so do not attempt them and do not ask me for keys or
connection strings. Your job is this one code change.

Task: make Stripe webhook processing idempotent so every event is
processed exactly once, even if Stripe delivers it twice. Do not change any
other behaviour.

1. Create a database migration adding a table public.webhook_events:
   id uuid primary key default gen_random_uuid(),
   stripe_event_id text not null unique,
   type text not null,
   status text not null default 'processing'
     check (status in ('processing','processed','failed')),
   error text,
   received_at timestamptz not null default now(),
   processed_at timestamptz
   Enable row level security on it and create NO policies: only the
   service-role client (which bypasses RLS) may touch it.

2. In the Stripe webhook edge function, keep signature verification exactly
   as it is. Immediately after the event is verified and parsed, claim it
   with the service-role client:
     insert into webhook_events (stripe_event_id, type)
     values (<event.id>, <event.type>)
     on conflict (stripe_event_id) do update
       set status = 'processing', error = null
       where webhook_events.status = 'failed'
     returning id
   - No row returned, the event was already handled or is being handled:
     return 200 with {"received": true, "duplicate": true} and do nothing else.
   - Row returned: run the existing processing, then set status='processed',
     processed_at=now() on that row.
   - Processing threw: set status='failed' with the error message and return
     500 so Stripe retries later.

Reply with ONLY a fenced text block containing:
STATUS: done | blocked
MIGRATION_FILE: <path of the SQL migration created>
WEBHOOK_FUNCTION: <path of the edge function changed>
NOTES: <one line, or "-">
```

**Paste this into Lovable chat** (add the reconciliation endpoint)

```plaintext
Background, for context only: I am moving this app's backend to a Supabase
project I own, and the migration itself runs on my own machine, not here. Do
not attempt it and do not ask me for keys or connection strings. Your job is
this one function.

Add a new edge function named reconcile-subscriptions to this project. It
pulls subscription truth from Stripe and upserts it into the database, so we
can re-sync at any time. Requirements:

1. POST only. Authenticate via header x-reconcile-secret compared to the
   RECONCILE_SECRET function secret using a constant-time comparison; return
   401 with an empty body on mismatch. Set verify_jwt = false for this
   function (it authenticates via the header instead).
2. Body: {"limit": number (max 100, default 50),
          "starting_after": string | null,
          "customer": string | null}.
3. Using the Stripe secret key already configured as a function secret, list
   subscriptions with status "all" (paginated with limit/starting_after; if
   "customer" is set, only that customer's subscriptions).
4. For each subscription, resolve the app user the same way the existing
   Stripe code does (stripe customer id column, or customer email), then
   upsert the app's subscription record with the service-role client:
   subscription status, current period end, price id, stripe_customer_id,
   stripe_subscription_id. Update-or-insert only; never delete rows.
5. Return JSON:
   {"processed": <subscriptions seen>, "updated": <rows written>,
    "next_cursor": <last subscription id if there are more, else null>}
6. If the platform lets you set function secrets, generate a strong random
   RECONCILE_SECRET and set it; otherwise tell me so I set it myself.

Reply with ONLY a fenced text block containing:
STATUS: done | blocked
FUNCTION_URL: <full https URL of the deployed function>
SECRET_SET: yes | no
TABLES_TOUCHED: <table names it upserts into>
NOTES: <one line, or "-">
```

Paste both replies back into your coding tool. The strict reply format exists so that neither side has to interpret prose, and so you never have to judge whether an answer was good enough.

Skip both if your app takes no payments.

## Prompt 4: copy the database, the data and the users

**Paste this into your AI coding tool**

```plaintext
Continue the migration, Phase 3 of MIGRATION.md: dump the schema, dump the
data, and restore both into my new Supabase project. Run the scripts
yourself.

The row count check is a hard gate. Do not continue past it. When it
finishes, show me the comparison as a simple table and tell me in one
sentence whether every table matched.
```

Behind that prompt: the restore loads users first so their IDs survive, turns off foreign key checks during the load, runs the whole thing as a single transaction, and then counts every table on both sides and compares.

If a count does not match, do not push on. It usually means something was still writing to the old backend during the dump, and the runbook's troubleshooting section covers the rest. Your tool should say so rather than shrugging.

## Prompt 5: copy files, functions and scheduled jobs

**Paste this into your AI coding tool**

```plaintext
Continue with Phase 4 through Phase 7 of MIGRATION.md:
1. Copy every storage bucket and every file to the new project. If anything
   fails, re-run the copy until it reports 0 failures, and tell me if it
   cannot get there.
2. Deploy the edge functions to the new project.
3. List every secret name the functions need, and tell me exactly where in
   the Supabase dashboard to set each one. Do not ask me for the values here.
4. Recreate the scheduled jobs with the new project URL and key.
Then give me a plain checklist of everything I still have to set by hand in
the Supabase dashboard, with where to click for each.
```

That last line matters. Auth configuration is the one part of a Supabase project that lives in no backup: your site URL, your redirect URLs, every social login with its own client credentials, and your email settings. None of it is copied by anything, and it is easy to discover at the worst moment. Get the checklist now, work through it before cutover.

## The Stripe step you do by hand

Skip if you take no payments. This is 5 minutes of clicking and it is what makes the cutover risk-free.

1.  Open the Stripe dashboard, go to Developers, then Webhooks.
    
2.  Add an endpoint pointing at your new project's webhook function URL. Your coding tool can tell you that URL.
    
3.  Subscribe it to exactly the same events as your existing endpoint.
    
4.  Leave the old endpoint switched on.
    
5.  Copy the new endpoint's signing secret, the value starting `whsec_`, and set it in 2 places: the new project's edge function secrets in the Supabase dashboard, and the credentials file on your machine, in your editor.
    

From this moment until 72 hours after cutover, every Stripe event goes to both backends. The table you added in prompt 2 makes the duplicate a no-op. Payments and renewals never pause during the migration, which is the entire point of doing it this way.

## Prompts 6 and 7: the cutover

Pick a quiet hour. With everything above done, your app is unavailable for minutes, not hours.

**Paste this into your AI coding tool**

```plaintext
We are doing the cutover now, Phase 8 of MIGRATION.md. Follow its order
exactly and do not skip ahead.

1. Set the freeze timestamp in the credentials file to right now, before I
   stop writes, and tell me when it is set.
2. I will tell you when the old app is frozen. Then run the delta data sync
   and the storage delta, and confirm the row counts still match.
3. PAUSE and tell me to flip the app in Lovable. Wait for me to confirm it is
   deployed before doing anything else.
4. Then replay the Stripe events since the freeze, dry run first so I can see
   what it would do, and run the reconcile step after.

After each stage, tell me in one line what ran and whether it worked.
```

When it pauses at step 3, put your app into maintenance mode or unpublish it, then send this.

**Paste this into Lovable chat** (flip the app to your own backend)

```plaintext
Cutover step: this app must stop using Lovable Cloud and use my own Supabase
project instead, starting now.

New project URL: <PASTE YOUR NEW PROJECT URL HERE>
New publishable (anon) key: <PASTE YOUR NEW PUBLISHABLE KEY HERE>
(both values are public by design; no secret keys are included and none are
needed for this step)

1. Point every Supabase client in this app at the project above, including
   src/integrations/supabase/client.ts and any env files or config the client
   reads. If switching the backend needs a settings action instead of a code
   change, tell me exactly where to click and stop.
2. Search the whole repo for the old project ref
   <PASTE THE OLD PROJECT REF HERE> and update every remaining reference
   (hardcoded URLs, env files, config). Do not change database schema or edge
   function code.
3. Publish/deploy the app.

Reply with ONLY a fenced text block containing:
STATUS: done | blocked
DEPLOYED_URL: <live app URL>
CLIENT_FILES_CHANGED: <paths>
OLD_REFS_REMAINING: none | <paths>
NOTES: <one line, or "-">
```

Paste that reply back into your coding tool, take the app out of maintenance mode, and let it finish the replay and reconcile steps. The replay re-sends every Stripe event since your freeze to the new endpoint, properly signed, and anything already handled is skipped.

If something goes badly wrong here, rollback is cheap for as long as the old endpoint and the old project are still alive: send the flip prompt again with your old project URL and key, unpublish and republish, and Stripe needs nothing because the old endpoint never stopped receiving events. Decide fast, though, because anything written to the new backend after the flip stays there.

## Prompts 8 and 9: verify, then fix

**Paste this into your AI coding tool**

```plaintext
Run the Phase 9 verification checklist from MIGRATION.md against my new
project and report each check as pass or fail in a simple list, in plain
English. For anything you cannot check yourself, tell me exactly what to
click in my app to check it. Do not call the migration done until every
check passes.
```

The checks that matter, in the language of someone using the app rather than reading a database:

*   Every table has the same number of rows as before
    
*   An existing user can sign in with the password they already had
    
*   A signed-in user sees their own data and nobody else's
    
*   An image or file loads from the new storage
    
*   A test purchase records exactly 1 payment event, and replaying it records 0 more
    
*   A live subscription shows the same status in your database as in Stripe
    
*   Your scheduled jobs have run recently and succeeded
    
*   No file in your project still mentions the old backend
    

When something fails, do not describe it to Lovable. Copy the exact output and send it.

**Paste this into Lovable chat** (fix whatever failed)

```plaintext
Post-migration verification on my own Supabase project found these failures:

<PASTE THE FAILING CHECKS AND EXACT ERRORS OR OUTPUT HERE>

Fix the causes in this app without changing intended behaviour. The database
now lives in the Supabase project this app was repointed to; do not reference
the old Lovable Cloud project anywhere.

Reply with ONLY a fenced text block containing:
STATUS: done | blocked
CHANGES: <files/migrations changed>
NOTES: <one line, or "-">
```

Then re-run prompt 8 until everything passes.

Afterwards: watch both Stripe endpoints for 72 hours, then delete the old one. Keep the old Lovable project paused rather than deleted for a week, because it is your safety net. Ask your coding tool to delete the dumps and the credentials file from your machine, since those dumps contain real user data.

## The 4 things that surprise people

**Everyone gets signed out once.** A new project signs its tokens with new keys, so every user logs in again the first time after the move. Their passwords and social logins still work. Put a line in your app saying so, or support tickets will tell you about it instead.

**File links saved in your database still point at the old backend.** If any column holds a full image URL rather than a file path, those URLs contain the old project address. They need rewriting on the new project, and rewriting a second time after the cutover copy, because that copy reloads the old values. Ask your tool to check for this in prompt 5.

**Lovable-managed Stripe does not transfer.** If your payments run through Lovable's built-in integration rather than your own Stripe account, the merchant account and your product catalogue stay with Lovable. You connect your own Stripe account, recreate the products and prices, and move existing subscriptions through Stripe's own account-to-account process, which is a support request to Stripe. Find this out before you start, not during the cutover.

**Stripe only remembers 30 days of events.** The replay reaches back about a month. Reconciliation does not depend on events at all, so subscription state is still recoverable, but a freeze older than that cannot be replayed.

## Turn the same app into an iOS and Android app

Owning the backend is half of it. The other half is that the thing your users open is still a browser tab.

Despia takes the web app you already built and ships it as a real native app for iOS and Android. Your Lovable app stays the source of truth. The web code runs in the platform WebView, WKWebView on iOS and the Chromium-based WebView on Android, and 50+ native device capabilities become available through a single JavaScript function inside that same codebase.

There is no second project to keep in sync. The environment check is 1 line, and native calls sit behind it:

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

if (isDespia) despia('successhaptic://')
```

The same pattern covers push notifications, biometric sign-in, the camera, location, native storage and the rest of the feature set. In a browser the check is false and your app behaves exactly as it does today, so you are not maintaining 2 versions of anything. If you would rather not write that yourself, it is a prompt for Lovable like any other.

3 practical consequences for a Lovable app:

**Publishing does not need a Mac.** Builds and code signing run in the cloud, from your browser. No Xcode, no Android Studio, no toolchain to install, no build machine to maintain. If you ever want the native projects, they export in full at any time, so this is not a one-way door.

**Web updates do not need a review.** Changes to your web content ship over the air through remote hydration, without an App Store resubmission. You keep the Lovable deploy loop you already have. Only changes to the native container itself need a new build.

**Your backend does not change.** The native app talks to the Supabase project you just migrated to, with the same URL, the same publishable key and the same security policies. Which is exactly why doing the migration first is worth it: the app sitting in the App Store points at infrastructure you own.

## Get it on the stores

Take the app you just moved onto your own Supabase project and ship it to iOS and Android without a terminal and without a Mac. Code signing and submission run from the browser.

[See the setup docs at setup.despia.com](https://setup.despia.com)