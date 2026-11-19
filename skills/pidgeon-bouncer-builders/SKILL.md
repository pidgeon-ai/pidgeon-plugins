---
name: pidgeon-bouncer-builders
description: Use when adding Pidgeon Bouncer to a web app, or when the app should learn which AI agents visit or sign up and who stands behind them (Web Bot Auth, the Signature-Agent header). Covers apps built with Lovable, Bolt and Replit, Next.js, Express and Supabase sign-ups.
---

# Pidgeon Bouncer: check the AI agents that reach this app

An AI agent that signs its requests under Web Bot Auth sends the headers
`Signature`, `Signature-Input` and `Signature-Agent` with each one. Pidgeon
Bouncer checks the signature and answers who stands behind the agent: its
operator, how well Pidgeon verified them, and their reputation. People never
send these headers, so their requests are never sent to Pidgeon.

Every way in needs a Pidgeon API key with `read` access (pidgeon.ai, API
keys), kept as a server-side secret named `PIDGEON_API_KEY`. Never put it in
the code, the repository or a variable the browser can read.

Pick the section for how this app runs, and follow it whole.

## An app built with Lovable

Add Pidgeon Bouncer to this app's sign-up, so it learns which AI agents sign up and who stands behind them, and turns away agents whose operator is flagged. People who sign up are never sent to Pidgeon and see no change.

An AI agent that signs its requests (Web Bot Auth) sends the headers Signature, Signature-Input and Signature-Agent with each one. The check runs in two Supabase Edge Functions the sign-up form calls.

If this app runs its own server code (server routes under src/routes), make the two functions server routes of the app instead, at /api/public/pidgeon-verdict and /api/public/pidgeon-signup, with the same code: use the server's Supabase admin client, call them with fetch from the sign-up form, and skip the config.toml step. A route reads request.url for the address the agent called, in place of SUPABASE_URL.

1. Ask me for a Pidgeon API key with read access (pidgeon.ai, API keys) and store it as the secret PIDGEON_API_KEY in this project’s Secrets (More, Settings, Secrets), or ask me for it in the secure input. Only the two Edge Functions below read it. Never put it in the code, in .env or in a VITE_ variable.

2. Add this table as a migration:

   create table public.pidgeon_signups (
     user_id uuid primary key references auth.users (id) on delete cascade,
     verification_id text not null,
     result text not null check (result in ('verified', 'signed_known', 'signed_unknown')),
     agent_name text,
     operator_name text,
     operator_verification text,
     band text,
     signature_agent jsonb,
     created_at timestamptz not null default now()
   );
   alter table public.pidgeon_signups enable row level security;
   create policy "Users read their own Pidgeon sign-up" on public.pidgeon_signups
     for select to authenticated using ((select auth.uid()) = user_id);

3. Add the Edge Function pidgeon-verdict. It asks Pidgeon about its own request, but only when that request carries an agent's signature headers: people's requests are never sent. Nothing in the page sets those headers: an agent's browser adds them to every request it sends, this one included.

   const cors = {
     'access-control-allow-origin': '*',
     'access-control-allow-headers': 'authorization, x-client-info, apikey, content-type',
     'access-control-allow-methods': 'POST, OPTIONS',
   };
   const json = (body: unknown, status = 200) =>
     new Response(JSON.stringify(body), { status, headers: { ...cors, 'content-type': 'application/json' } });
   const NOT_SENT = new Set(['cookie', 'authorization', 'proxy-authorization', 'apikey']);

   Deno.serve(async (req) => {
     if (req.method === 'OPTIONS') return new Response(null, { headers: cors });
     if (!req.headers.has('signature') && !req.headers.has('signature-agent')) return json({ verdict: null });
     const headers: Record<string, string> = {};
     req.headers.forEach((value, name) => {
       if (!NOT_SENT.has(name)) headers[name] = value;
     });
     // The address the agent called: inside the function, req.url is another.
     const url = `${Deno.env.get('SUPABASE_URL')}/functions/v1/pidgeon-verdict${new URL(req.url).search}`;
     try {
       const response = await fetch('https://pidgeon.ai/v1/verify', {
         method: 'POST',
         headers: { authorization: `Bearer ${Deno.env.get('PIDGEON_API_KEY')}`, 'content-type': 'application/json' },
         body: JSON.stringify({ request: { method: req.method, url, headers } }),
         signal: AbortSignal.timeout(3000),
       });
       if (!response.ok) console.error('Pidgeon Bouncer: Pidgeon answered', response.status);
       const answer = response.ok ? await response.json() : {};
       return json({ verdict: typeof answer.verdict === 'string' ? answer.verdict : null });
     } catch {
       return json({ verdict: null }); // Pidgeon unreachable: the sign-up goes on as a person's would
     }
   });

