---
name: mpgroup-nmi-checkout
description: Build a secure card checkout on MP Group's NMI gateway (a one-time card sale using NMI's Payment Component and v5 REST API), in test mode only. Use this skill whenever the user wants to add payments, a checkout, a "pay" button, or card processing with MP Group or NMI, mentions an NMI test account or tokenization key, or asks how to take card payments for a client through MP Group, even if they don't say "skill" or "NMI" explicitly. Also use it when they ask how to go live with MP Group.
---

# MP Group NMI Checkout (pilot)

This skill helps you build ONE thing: a one-time card sale for a fixed amount, running against an MP Group **test** account on the NMI gateway. It ends with a successful test payment and a clear handoff to MP Group for going live.

This skill works in Claude Code and Codex. Where a step differs between them, both versions are given.

MP Group is a payments partner. The developer's client will be approved and set up for real payments by a person at MP Group, not by you. Your job is the test integration.

## Hard rules (never break these)

1. **Never read, open, print, or summarize `.env` or any file holding keys.** Not even to "check" it. To confirm keys are set, have the app check at startup that each variable exists and print only the *names* of missing ones.
2. **Never ask the user to paste a key, password, or card number into the chat.** The user puts keys in `.env` themselves.
3. **Never put a key in source code, a committed config file (including `.mcp.json` or `.codex/config.toml`), or this skill.** Keys live only in `.env` (gitignored) locally, or the hosting platform's secret settings in production.
4. **Never use or ask for an MP Group partner API key.** It controls MP Group's whole portfolio. This integration only needs two keys that belong to one test merchant account. If the user offers a partner key, decline and explain why.
5. **Test money only.** Use NMI test card numbers. Never run anything against a live account, never set `NMI_ENV=live`, and never process a real card. Going live is handled by people (see Step 7).
6. **The server decides the price.** The browser sends a product ID, never an amount. The server looks up the amount.
7. **Card details never touch the app's servers.** Use NMI's Payment Component, which turns card details into a token in the browser. The server only ever sees the token.
8. **Log only safe fields:** transaction ID, response code, response text. Never log request bodies, headers, tokens, or raw error objects from HTTP clients (they can contain the `Authorization` header).

Read `references/security-rules.md` before writing any code. It explains each rule and how to apply it.

## Before you start

Check that the **NMI Developer Docs MCP** is connected (a tool from `https://docs.nmi.com/mcp`). Use it to look up current NMI details instead of guessing. It needs no account or key. If it isn't connected, tell the user to run the command for the tool they're using, then restart it:

- **Claude Code:** `claude mcp add --transport http nmi-docs https://docs.nmi.com/mcp`
- **Codex:** `codex mcp add nmi-docs --url https://docs.nmi.com/mcp`

Then ask the user only these questions, if the project doesn't already answer them:
- Which language/framework the app uses (detect it from the project files first).
- Whether they have their MP Group **test** account keys ready (yes/no only, never the values).

## The flow

### Step 1: Protect secrets first

Before any payment code:
- Make sure `.gitignore` contains `.env` and `.env.*` (but not `.env.example`).
- Create `.env.example` with placeholder names only:
  ```
  NMI_ENV=test
  NMI_API_BASE_URL=https://secure.nmi.com
  NMI_PUBLIC_TOKENIZATION_KEY=replace-me
  NMI_PRIVATE_API_KEY=replace-me
  ```
- Add a hard block on reading the real file where the tool supports one:
  - **Claude Code:** add a deny rule in `.claude/settings.json`:
    ```json
    { "permissions": { "deny": ["Read(./.env)", "Read(./.env.*)"] } }
    ```
  - **Codex or any other tool:** if you can't add a block like this, rule 1 still applies in full. Never open `.env`, and tell the user there's no hard block so they know to watch for it.
- Tell the user: "Copy `.env.example` to `.env` and fill in the two keys from your MP Group test account yourself. Don't paste them here."

About `NMI_API_BASE_URL`: test accounts created in MP Group's partner portal are expected to run on `https://secure.nmi.com`. Accounts from NMI's public sandbox sign-up use `https://sandbox.nmi.com`. If the first test sale returns an authentication error, this setting is the first thing to check.

### Step 2: Look up the Payment Component package

Use the NMI Docs MCP to find the Payment Component's install instructions for the project's framework (React or vanilla JS).

NMI's docs sometimes show **placeholder** package names like `USER.PAYNPMPACKAGE` or `USER.PAYREACTNPMPACKAGE`. If you see a name starting with `USER.`, it is not real. **Do not guess a package name or install anything similar-sounding.** Stop and ask the user for the real package name (MP Group can get it from NMI). If they can't get it, use the Collect.js fallback described in `references/checkout-flow.md`.

### Step 3: Build the checkout page

Follow `references/checkout-flow.md`. In short:
- Show one product with a fixed price (default: $25.00, product ID `pilot-item`).
- Mount the Payment Component with the **public** tokenization key.
- On submit, send `{ productId, paymentToken }` to the app's own server. No amount.
- Disable the pay button while a payment is in progress so a double-click can't charge twice.
- Show a clear success screen (with transaction ID) or a decline message.

### Step 4: Build the server endpoint

Follow `references/checkout-flow.md`. In short:
- Look up the amount from the server-side product list.
- Call NMI's v5 sale endpoint: `POST {NMI_API_BASE_URL}/api/v5/payments/sale` with the **private** key in the `Authorization` header, and body `{ "amount": <number>, "payment_details": { "payment_token": "<token>" } }`.
- Treat `response === "1"` as approved. Anything else is a decline or error; show `response_text` to the user.
- Refuse to run if `NMI_ENV` is not `test`.
- Confirm any field names you're unsure of with the NMI Docs MCP.

### Step 5: Test it

Follow `references/testing.md`. Tell the user which test card to type **into the payment form in the browser** (never into the chat). Confirm:
- An approved sale shows a transaction ID.
- A decline is handled cleanly (use the test-only decline switch in `references/testing.md`).
- No key, token, or card number appears in the code, terminal output, logs, or git history.

### Step 6: Report back

Summarize for the user in plain language:
- What was built and which files changed.
- The test results (approved and declined).
- Anything that needed a manual fix, so MP Group can improve this skill.

### Step 7: Going live (hand off, don't do)

If the user asks about real payments, do **not** change any settings yourself. Read `references/go-live.md` and walk them through it. The key message: their client must be approved by MP Group and its processor first, and MP Group's contact will set up the live account.

## Out of scope for this pilot

Don't build these, even if asked. Tell the user they're planned for later and MP Group can help: refunds or voids, saved cards / Customer Vault, recurring billing, ACH, Apple Pay or Google Pay, 3D Secure, webhooks, surcharging, in-person or terminal payments, and the Authorize.net or MX Merchant gateways.

## Optional: NMI account MCP

NMI also publishes a read-only account MCP (`@nmipayments/nmi-mcp`) that looks up transactions. It is **not needed** for this pilot. If the user wants it anyway: test account keys only, `NMI_ENVIRONMENT=sandbox` only with test data, added only to the user's personal settings (never a committed `.mcp.json` in Claude Code or a committed `.codex/config.toml` in Codex), and the user enters the key themselves.
