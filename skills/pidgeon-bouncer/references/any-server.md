# Pidgeon Bouncer on any server

For a server that is not Next.js. It is the same prompt as <https://pidgeon.ai/docs/bouncer#prompt>:
follow it as written.

```text
Add Pidgeon Bouncer to this server, so it sees which AI agents are calling it and who runs them, and acts on the site rules set in Pidgeon.

1. Install Pidgeon's SDK: `npm install @pidgeon-ai/sdk` for TypeScript, or `pip install pidgeon` for Python.
2. Read the API key from the environment variable PIDGEON_API_KEY. Never put it in the code or the repository.
3. In a middleware that runs before the routes that matter (sign-ups, checkout, forms, APIs), when a request carries a Signature-Agent header, or its User-Agent names a known agent (declaredAgentToken from the SDK returns its name), ask Pidgeon about it before handling it:
     import { Pidgeon } from '@pidgeon-ai/sdk';
     const pidgeon = new Pidgeon({ apiKey: process.env.PIDGEON_API_KEY });
     const answer = await pidgeon.verify(request, { place: 'signup' });
   place says where the request is: signup, checkout, api or other. Python: pidgeon.verify(method, url, headers, place="signup").
   The url must be absolute, as the agent called it (https://your.host/path?query): Node's req.url is only the path, so build it from the public host. Header values that are arrays are joined with ", ".
   Without the SDK: POST https://pidgeon.ai/v1/verify with "Authorization: Bearer $PIDGEON_API_KEY" and the body
     {"request": {"method": "...", "url": "...", "headers": {"signature": "...", "signature-input": "...", "signature-agent": "..."}}, "place": "signup"}
   Verify each request once: a signature with a nonce answers valid: false, reason replayed the second time.
4. Act on answer.decision, which the site rule for that place decided (the site owner sets the rules in Pidgeon, Bouncer → Rules):
   - allow: let it in, and skip the site's own friction (a CAPTCHA, a review).
   - block: turn it away. answer.rule says why.
   - limit: leave it to your existing rules (Pidgeon couldn't confirm who it is, or the place is a public page, which never blocks).
   - watch: the place only watches (answer.would says what it would have decided). Let it through.
   - Requests that neither sign nor name a known agent are left to your existing rules, and are never sent to Pidgeon.
5. If Pidgeon can't be reached (a network error, a 5xx or a 429), let the request through to your existing rules rather than failing it. A 401, 402 or 403 is this server's setup (the key, its access, billing), not an outage: never let agents through on it; log it loudly and treat the agent as unverified until it is fixed.

Docs: https://pidgeon.ai/docs
```