4. Add the Edge Function pidgeon-signup. It checks the verdict the new user signed up with, once, and keeps who stands behind the agent, or deletes the user when the verdict does not hold or the operator is flagged.

   import { createClient } from 'npm:@supabase/supabase-js@2';

   (the same cors and json as in pidgeon-verdict)
   const admin = createClient(Deno.env.get('SUPABASE_URL')!, Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!);

   Deno.serve(async (req) => {
     if (req.method === 'OPTIONS') return new Response(null, { headers: cors });
     const { user_id } = await req.json().catch(() => ({}));
     if (typeof user_id !== 'string' || !/^[0-9a-f-]{36}$/i.test(user_id)) return json({ error: 'user_id is not a user id' }, 400);
     const { data } = await admin.auth.admin.getUserById(user_id);
     const user = data?.user;
     const verdict = user?.user_metadata?.pidgeon_verdict;
     if (!user || typeof verdict !== 'string') return json({ result: null }); // a person's sign-up
     if (Date.now() - Date.parse(user.created_at) > 10 * 60 * 1000) return json({ result: null });

     let response: Response;
     try {
       response = await fetch('https://pidgeon.ai/v1/verify/verdict', {
         method: 'POST',
         headers: { authorization: `Bearer ${Deno.env.get('PIDGEON_API_KEY')}`, 'content-type': 'application/json' },
         body: JSON.stringify({ verdict, origin: new URL(Deno.env.get('SUPABASE_URL')!).origin }),
         signal: AbortSignal.timeout(3000),
       });
     } catch {
       return json({ result: 'unchecked' }); // Pidgeon unreachable: keep the user, as a person's sign-up
     }
     if (!response.ok) {
       console.error('Pidgeon Bouncer: Pidgeon answered', response.status); // 401, 402, 403: the key
       return json({ result: 'unchecked' });
     }
     const check = await response.json();
     if (!check.valid) {
       if (check.reason === 'used') {
         const { data: row } = await admin.from('pidgeon_signups').select('user_id').eq('user_id', user_id).maybeSingle();
         if (row) return json({ result: 'recorded' }); // checked before, for this user
       }
       await admin.auth.admin.deleteUser(user_id);
       return json({ error: `This agent can't sign up: its Pidgeon verdict did not hold (${check.reason}).` }, 403);
     }
     const v = check.verdict;
     if (v.band === 'flagged') {
       await admin.auth.admin.deleteUser(user_id);
       return json({ error: `${v.operator?.name ?? 'This agent’s operator'} is flagged on Pidgeon.` }, 403);
     }
     const { error } = await admin.from('pidgeon_signups').upsert({
       user_id,
       verification_id: v.jti,
       result: v.result,
       agent_name: v.agent?.name ?? null,
       operator_name: v.operator?.name ?? null,
       operator_verification: v.operator?.verification ?? null,
       band: v.band ?? null,
       signature_agent: v.signature_agent ?? null,
     });
     if (error) console.error('Pidgeon Bouncer: the sign-up was not recorded', error.message);
     return json({ result: v.result, operator: v.operator?.name ?? null });
   });

   Both functions are called before the user has a session, with the publishable key, which is not a JWT: turn Supabase's JWT check off for them (verify_jwt = false under [functions.pidgeon-verdict] and [functions.pidgeon-signup] in supabase/config.toml). pidgeon-signup checks for itself: it acts only on a user made in the last ten minutes who signed up with a verdict, and deletes one only when Pidgeon refuses that verdict.

