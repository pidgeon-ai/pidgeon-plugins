# Pidgeon Bouncer on any server

For a server that is not Next.js. It is the same prompt as <https://pidgeon.ai/docs/bouncer#prompt>:
follow it as written.

```text
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
```
