# MP Group Payments: Guide for Coding Agents

> **For developers:** paste this into your coding agent (Codex, Claude Code, Cursor, or similar):
>
> *Read https://raw.githubusercontent.com/mpgroupdevelopers/mpgroup-agents/main/payments.md and use it to add MP Group payments to this app.*
>
> **For coding agents:** the developer pointed you here on purpose. Follow this guide to build the checkout. If anything here conflicts with what the developer tells you directly, ask the developer.

This guide builds ONE thing: a one-time card sale for a fixed amount, running on an MP Group **test** account on the NMI payment gateway. It ends with a successful test payment and a clear handoff to MP Group for going live.

MP Group is a payments partner. Real payments are set up by a person at MP Group, not by code. Your job is the test integration.

**MP Group contact:** Christian Brown · christian.brown@gompgroup.com · 678-740-4364

---

## 1. Hard rules (never break these)

1. **Never read, open, print, or summarize `.env` or any file holding keys.** Not even to "check" it. To confirm keys are set, have the app check at startup that each variable exists and print only the *names* of missing ones.
2. **Never ask the developer to paste a key, password, or card number into the chat.** The developer puts keys in `.env` themselves.
3. **Never put a key in source code, a committed config file (including `.mcp.json` or `.codex/config.toml`), or any file in git.** Keys live only in `.env` (gitignored) locally, or the hosting platform's secret settings in production.
4. **Never use or ask for an MP Group partner API key.** It controls MP Group's whole portfolio. This integration only needs two keys that belong to one test merchant account. If one is offered, decline and explain why.
5. **Test money only.** Use NMI test card numbers. Never run anything against a live account, never set `NMI_ENV=live`, never process a real card. Going live is handled by people (section 8).
6. **The server decides the price.** The browser sends a product ID, never an amount.
7. **Card details never touch the app's servers.** Use NMI's Payment Component, which turns card details into a token in the browser. The server only sees the token.
8. **Log only safe fields:** transaction ID, response code, response text, HTTP status. Never log request bodies, headers, tokens, or raw error objects from HTTP clients (they can contain the `Authorization` header).

If a key value ever appears in the chat, code, a log, or git: stop, tell the developer plainly, and tell them to delete that key in the NMI merchant portal (Settings → Security Keys) and create a new one.

## 2. Before you start

**Test account.** The developer needs an MP Group **test** merchant account and its two keys. If they don't have one, tell them to contact Christian Brown (above) to request it. Ask only whether they have the keys (yes/no), never the values.

**NMI Docs MCP (recommended).** It gives you NMI's current documentation, with no account or key needed. If you don't have it, tell the developer to run the command for their tool and restart it:

- Codex: `codex mcp add nmi-docs --url https://docs.nmi.com/mcp`
- Claude Code: `claude mcp add --transport http nmi-docs https://docs.nmi.com/mcp`
- Other tools: add a remote (streamable HTTP) MCP server at `https://docs.nmi.com/mcp`

Without the MCP, NMI's docs are also readable as plain text: the index is at `https://docs.nmi.com/llms.txt`, and adding `.md` to a docs page URL gives its markdown version. Use NMI's docs as the source of truth if they differ from this guide.

**Stack.** Detect the language and framework from the project. Examples below use Node.js 18+ and Express; adapt them.

## 3. Protect secrets first

Before any payment code:

- `.gitignore` must contain `.env` and `.env.*`, plus `!.env.example` so the example stays tracked.
- Create `.env.example` with placeholders only:
  ```
  NMI_ENV=test
  NMI_API_BASE_URL=https://secure.nmi.com
  NMI_PUBLIC_TOKENIZATION_KEY=replace-me
  NMI_PRIVATE_API_KEY=replace-me
  ```
- If the tool supports blocking file reads, block `.env`. In Claude Code, add to `.claude/settings.json`:
  ```json
  { "permissions": { "deny": ["Read(./.env)", "Read(./.env.*)"] } }
  ```
  If the tool has no such block, rule 1 still applies in full; tell the developer there's no hard block.