5. In the sign-up form, around the existing supabase.auth.signUp call (keep its other options):

   let verdict: string | null = null;
   try {
     const { data: pidgeon } = await supabase.functions.invoke('pidgeon-verdict', { method: 'POST' });
     if (typeof pidgeon?.verdict === 'string') verdict = pidgeon.verdict;
   } catch {
     // Never block a person on it: sign up without a verdict.
   }
   const { data, error } = await supabase.auth.signUp({
     email,
     password,
     // Keep the options it already passes (emailRedirectTo, other data) and add pidgeon_verdict beside them.
     options: { emailRedirectTo, data: { pidgeon_verdict: verdict } },
   });
   if (!error && data.user && verdict) {
     const { error: refused } = await supabase.functions.invoke('pidgeon-signup', { body: { user_id: data.user.id } });
     if (refused?.context instanceof Response && refused.context.status === 403) {
       const { error: why } = await refused.context.json().catch(() => ({ error: 'This agent can’t sign up.' }));
       // Show why as the sign-up's error: the account was not kept.
     }
   }

6. Check it: a sign-up from a normal browser gets { "verdict": null } from pidgeon-verdict without any call to Pidgeon, and never calls pidgeon-signup. An agent's sign-up leaves a row in pidgeon_signups naming its operator.

Trust pidgeon_signups, never user_metadata.pidgeon_verdict: a user can change their own metadata.

Docs: https://pidgeon.ai/docs/bouncer#lovable

## An app built with Bolt

Add Pidgeon Bouncer to this app, so it learns which AI agents visit and sign up, and who stands behind them. People's requests are never sent to Pidgeon and see no change.

An AI agent that signs its requests (Web Bot Auth) sends the headers Signature, Signature-Input and Signature-Agent with each one. Where the check goes depends on how this app runs. Look at package.json:

A. It uses Next.js: check every agent request in the proxy.
   1. Install Pidgeon's SDK: npm install @pidgeon-ai/sdk
   2. Ask me for a Pidgeon API key with read access (pidgeon.ai, API keys) and store it as the server-side environment variable PIDGEON_API_KEY where Bolt keeps this app's secrets, never in the code, the repository or a NEXT_PUBLIC_ variable.
   3. Add proxy.ts at the root (src/proxy.ts if the app uses src/). Before Next.js 16 the file is middleware.ts and the function is named middleware, not proxy. If the app already has one, call observe inside it.

      import { NextResponse, type NextFetchEvent, type NextRequest } from 'next/server';
      import { bouncer } from '@pidgeon-ai/sdk/next';

      // Without a key it does nothing, so the preview is untouched.
      const observe = bouncer({ apiKey: process.env.PIDGEON_API_KEY });

      export function proxy(request: NextRequest, event: NextFetchEvent) {
        observe(request, event); // checks agents after the response; never slows, blocks or throws
        return NextResponse.next();
      }

   4. Check it: a normal page visit makes no call to Pidgeon. The checks show on pidgeon.ai, Bouncer.

