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

Lovable Cloud is the fastest way to get a working backend behind an app you described in a chat window. It is also a backend you do not hold the keys to. At some point you want the database, the auth users, the storage bucket and the Stripe webhook in a project on your own Supabase account, with your own billing, your own region and a connection string you can point psql at.

This post walks that migration end to end. The end state: the same app, running against your own Supabase project, with every user ID unchanged, every password still working, every file in place, and not one Stripe event lost along the way.

All of it is packaged as an open source agent skill: [github.com/despia-native/lovable-to-supabase](https://github.com/despia-native/lovable-to-supabase). It installs a runbook plus 7 scripts into your repo and drives the whole thing with your agent. You can also just read the runbook and run the commands yourself.

## What you are actually moving

A Lovable Cloud project is Supabase underneath, which is the good news. The pieces that need to move:

| Piece | How it moves |
| --- | --- |
| Schema (tables, enums, functions, triggers, RLS policies) | `pg_dump` of the public schema, plus auth triggers and storage policies |
| Data | Data dump, `auth.users` and `auth.identities` first so user IDs stay stable |
| Auth users | Same dump. Bcrypt hashes travel, so passwords keep working |
| Storage buckets and files | Copied object by object through service-role clients |
| Edge functions | Redeployed from your repo with the Supabase CLI |
| Cron jobs | Regenerated as `cron.schedule` statements with the new URLs |
| Stripe webhooks | Second endpoint runs in parallel, then the old one is retired |

Stable user IDs are the load-bearing detail. Every foreign key in your app points at `auth.users.id`, and every RLS policy compares against `auth.uid()`. Preserve those IDs and the entire app keeps working after the move. Regenerate them and you own a data-repair project instead of a migration.

## 2 lanes, and you are the wire

The reason migrations like this go wrong is that people try to do all of it in one place. Lovable's sandbox cannot reach your new project's Postgres port, and it must never hold a service-role key or a database password. So every step belongs to exactly one of 2 lanes:

**The Lovable lane** is code changes: repointing the client, adding an idempotency table, guarding the webhook handler. You do these by pasting a prompt block into Lovable chat.

**The local lane** is everything that touches real credentials: dumps, restores, the storage copy, function deploys, the Stripe replay. These run on your machine, in Cursor or Codex or Claude Code or a plain terminal, with credentials living in one gitignored file.

Lovable and your local agent cannot talk to each other. You are the wire between them. That is why every cross-lane step in the runbook is a numbered, fenced block that ends with a strict reply contract, so you paste a block one way and paste the answer back the other way without either side improvising.

One rule holds the whole security model together: the only credentials that ever go into a Lovable prompt are the new project URL and the publishable key, because both ship in your client bundle anyway. Service-role keys, database passwords and Stripe secret keys go into `.env.migration` and nowhere else. Not into chat, not into a commit, not into a screenshot.

Every prompt in this post is labelled with the window it goes into. There are only 2:

| Prompt | Paste it into |
| --- | --- |
| 1\. Run the migration | Your AI coding tool (Cursor, Claude Code, Codex) |
| 2\. Idempotent webhooks | Lovable chat |
| 3\. Reconciliation endpoint | Lovable chat |
| 4\. Flip the backend | Lovable chat |
| 5\. Fix verification failures | Lovable chat |

## Install the skill

With the skills CLI, which works for Claude Code, Codex and Cursor:

```bash
npx skills add despia-native/lovable-to-supabase
```

Or manually for Claude Code:

```bash
git clone https://github.com/despia-native/lovable-to-supabase /tmp/lovable-to-supabase
cp -r /tmp/lovable-to-supabase/skills/* ~/.claude/skills/
```

Then open your Lovable-exported repo and say:

```plaintext
migrate this app off Lovable Cloud to my own Supabase project
```

The agent installs `MIGRATION.md` and `scripts/migration/` into the repo, adds the gitignore entries before any secret file exists, and walks the phases with you. If you would rather not use an agent at all, copy the same 2 paths into your repo and follow the runbook top to bottom.

If your tool does not read skills, or you want it pinned to the repo and its rules explicitly, paste the full version instead.

**Prompt 1. Paste this into your AI coding tool running locally** (Cursor, Claude Code, Codex, anything with a shell in your repo)

```plaintext
CONTEXT
This repo is a Lovable export. Its backend is Lovable Cloud (Supabase under
the hood) and I am moving it onto a Supabase project I own. You have a shell
on my machine. Lovable chat is a separate window I control; you and Lovable
cannot talk to each other, so I relay blocks between you.

SOURCE OF TRUTH
Read https://github.com/despia-native/lovable-to-supabase before writing any
code. Install its skills/lovable-to-supabase/MIGRATION.md and
skills/lovable-to-supabase/scripts/migration/ into this repo and follow that
runbook. Use its numbered handoff blocks as written, filling in placeholders.
Do not invent your own wording where a canonical block exists, and do not
improvise a step the runbook does not have. Ask me rather than assuming.

MY SETUP
Source project ref: <OLD_REF>
Target Supabase project ref: <TARGET_REF>
Target region: <same region as source>
Payments: <my own Stripe account | Lovable-managed | none>
Storage buckets: <list, or "check the inventory">

TASK
1. Install the runbook and scripts, chmod +x the shell scripts, and append
   .env.migration and scripts/migration/dumps/ to .gitignore BEFORE creating
   any env file.
2. Create scripts/migration/.env.migration from the example with chmod 600,
   pre-filling only non-secret values you can read from this repo, then stop
   so I can fill the secrets in my editor.
3. Run 00-inventory.sh and summarise it, flagging tables without RLS, cron
   jobs, vault secret names, and edge function env var names.
4. Give me the Phase 2 blocks to paste into Lovable, then wait for my replies.
5. Run Phase 3 through Phase 7 in runbook order, stopping at the first failure.
6. Walk me through the Phase 8 cutover in its exact order, pausing for the
   flip block and my confirmation that Lovable has deployed.
7. Run the Phase 9 verification checklist and report each check pass or fail.

RULES
Never print, cat, or echo .env.migration, dump files, or anything under
scripts/migration/dumps/.
Never ask me to paste a secret into this chat.
Never put anything other than the project URL and publishable key into a
Lovable-bound block.
Never continue past a failed row-count gate, a schema-drift abort, or a
storage copy that reports failures.
Never reorder the cutover sequence, and never run the delta dump before the
freeze timestamp is set.
Never pipe input into the restore script's interactive confirmation.
Never use git add -f on an ignored path.

DONE WHEN
Every table's row count matches the source.
An existing user signs in with their old password.
As a signed-in user with the publishable key, I can read my own rows and not
another user's.
A storage object loads from the new bucket.
A test checkout writes exactly 1 webhook_events row, and re-running the replay
adds 0 more.
git grep for the old project ref returns nothing.
```

The 2 sections that turn this from a wish list into something an agent can execute are MY SETUP and DONE WHEN. Filling in the refs stops it guessing at project identifiers, and acceptance criteria give it a self-check, so it stops when the migration actually works rather than when it has finished writing commands.

The rest of this post is what happens next, in order, and the 4 prompts you paste into Lovable along the way.

## Step 1: look at what you have before you touch anything

Create the target Supabase project in the same region as the source, then fill `scripts/migration/.env.migration` in your editor. Source credentials come from Lovable's Cloud settings, currently under the Cloud tab and its advanced settings, where the direct database connection string lives. That UI moves around. If a value is not exposed, ask Lovable support for it, because it is your data.

Then run the read-only inventory:

```bash
./scripts/migration/00-inventory.sh
```

It prints tables with row estimates, enums, functions, triggers, RLS policies, tables that are missing RLS, extensions, realtime tables, cron jobs, storage buckets, auth user counts, the names of your vault secrets, and every environment variable name your edge functions read. Nothing secret is printed. Keep the output, because 4 of the later phases are checklists against it.

2 things to look for in that output. Tables with no RLS policy are a hole you should close before migrating them, not after. And any cron job whose command has a service-role key baked into it is a leak that should be rewritten to read from vault, then rotated.

## Step 2: wire the app while it is still on Lovable

This is the step people skip, and it is the one that makes the payment story safe.

Before you dump anything, ask Lovable to make 2 changes on the source project: a `webhook_events` table that makes Stripe processing exactly-once, and a protected endpoint that can re-sync subscription truth from Stripe at any time. Paste each block whole, wait for its reply block, and keep the replies.

**Prompt 2. Paste this into Lovable chat** (idempotent webhook processing)

```plaintext
You are working on my app which currently uses Lovable Cloud (Supabase under
the hood). Task: make Stripe webhook processing idempotent so every event is
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
   - No row returned, the event was already handled (or is being handled):
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

**Prompt 3. Paste this into Lovable chat** (protected reconciliation endpoint)

```plaintext
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

Paste each reply back to your local tool. The reply contract exists so neither side has to interpret prose.

Why these changes happen on the source project rather than the target: the schema dump in the next step carries them across for free, and the cutover delta cannot then hit schema drift between the 2 projects. Idempotent webhook processing is also what lets both Stripe endpoints run at the same time later, since a duplicate delivery becomes a no-op instead of a double charge in your own records.

## Step 3: move schema, data and users

3 commands, in this order:

```bash
./scripts/migration/01-dump-schema.sh
./scripts/migration/02-dump-data.sh
./scripts/migration/03-restore.sh all
```

The restore disables triggers and foreign key checks during the load, runs inside a single transaction, then compares row counts table by table against the source.

That comparison is a hard gate. Every row must read `ok` before you go further. A mismatch on one busy table almost always means writes were not fully frozen, usually a cron job or a webhook still pointing at the source. The troubleshooting appendix covers the rest, including the 2 that catch people most often: a `pg_dump` older than the source's Postgres major version, and a network with no IPv6 route to the direct hostname, where the session pooler on port 5432 works and the transaction pooler on port 6543 does not.

## Step 4: storage, functions, secrets and cron

Buckets and objects are not in a database dump:

```bash
bun scripts/migration/04-copy-storage.ts
```

It recreates each bucket with the same public flag, size limit and MIME allow-list, then streams objects across. It is re-runnable and skips anything already present at the same size, so you run the slow bulk copy today and a fast delta at cutover.

Then deploy functions to the target and set every secret name from the inventory:

```bash
supabase link --project-ref <TARGET_REF>
supabase functions deploy
```

Cron jobs get regenerated from the schema dump with the new project URL and the new publishable key substituted in. Auth configuration is the one part that is genuinely manual: site URL, redirect URLs, every OAuth provider with client credentials from your own console accounts, email templates and SMTP. None of it is in any dump.

## Step 5: run both Stripe webhooks at once

Add a second endpoint in the Stripe dashboard pointing at the target project's webhook function, subscribed to the same event set as the existing one, and leave the old endpoint enabled. Copy the new signing secret into the target function's secrets and into `.env.migration`.

From here until 72 hours after cutover, every Stripe event is delivered to both projects. The idempotency table means that costs you nothing. Payments and renewals never pause during the migration, which is the point.

## Step 6: the cutover, in strict order

Pick a low-traffic window. With everything above done, the downtime is a delta dump plus a restore, so minutes.

1.  Record `FREEZE_TIMESTAMP` in `.env.migration`, set to now, before you stop writes. Overlap is safe because of idempotency. A gap is not.
    
2.  Freeze writes on the old app. Reads can continue.
    
3.  Re-run the data dump and `03-restore.sh data` for the delta.
    
4.  Re-run the storage copy for its delta.
    
5.  Flip the app to the new project with the block below.
    
6.  Unfreeze.
    
7.  Replay Stripe events since the freeze with `06-stripe-replay.ts`.
    
8.  Reconcile subscription state from Stripe with `07-stripe-reconcile.ts`.
    
9.  Verify.
    

**Prompt 4. Paste this into Lovable chat** (flip the backend, cutover step 5)

```plaintext
Cutover step: this app must stop using Lovable Cloud and use my own Supabase
project instead, starting now.

New project URL: <PASTE TARGET_SUPABASE_URL HERE>
New publishable (anon) key: <PASTE TARGET PUBLISHABLE KEY HERE>
(both values are public by design; no secret keys are included and none are
needed for this step)

1. Point every Supabase client in this app at the project above - including
   src/integrations/supabase/client.ts and any env files/config the client
   reads. If backend switching needs a settings/integration action instead of
   a code change, tell me exactly where to click and stop.
2. Search the whole repo for the old project ref
   <PASTE OLD PROJECT REF HERE> and update every remaining reference
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

The replay script lists Stripe events since your freeze timestamp and re-delivers them to the new endpoint, re-signed with that endpoint's own signing secret. Signature verification stays fully enabled. Anything the parallel endpoint already processed is skipped by `webhook_events`.

If the cutover goes wrong, rollback is cheap for exactly as long as you leave the old endpoint and the old project alive: repeat the flip block with the old URL and key, unfreeze, and Stripe needs nothing because the old endpoint never stopped receiving events. Keep that decision window short, since writes that landed on the new project after the flip stay there.

## Step 7: verify like you do not trust yourself

Run all of these against the new project before calling it done:

*   Row counts match per table
    
*   An existing user signs in with their old password, and an OAuth user signs in
    
*   RLS holds for a real signed-in user, checked with the publishable key rather than the service-role key, which bypasses RLS and proves nothing
    
*   A storage object loads from the new bucket
    
*   A test checkout writes exactly 1 `webhook_events` row
    
*   Re-running the replay adds 0 rows to that table
    
*   A live subscription's status and period end match the Stripe dashboard
    
*   `cron.job_run_details` shows a recent successful run
    
*   `git grep` for the old project ref comes back clean
    

When one of these fails, route it back with its evidence rather than describing it.

**Prompt 5. Paste this into Lovable chat** (fix verification failures)

```plaintext
Post-migration verification on my own Supabase project found these failures:

<PASTE THE FAILING CHECKS AND EXACT ERRORS/OUTPUT HERE>

Fix the causes in this app without changing intended behaviour. The database
now lives in the Supabase project this app was repointed to; do not reference
the old Lovable Cloud project anywhere.

Reply with ONLY a fenced text block containing:
STATUS: done | blocked
CHANGES: <files/migrations changed>
NOTES: <one line, or "-">
```

Afterwards: watch both Stripe endpoints for 72 hours, then delete the old one. Keep the old project paused rather than deleted for a week. Delete the dumps and the env file from your machine, because those dumps contain real PII and password hashes.

## The 4 things that silently break

**Sessions.** A new project signs tokens with new keys, so every user is signed out once and logs in again. Passwords and OAuth links survive. Say it in a banner rather than letting people think the app broke.

**Absolute storage URLs stored in the database.** Public object URLs contain the project ref, so any `avatar_url` column holding a full URL points at the old host. Rewrite them on the target, and rewrite them again after the cutover delta, because reloading source rows restores the old hostnames.

**Lovable-managed Stripe.** If payments run through Lovable's built-in integration rather than your own Stripe account, the merchant account and the product catalog do not transfer. You connect your own account, recreate products and prices, and move existing subscriptions through Stripe's own account-to-account migration, which is a support request. Find this out in Phase 0, not during the cutover.

**The 30-day replay window.** Stripe's event list reaches back about 30 days. Reconciliation does not depend on events, so it still works, but a freeze window older than that cannot be replayed.

## Turn the same app into an iOS and Android app

Owning the backend is half of it. The other half is that the thing your users open is a browser tab.

Despia takes the web app you already built and ships it as a real native binary for iOS and Android. Your Lovable app stays the source of truth. The web code runs in the platform WebView, WKWebView on iOS and the Chromium-based WebView on Android, and 50+ native device capabilities become available through a single JavaScript function inside that same codebase.

There is no second project to keep in sync. The environment check is a one-liner, and native calls sit behind it:

```javascript
const isDespia = navigator.userAgent.toLowerCase().includes('despia')

if (isDespia) despia('successhaptic://')
```

The same pattern covers push notifications, biometric authentication, the camera, geolocation, native storage and the rest of the feature set. In a browser the check is false and your web app behaves exactly as it does today, so you are not maintaining 2 code paths.

3 practical consequences for a Lovable app:

**Publishing does not need a Mac.** Builds and code signing run in the cloud, from the browser. No Xcode, no Android Studio, no local toolchain to install and no build machine to maintain. If you ever want the native projects, they export in full at any time, so this is not a one-way door.

**Web updates do not need a review.** Changes to your web content ship over the air through remote hydration, without an App Store resubmission. You keep the Lovable deploy loop you already have. Only changes to the native container itself need a new build.

**Your backend does not change.** The native app talks to the Supabase project you just migrated to, with the same URL, the same publishable key and the same RLS policies. That is exactly why doing the migration first is worth it: the app on the store points at infrastructure you own.

## Get it on the stores

Take the app you just moved onto your own Supabase project and ship it to iOS and Android without a CLI or a Mac. Code signing and submission run from the browser.

[See the setup docs at setup.despia.com](https://setup.despia.com)