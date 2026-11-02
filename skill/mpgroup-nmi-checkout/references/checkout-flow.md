# Checkout flow

One product, one fixed price, one sale. Adapt the examples to the project's language and framework. The examples use Node.js 18+ (built-in `fetch`) and Express because they're short and readable. Use the NMI Docs MCP to confirm any NMI detail you're unsure of; NMI's docs are the source of truth if they differ from this file.

## How the pieces fit

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

## Server side (Node.js / Express example)

```js
// server/payments.js
const express = require("express");
const router = express.Router();

// Prices live on the server. The browser never sends an amount.
const PRODUCTS = {
  "pilot-item": { name: "Pilot item", amount: 25.0 },
};

router.post("/api/checkout", async (req, res) => {
  if (process.env.NMI_ENV !== "test") {
    return res.status(503).json({ success: false, error: "Payments are disabled: this pilot runs in test mode only." });
  }

  const { productId, paymentToken } = req.body || {};
  const product = PRODUCTS[productId];
  if (!product || typeof paymentToken !== "string" || !paymentToken) {
    return res.status(400).json({ success: false, error: "Invalid checkout request." });
  }

  let amount = product.amount;
  // Test-only decline switch (see testing.md). Never active outside test mode.
  if (process.env.NMI_TEST_FORCE_DECLINE === "true") amount = 0.5;

  try {
    const nmiRes = await fetch(`${process.env.NMI_API_BASE_URL}/api/v5/payments/sale`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: process.env.NMI_PRIVATE_API_KEY,
      },
      body: JSON.stringify({
        amount,
        payment_details: { payment_token: paymentToken },
      }),
    });

    const data = await nmiRes.json().catch(() => ({}));
    // Safe log: no headers, no body, no token.
    console.log("NMI sale", { status: nmiRes.status, transactionId: data.id, response: data.response, responseText: data.response_text });

    if (data.response === "1") {
      return res.json({ success: true, transactionId: data.id });
    }
    return res.status(402).json({ success: false, error: data.response_text || "The payment was declined." });
  } catch (err) {
    // Never log the raw error object.
    console.error("NMI sale failed:", err.message);
    return res.status(502).json({ success: false, error: "Payment service unavailable. Please try again." });
  }
});

module.exports = router;
```

Notes:
- Mount with `app.use(express.json())` and `app.use(router)`.
- Add the startup check from `security-rules.md`.
- For other languages, keep the same shape: price lookup, test-mode guard, one HTTPS call, safe log, simple JSON result.

## Browser side: Payment Component (preferred)

NMI's Payment Component is NMI's recommended way to collect card details for new integrations. Get the install command and exact component and prop names from the NMI Docs MCP ("Payment Component quick start"). As of NMI's docs, the React version is a payments component that takes:
- `tokenizationKey`: the **public** tokenization key,
- `onPay`: a callback that receives the payment token after the customer submits.

Confirm the exact import name and what `onPay` receives with the Docs MCP. If the package name starts with `USER.`, it's a placeholder: stop and ask the user (see SKILL.md Step 2).

Shape of the page logic (adapt to the real component API):

```jsx
const [busy, setBusy] = useState(false);
const [result, setResult] = useState(null);

async function handlePay(paymentToken) {
  if (busy) return;           // block double charges
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

How the public key reaches the browser depends on the framework (for example a `NEXT_PUBLIC_` or `VITE_` variable, or the server rendering it into the page). Only the public tokenization key may be exposed this way.

## Browser side: Collect.js fallback

Use this only if the real Payment Component package name can't be found. Collect.js is NMI's older tokenization script; it loads from a script tag, so no package name is needed. Confirm current details with the NMI Docs MCP (search "Collect.js quick start"), including:
- the script URL and the attribute that carries the public tokenization key,
- how the callback returns the token,
- whether the v5 sale endpoint accepts Collect.js tokens or the integration needs NMI's classic transaction API instead.

Tell the user you're using the fallback and why, so MP Group knows to fix the package-name gap.

## Done means

- [ ] `.env` is gitignored; `.env.example` has placeholders only.
- [ ] The server owns the price; the browser sends only `productId` and the token.
- [ ] The server refuses to run unless `NMI_ENV=test`.
- [ ] Logs contain only transaction ID, response code, response text, HTTP status.
- [ ] The pay button can't be double-submitted.
- [ ] Approved and declined test payments both work (see `testing.md`).