B. It is a Vite app whose sign-up uses Bolt Database or Supabase: check agents as they sign up, in two server functions (Supabase Edge Functions) the sign-up form calls.

   1. Ask me for a Pidgeon API key with read access (pidgeon.ai, API keys) and store it as the secret PIDGEON_API_KEY in Bolt’s Secrets (the database icon, Secrets). Only the two Edge Functions below read it. Never put it in the code, in .env or in a VITE_ variable.

   2. Add this table as a migration:

      create table public.pidgeon_signups (
        user_id uuid primary key references auth.users (id) on delete cascade,
        verification_id text not null,
        result text not null check (result in ('verified', 'signed_known', 'signed_unknown')),
        agent_name text,
        operator_name text,
        operator_verification text,
        band text,
        signature_agent jsonb,
        created_at timestamptz not null default now()
      );
      alter table public.pidgeon_signups enable row level security;
      create policy "Users read their own Pidgeon sign-up" on public.pidgeon_signups
        for select to authenticated using ((select auth.uid()) = user_id);

   3. Add the Edge Function pidgeon-verdict. It asks Pidgeon about its own request, but only when that request carries an agent's signature headers: people's requests are never sent. Nothing in the page sets those headers: an agent's browser adds them to every request it sends, this one included.

      const cors = {
        'access-control-allow-origin': '*',
        'access-control-allow-headers': 'authorization, x-client-info, apikey, content-type',
        'access-control-allow-methods': 'POST, OPTIONS',
      };
      const json = (body: unknown, status = 200) =>
        new Response(JSON.stringify(body), { status, headers: { ...cors, 'content-type': 'application/json' } });
      const NOT_SENT = new Set(['cookie', 'authorization', 'proxy-authorization', 'apikey']);

      Deno.serve(async (req) => {
        if (req.method === 'OPTIONS') return new Response(null, { headers: cors });
        if (!req.headers.has('signature') && !req.headers.has('signature-agent')) return json({ verdict: null });
        const headers: Record<string, string> = {};
        req.headers.forEach((value, name) => {
          if (!NOT_SENT.has(name)) headers[name] = value;
        });
        // The address the agent called: inside the function, req.url is another.
        const url = `${Deno.env.get('SUPABASE_URL')}/functions/v1/pidgeon-verdict${new URL(req.url).search}`;
        try {
          const response = await fetch('https://pidgeon.ai/v1/verify', {
            method: 'POST',
            headers: { authorization: `Bearer ${Deno.env.get('PIDGEON_API_KEY')}`, 'content-type': 'application/json' },
            body: JSON.stringify({ request: { method: req.method, url, headers } }),
            signal: AbortSignal.timeout(3000),
          });
          if (!response.ok) console.error('Pidgeon Bouncer: Pidgeon answered', response.status);
          const answer = response.ok ? await response.json() : {};
          return json({ verdict: typeof answer.verdict === 'string' ? answer.verdict : null });
        } catch {
          return json({ verdict: null }); // Pidgeon unreachable: the sign-up goes on as a person's would
        }
      });

   4. Add the Edge Function pidgeon-signup. It checks the verdict the new user signed up with, once, and keeps who stands behind the agent, or deletes the user when the verdict does not hold or the operator is flagged.

      import { createClient } from 'npm:@supabase/supabase-js@2';

      (the same cors and json as in pidgeon-verdict)
      const admin = createClient(Deno.env.get('SUPABASE_URL')!, Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!);

      Deno.serve(async (req) => {
        if (req.method === 'OPTIONS') return new Response(null, { headers: cors });
        const { user_id } = await req.json().catch(() => ({}));
        if (typeof user_id !== 'string' || !/^[0-9a-f-]{36}$/i.test(user_id)) return json({ error: 'user_id is not a user id' }, 400);
        const { data } = await admin.auth.admin.getUserById(user_id);
        const user = data?.user;
        const verdict = user?.user_metadata?.pidgeon_verdict;
        if (!user || typeof verdict !== 'string') return json({ result: null }); // a person's sign-up
        if (Date.now() - Date.parse(user.created_at) > 10 * 60 * 1000) return json({ result: null });

        let response: Response;
        try {
          response = await fetch('https://pidgeon.ai/v1/verify/verdict', {
            method: 'POST',
            headers: { authorization: `Bearer ${Deno.env.get('PIDGEON_API_KEY')}`, 'content-type': 'application/json' },
            body: JSON.stringify({ verdict, origin: new URL(Deno.env.get('SUPABASE_URL')!).origin }),
            signal: AbortSignal.timeout(3000),
          });
        } catch {
          return json({ result: 'unchecked' }); // Pidgeon unreachable: keep the user, as a person's sign-up
        }
        if (!response.ok) {
          console.error('Pidgeon Bouncer: Pidgeon answered', response.status); // 401, 402, 403: the key
          return json({ result: 'unchecked' });
        }
        const check = await response.json();
        if (!check.valid) {
          if (check.reason === 'used') {
            const { data: row } = await admin.from('pidgeon_signups').select('user_id').eq('user_id', user_id).maybeSingle();
            if (row) return json({ result: 'recorded' }); // checked before, for this user
          }
          await admin.auth.admin.deleteUser(user_id);
          return json({ error: `This agent can't sign up: its Pidgeon verdict did not hold (${check.reason}).` }, 403);
        }
        const v = check.verdict;
        if (v.band === 'flagged') {
          await admin.auth.admin.deleteUser(user_id);
          return json({ error: `${v.operator?.name ?? 'This agent’s operator'} is flagged on Pidgeon.` }, 403);
        }
        const { error } = await admin.from('pidgeon_signups').upsert({
          user_id,
          verification_id: v.jti,
          result: v.result,
          agent_name: v.agent?.name ?? null,
          operator_name: v.operator?.name ?? null,
          operator_verification: v.operator?.verification ?? null,
          band: v.band ?? null,
          signature_agent: v.signature_agent ?? null,
        });
        if (error) console.error('Pidgeon Bouncer: the sign-up was not recorded', error.message);
        return json({ result: v.result, operator: v.operator?.name ?? null });
      });

      Both functions are called before the user has a session, with the publishable key, which is not a JWT: turn Supabase's JWT check off for them (verify_jwt = false under [functions.pidgeon-verdict] and [functions.pidgeon-signup] in supabase/config.toml). pidgeon-signup checks for itself: it acts only on a user made in the last ten minutes who signed up with a verdict, and deletes one only when Pidgeon refuses that verdict.

   5. In the sign-up form, around the existing supabase.auth.signUp call (keep its other options):

      let verdict: string | null = null;
      try {
        const { data: pidgeon } = await supabase.functions.invoke('pidgeon-verdict', { method: 'POST' });
        if (typeof pidgeon?.verdict === 'string') verdict = pidgeon.verdict;
      } catch {
        // Never block a person on it: sign up without a verdict.
      }
      const { data, error } = await supabase.auth.signUp({
        email,
        password,
        // Keep the options it already passes (emailRedirectTo, other data) and add pidgeon_verdict beside them.
        options: { emailRedirectTo, data: { pidgeon_verdict: verdict } },
      });
      if (!error && data.user && verdict) {
        const { error: refused } = await supabase.functions.invoke('pidgeon-signup', { body: { user_id: data.user.id } });
        if (refused?.context instanceof Response && refused.context.status === 403) {
          const { error: why } = await refused.context.json().catch(() => ({ error: 'This agent can’t sign up.' }));
          // Show why as the sign-up's error: the account was not kept.
        }
      }

   6. Check it: a sign-up from a normal browser gets { "verdict": null } from pidgeon-verdict without any call to Pidgeon, and never calls pidgeon-signup. An agent's sign-up leaves a row in pidgeon_signups naming its operator.

   Trust pidgeon_signups, never user_metadata.pidgeon_verdict: a user can change their own metadata.

