# Testing

All testing uses an MP Group **test** merchant account. Test accounts never send transactions to a payment processor and are not billed. Confirm current test values with the NMI Docs MCP ("Test and Deploy") if anything below doesn't work.

## Test cards

Tell the user to type these **into the payment form in the browser**, never into the chat.

| Card | Number |
|---|---|
| Visa | 4111 1111 1111 1111 |
| Mastercard | 5431 1111 1111 1111 |
| Discover | 6011 0009 9130 0009 |
| American Express | 3411 1111 1111 111 |

- Expiration: **10/29**
- CVV: any 3 digits (4 for Amex). NMI's docs say **999** simulates a CVV match.
- For an address match, NMI's docs say to use **888** in the street address and **77777** as the ZIP.

## Approved test

1. Start the app with `NMI_ENV=test`.
2. Pay with the Visa test card.
3. Expect a success screen with a transaction ID, and a log line showing `response: "1"`.

## Declined test

NMI's test environment declines amounts under $1.00. Because the price is fixed on the server, use the test-only switch instead of changing the price:

1. The user adds `NMI_TEST_FORCE_DECLINE=true` to their `.env` themselves, and restarts the app.
2. Pay with the Visa test card.
3. Expect a clean decline message, no crash, and the pay button usable again.
4. The user removes `NMI_TEST_FORCE_DECLINE` (or sets it to `false`) afterward.

Don't edit `.env` yourself; tell the user what to change.

## If something fails

| What you see | Most likely cause | What to do |
|---|---|---|
| Authentication / access denied from NMI | Wrong `NMI_API_BASE_URL` for this account type, or the key lacks permission | Ask the user to check the base URL (`https://secure.nmi.com` for MP Group partner-portal test accounts, `https://sandbox.nmi.com` for NMI public sandbox accounts) and that the private key has API permissions |
| Payment form never appears | Wrong or missing public tokenization key, or package not loaded | Have the user confirm the public key is set; check the browser console for errors (don't print key values) |
| Declines on every card | Amount under $1.00, or the decline switch left on | Check `NMI_TEST_FORCE_DECLINE` is off |
| "Payments are disabled" | `NMI_ENV` isn't `test` | Ask the user to set `NMI_ENV=test` |

## Final safety check

Before reporting done:
- Run `git status` and confirm `.env` is not tracked or staged.
- Search the project (excluding `.env` and dependency folders) for anything that looks like a key, and confirm none of the user's key values were written into code. You can't see the values, so search for suspicious long strings in source and config files, and ask the user to confirm.
- Confirm logs from the test runs show only transaction ID, response code, response text and HTTP status.