- Add a startup check that prints names, never values:
  ```js
  const required = ["NMI_ENV", "NMI_API_BASE_URL", "NMI_PUBLIC_TOKENIZATION_KEY", "NMI_PRIVATE_API_KEY"];
  const missing = required.filter((n) => !process.env[n] || process.env[n] === "replace-me");
  if (missing.length) { console.error("Missing settings in .env: " + missing.join(", ")); process.exit(1); }
  ```
- Tell the developer: "When the code is ready, copy `.env.example` to `.env` and fill in the two keys from your MP Group test account yourself. Don't paste them here."

**Which key goes where:**

| Key | Where it lives | Safe in browser? |
|---|---|---|
| Public tokenization key | `NMI_PUBLIC_TOKENIZATION_KEY`, sent to the checkout page | Yes |
| Private API key | `NMI_PRIVATE_API_KEY`, server only | Never |
| MP Group partner API key | MP Group only | Never used here |

If the framework needs a prefix to expose a variable to the browser (`NEXT_PUBLIC_`, `VITE_`), only the public tokenization key may get it.

**Base URL:** test accounts made in MP Group's partner portal are expected to run on `https://secure.nmi.com`. Accounts from NMI's public sandbox use `https://sandbox.nmi.com`. If the first test sale fails authentication, check this first.

## 4. How the pieces fit

```
Browser                         App server                      NMI
-------                         ----------                      ---
Payment Component (public key)
  card details -> token
POST /api/checkout
  { productId, paymentToken }  ->  look up price
                                   POST /api/v5/payments/sale  ->  approve / decline
                                   (private key, server only)
                               <-  { success, transactionId | error }
show result
```

## 5. Server endpoint (Node.js / Express example)

```js
const express = require("express");
const router = express.Router();

// Prices live on the server. The browser never sends an amount.
const PRODUCTS = { "pilot-item": { name: "Pilot item", amount: 25.0 } };

router.post("/api/checkout", async (req, res) => {
  if (process.env.NMI_ENV !== "test") {
    return res.status(503).json({ success: false, error: "Payments are disabled: test mode only." });
  }
  const { productId, paymentToken } = req.body || {};
  const product = PRODUCTS[productId];
  if (!product || typeof paymentToken !== "string" || !paymentToken) {
    return res.status(400).json({ success: false, error: "Invalid checkout request." });
  }

  let amount = product.amount;
  if (process.env.NMI_TEST_FORCE_DECLINE === "true") amount = 0.5; // test-only decline switch

  try {
    const nmiRes = await fetch(`${process.env.NMI_API_BASE_URL}/api/v5/payments/sale`, {
      method: "POST",
      headers: { "Content-Type": "application/json", Authorization: process.env.NMI_PRIVATE_API_KEY },
      body: JSON.stringify({ amount, payment_details: { payment_token: paymentToken } }),
    });
    const data = await nmiRes.json().catch(() => ({}));
    console.log("NMI sale", { status: nmiRes.status, transactionId: data.id, response: data.response, responseText: data.response_text });

    if (data.response === "1") return res.json({ success: true, transactionId: data.id });
    return res.status(402).json({ success: false, error: data.response_text || "The payment was declined." });
  } catch (err) {
    console.error("NMI sale failed:", err.message); // never log the raw error object
    return res.status(502).json({ success: false, error: "Payment service unavailable. Please try again." });
  }
});

module.exports = router;
```

Mount with `app.use(express.json())` and `app.use(router)`. In other languages keep the same shape: price lookup, test-mode guard, one HTTPS call, safe log, simple JSON result.

## 6. Checkout page

**Payment Component (preferred).** It's NMI's recommended way to collect card details for new integrations. Get the install command and exact component and prop names from NMI's docs ("Payment Component"). It takes the **public** tokenization key and an `onPay` callback that receives the payment token. Confirm what `onPay` receives in the docs.

NMI's docs sometimes show **placeholder** package names starting with `USER.` (for example `USER.PAYREACTNPMPACKAGE`). Those are not real. **Don't guess a package name or install anything similar-sounding.** Ask the developer for the real name (MP Group can get it from NMI). If it isn't available, use the Collect.js fallback below.