C. Neither: tell me there is no server here to check agents on, and stop.

Docs: https://pidgeon.ai/docs/bouncer#bolt

## An app built with Replit

Add Pidgeon Bouncer to this app's server, so it learns which AI agents call it and who stands behind them, and turns away agents with a bad signature or a flagged operator where they act. People's requests are never sent to Pidgeon and see no change.

An AI agent that signs its requests (Web Bot Auth) sends the headers Signature, Signature-Input and Signature-Agent with each one.

1. Install Pidgeon's SDK: npm install @pidgeon-ai/sdk (for a Python server, pip install pidgeon).
2. Ask me for a Pidgeon API key with read access (pidgeon.ai, API keys) and store it in Secrets as PIDGEON_API_KEY, for the workspace and the published app. Never put it in the code or the repository.
3. Add this middleware (server/pidgeon.ts, or next to the server's other middleware):

   import type { NextFunction, Request, Response } from 'express';
   import { Pidgeon, PidgeonError, type VerifyAnswer } from '@pidgeon-ai/sdk';

   declare global {
     namespace Express {
       interface Request {
         pidgeon?: VerifyAnswer | null;
       }
     }
   }

   const pidgeon = process.env.PIDGEON_API_KEY ? new Pidgeon({ apiKey: process.env.PIDGEON_API_KEY }) : null;
   const NOT_SENT = new Set(['cookie', 'authorization', 'proxy-authorization']);

   /** Asks Pidgeon about each request an agent signed, once. People's requests are never sent. */
   export async function pidgeonBouncer(req: Request, _res: Response, next: NextFunction) {
     req.pidgeon = null;
     const agent = req.headers['signature'] || req.headers['signature-agent'];
     if (!pidgeon || !agent || req.method === 'OPTIONS') return next();
     // The address the agent called: the public host, as Replit's proxy passes it on.
     const host = req.get('x-forwarded-host') ?? req.get('host');
     const headers: Record<string, string> = {};
     for (const [name, value] of Object.entries(req.headers)) {
       if (value !== undefined && !NOT_SENT.has(name)) headers[name] = Array.isArray(value) ? value.join(', ') : value;
     }
     try {
       req.pidgeon = await pidgeon.verify({ method: req.method, url: `https://${host}${req.originalUrl}`, headers });
     } catch (error) {
       // 401, 402, 403: this server's key, its access or billing. Pidgeon down: carry on.
       if (error instanceof PidgeonError && [401, 402, 403].includes(error.status)) {
         console.error('Pidgeon Bouncer is misconfigured:', error.code);
       }
     }
     next();
   }

   /** Where an agent acts: turns away a bad signature and a flagged operator. */
   export function onlyTrustedAgents(req: Request, res: Response, next: NextFunction) {
     const answer = req.pidgeon;
     if (answer?.result === 'refused') return res.status(403).json({ error: `Agent refused: ${answer.reason}` });
     if (answer?.valid && answer.reputation.band === 'flagged') return res.status(403).json({ error: 'Agent operator flagged' });
     next();
   }

4. Register pidgeonBouncer with app.use before the routes, and put onlyTrustedAgents on the routes where an agent acts: sign-up, sign-in, checkout, forms and the API. Leave pages and assets open: crawlers that don't sign yet must get what they get today.
5. Where it helps, read req.pidgeon in a route: answer.valid is true for an agent Pidgeon verified, with answer.operator (name, verification: none, domain, kyc or kyb) and answer.reputation.band (unverified, new, good or flagged). answer.result signed_known or signed_unknown is an agent outside Pidgeon whose signature holds; answer.signature_agent names its directory.
6. For a Python server, do the same in a before-request hook: pidgeon.Pidgeon(api_key=os.environ["PIDGEON_API_KEY"]).verify(method, url, headers), with the same url and headers.
7. Check it: a normal page visit makes no call to Pidgeon, and an agent's request does once.

Docs: https://pidgeon.ai/docs/bouncer#replit

## Any other server

Add Pidgeon Bouncer to this server, so it learns which AI agents are calling it and who stands behind them.

1. Install Pidgeon's SDK: `npm install @pidgeon-ai/sdk` for TypeScript, or `pip install pidgeon` for Python.
2. Read the API key from the environment variable PIDGEON_API_KEY. Never put it in the code or the repository.
3. In a middleware that runs before the routes that matter (sign-ups, checkout, forms, APIs), when a request carries a Signature-Agent header, ask Pidgeon about it before handling it:
     import { Pidgeon } from '@pidgeon-ai/sdk';
     const pidgeon = new Pidgeon({ apiKey: process.env.PIDGEON_API_KEY });
     const answer = await pidgeon.verify(request);
   The url must be absolute, as the agent called it (https://your.host/path?query): Node's req.url is only the path, so build it from the public host. Header values that are arrays are joined with ", ".
   Without the SDK: POST https://pidgeon.ai/v1/verify with "Authorization: Bearer $PIDGEON_API_KEY" and the body
     {"request": {"method": "...", "url": "...", "headers": {"signature": "...", "signature-input": "...", "signature-agent": "..."}}}
   Verify each request once: a signature with a nonce answers valid: false, reason replayed the second time.
4. Decide with the answer:
   - answer.valid is true: answer.operator says who stands behind the agent (name, verification: none, domain, kyc or kyb) and answer.reputation.band their reputation (unverified, new, good or flagged). Let a good reputation through; treat flagged as you would a bad actor. A suspended operator's agents answer valid: false, reason revoked.
   - answer.valid is false: the signature didn't check out (answer.reason). Treat it as an anonymous bot.
   - Requests without a Signature-Agent header are not agents Pidgeon knows: leave them to your existing rules.
5. If Pidgeon can't be reached (a network error, a 5xx or a 429), let the request through to your existing rules rather than failing it. A 401, 402 or 403 is this server's setup (the key, its access, billing), not an outage: never let agents through on it; log it loudly and treat the agent as unverified until it is fixed.

Docs: https://pidgeon.ai/docs

All of it, as Markdown: https://pidgeon.ai/docs/bouncer.md
