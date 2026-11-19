---
name: pidgeon-bouncer
description: Add Pidgeon Bouncer to a Next.js app, so it learns which AI agents call it and who stands behind them — bouncer() from @pidgeon-ai/sdk/next in proxy.ts (middleware.ts before Next.js 16), with the PIDGEON_API_KEY that the Pidgeon Vercel integration sets. Use when the user asks to add Pidgeon or Pidgeon Bouncer, to see or verify the AI agents or bots visiting the site, to check Web Bot Auth signatures, or when PIDGEON_API_KEY is in the project's environment.
license: MIT
metadata:
  docs: https://pidgeon.ai/docs/bouncer
---

# Pidgeon Bouncer for Next.js

Pidgeon Bouncer checks the signature an AI agent puts on its requests (Web
Bot Auth) and names the operator behind it: the person or company that runs
the agent, how well Pidgeon verified them, and their reputation. In a
Next.js app it runs in the proxy and records every agent's request with
Pidgeon **after** the response: a visit is never slowed, blocked or changed,
even when Pidgeon is down. The checks show up in the site owner's Pidgeon
dashboard.

## The key

The code reads `PIDGEON_API_KEY` from the environment. Installing Pidgeon
Bouncer from the Vercel Marketplace (<https://vercel.com/marketplace/pidgeon>)
sets it on the projects the person picks, for production and preview, as a
sensitive variable. Elsewhere, it is an API key with `read` access from
<https://pidgeon.ai/keys>.

- Never write the key, or a placeholder for it, into the code, a committed
  `.env` file or a prompt. Never ask the person to paste it into the chat.
- Without the key `bouncer()` does nothing. That is expected in local
  development and in previews without the variable: don't add a fallback,
  and don't make the build fail on it.
- A deployment made before the integration was installed has no key: it
  needs a redeploy.

## Steps

1. **Install the SDK** with the project's package manager:
   `npm install @pidgeon-ai/sdk` (or `pnpm add`, `yarn add`, `bun add`).
   Version 2.5.0 or later has `@pidgeon-ai/sdk/next`.
2. **Find the proxy file.** Read the `next` version in `package.json`.
   - Next.js 16 or later: `proxy.ts`, exporting a function named `proxy`
     (or a default export).
   - Before 16: `middleware.ts`, exporting a function named `middleware`.

   It sits beside the `app` (or `pages`) folder: the project root, or
   `src/` when the app lives there. A project that already has a
   `middleware.ts` on Next.js 16 keeps it; add Bouncer there and leave the
   rename to the person.

3. **No proxy file yet:** create it.

   ```ts
   import { NextResponse, type NextFetchEvent, type NextRequest } from 'next/server';
   import { bouncer } from '@pidgeon-ai/sdk/next';

   // A key with read access. Without one it does nothing: local development and previews are untouched.
   const observe = bouncer({ apiKey: process.env.PIDGEON_API_KEY });

   export function proxy(request: NextRequest, event: NextFetchEvent) {
     observe(request, event); // returns at once; never throws
     return NextResponse.next();
   }
   ```

   For `middleware.ts`, name the function `middleware` instead of `proxy`.
   Leave out a `config.matcher`: the helper ignores every request without
   agent headers at the cost of a header check, so it can see the whole
   site.

4. **A proxy file already exists:** keep everything it does, and call
   `observe(request, event)` as the first line of its function, before any
   early return. Make the function take the event as its second parameter
   if it does not.

   If the file exports a wrapper's result (an auth library's middleware,
   `export default someMiddleware(...)`), keep the wrapper and call it from
   a function of your own:

   ```ts
   import type { NextFetchEvent, NextRequest } from 'next/server';
   import { bouncer } from '@pidgeon-ai/sdk/next';

   const observe = bouncer({ apiKey: process.env.PIDGEON_API_KEY });
   const existing = someMiddleware(/* exactly as it was */);

   export default function proxy(request: NextRequest, event: NextFetchEvent) {
     observe(request, event);
     return existing(request, event);
   }
   ```

   Keep the file's `config` export unchanged. Bouncer sees only the paths its
   matcher lets through; say so to the person if the matcher leaves out
   pages agents are likely to visit.

5. **Check it:** the project type-checks and builds. Then tell the person
   that agents' visits appear at <https://pidgeon.ai/bouncer> once the
   deployment carries `PIDGEON_API_KEY`, and that they need the Pidgeon
   integration on the Vercel project (or the key set by hand) for that.

## What not to do

- Don't block, redirect, challenge or rate-limit agents with it. `observe`
  is the only mode today; it records and never acts. Pages meant to be read
  and cited (docs, articles, `llms.txt`) must answer AI crawlers exactly as
  before.
- Don't `await` anything in the proxy for Bouncer, and don't wrap the call
  in `try`: it returns at once and never throws.
- Don't call `/v1/verify` yourself for the same request: each signature is
  checked once, and a second check of a signed nonce answers `replayed`.

## Options

`bouncer({ apiKey, url, onAnswer, onError, timeoutMs })`:

- `url(request)`: the URL the agent called, when the proxy sees another (a
  load balancer that changes the host). Defaults to `request.url`.
- `onAnswer(answer, request)`: Pidgeon's answer, after the response, to log
  or count it.
- `onError(error, request)`: a check that failed. Defaults to `console.warn`.
- `timeoutMs`: how long a check may take, 5,000 by default.

## Deciding at sign-up

`bouncer()` answers after the response, too late to decide on. To decide on
a sign-up's own request (let a good operator's agent through, stop a flagged
one), `verdictFor` from `@pidgeon-ai/sdk/next` waits for the answer and gives
a verdict, or `null`: <https://pidgeon.ai/docs/bouncer#verdict-nextjs>.

## Not Next.js

For any other server, follow `references/any-server.md`.

Docs: <https://pidgeon.ai/docs/bouncer.md>. Questions: support@pidgeon.ai.
