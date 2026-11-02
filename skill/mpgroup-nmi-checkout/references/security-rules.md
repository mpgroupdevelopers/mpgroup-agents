# Security rules, explained

Read this before writing payment code. Each rule says what to do and why.

## The three kinds of keys

| Key | Where it lives | What it can do | Safe in browser? |
|---|---|---|---|
| Public tokenization key | `NMI_PUBLIC_TOKENIZATION_KEY`, sent to the checkout page | Turns card details into a one-time token | Yes |
| Private API key | `NMI_PRIVATE_API_KEY`, server only | Charges tokens on ONE merchant account | **No, never** |
| MP Group partner API key | MP Group's own systems only | Controls every merchant MP Group manages | **Never used here** |

If a framework needs a prefix to expose a variable to the browser (for example `NEXT_PUBLIC_` or `VITE_`), only the **public** tokenization key may get that prefix. The private key must never have a browser-exposed prefix.

## Keeping keys out of your view

- Do not open `.env` with any tool, `cat`, `grep`, editor, or script, including to "check formatting."
- To confirm setup, write a startup check like this (adapt to the language):

  ```js
  const required = ["NMI_ENV", "NMI_API_BASE_URL", "NMI_PUBLIC_TOKENIZATION_KEY", "NMI_PRIVATE_API_KEY"];
  const missing = required.filter((name) => !process.env[name] || process.env[name] === "replace-me");
  if (missing.length) {
    console.error("Missing settings in .env: " + missing.join(", "));
    process.exit(1);
  }
  ```

  It prints names, never values.
- Never echo environment variables in the terminal (`env`, `printenv`, `echo $NMI_...`).
- If a key value ever appears in the chat, the code, a log, or git: stop, tell the user plainly, and tell them to delete that key in the NMI merchant portal (Settings → Security Keys) and create a new one. Don't try to hide it.

## Keeping keys out of git

- `.gitignore` must contain `.env` and `.env.*` but keep `.env.example` tracked (add `!.env.example`).
- Before saying you're done, check `git status` and confirm `.env` is not staged or tracked. If it was ever committed, tell the user; removing it from history and replacing the keys is required.
- Never write keys into `.mcp.json`, `.codex/config.toml`, `package.json`, Docker files, CI config, or test fixtures.

## Keeping card data out of everything

- The Payment Component collects card details inside NMI's secure fields. The app only receives a token.
- Never build your own card number or CVV input fields.
- Never log, store, or return the payment token after use.
- Test card numbers are public and fine to mention in chat as instructions ("type 4111 1111 1111 1111 into the form"). Real card numbers must never appear anywhere.

## Safe logging

Log only:

```js
console.log("NMI sale", { transactionId: data.id, response: data.response, responseText: data.response_text });
```

Never log:
- the request body or headers,
- the full NMI response,
- a raw error object from axios, fetch wrappers, or similar libraries (they often include the request config, with the `Authorization` header). Log `error.message` and the HTTP status only.

## Test vs live

- The server must refuse to process if `NMI_ENV` is not exactly `test`. This pilot never processes live money.
- Do not add a code path, flag, or instruction that switches to live. Going live is a separate, human-run process (see `go-live.md`).

## Price integrity

- Keep a server-side list of products and prices. The browser sends a product ID.
- Reject unknown product IDs.
- Never accept an `amount` field from the browser, even "for convenience."