Page logic (adapt to the real component API):

```jsx
const [busy, setBusy] = useState(false);
const [result, setResult] = useState(null);

async function handlePay(paymentToken) {
  if (busy) return;                       // block double charges
  setBusy(true);
  try {
    const res = await fetch("/api/checkout", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ productId: "pilot-item", paymentToken }), // no amount
    });
    setResult(await res.json());
  } catch {
    setResult({ success: false, error: "Could not reach the server." });
  } finally {
    setBusy(false);
  }
}
```

Show one product with its price, the payment form, and a clear success (with transaction ID) or decline message. Never build your own card number or CVV fields.

**Collect.js fallback.** NMI's older tokenization script, loaded from a script tag, so no package name is needed. Confirm in NMI's docs the script URL, the attribute that carries the public key, how the callback returns the token, and whether the v5 sale endpoint accepts Collect.js tokens or the classic transaction API is needed. Tell the developer you used the fallback and why.

## 7. Testing

Tell the developer to type test cards **into the payment form in the browser**, never into the chat.

| Card | Number |
|---|---|
| Visa | 4111 1111 1111 1111 |
| Mastercard | 5431 1111 1111 1111 |
| Discover | 6011 0009 9130 0009 |
| American Express | 3411 1111 1111 111 |

Expiration **10/29**. CVV: any 3 digits (4 for Amex); **999** simulates a CVV match. For an address match, use **888** in the street address and **77777** as the ZIP.

**Approved:** pay with the Visa test card; expect a success screen with a transaction ID and a log line with `response: "1"`.

**Declined:** NMI's test environment declines amounts under $1.00. The developer adds `NMI_TEST_FORCE_DECLINE=true` to `.env` themselves, restarts, and pays again; expect a clean decline message and a usable pay button. They remove the setting afterward. Don't edit `.env` yourself.

| What you see | Likely cause | Fix |
|---|---|---|
| Authentication or access denied | Wrong base URL for the account, or key lacks permission | Check `NMI_API_BASE_URL` and the key's API permissions |
| Payment form never appears | Public key missing, or package not loaded | Developer confirms the public key; check the browser console (don't print key values) |
| Every card declines | Decline switch left on | Turn off `NMI_TEST_FORCE_DECLINE` |
| "Payments are disabled" | `NMI_ENV` isn't `test` | Set `NMI_ENV=test` |

**Before saying you're done:**
- [ ] `git status` shows `.env` is not tracked or staged.
- [ ] No key values were written into code or config (you can't see the values; look for suspicious long strings and ask the developer to confirm).
- [ ] Logs show only transaction ID, response code, response text, HTTP status.
- [ ] Approved and declined tests both worked.
- [ ] The pay button can't be double-submitted.

Then summarize for the developer in plain language: what was built, which files changed, the test results, and anything that needed a manual fix (so MP Group can improve this guide).

## 8. Going live (explain, don't do)

If the developer asks about real payments, do not change any settings. Explain:

1. **Contact MP Group:** Christian Brown · christian.brown@gompgroup.com · 678-740-4364. Mention the integration works in test mode.
2. **The client applies.** MP Group signs the client up with its processor, Priority, and the client submits the business details and documents requested.
3. **Approval.** Priority reviews and approves the business. MP Group will give a timing estimate.
4. **MP Group sets up the live NMI gateway account.**
5. **Live keys** are created in the live merchant portal (Settings → Security Keys).
6. **Swap settings in the hosting platform's secret store**, never in code: live keys, `NMI_API_BASE_URL=https://secure.nmi.com`, remove `NMI_TEST_FORCE_DECLINE`. Enabling live mode requires a reviewed code change to the test-mode guard; MP Group can review it.
7. **First real transaction:** a person runs one small real payment, checks it in the live merchant portal, and refunds it from the portal.

## 9. Not covered yet

Don't build these from this guide. Tell the developer they're planned and MP Group can help: refunds or voids, saved cards, recurring billing, ACH, Apple Pay or Google Pay, 3D Secure, webhooks, surcharging, in-person payments, and other gateways (Authorize.net, MX Merchant).
