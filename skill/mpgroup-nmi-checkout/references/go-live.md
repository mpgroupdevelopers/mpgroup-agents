# Going live with MP Group

You (the assistant) don't take any of these steps. Explain them to the user in plain language and point them to MP Group.

## What to tell the user

Real payments need an approved merchant account. MP Group handles that with a person, not a setting.

1. **Contact MP Group.** Christian Brown, christian.brown@gompgroup.com, 678-740-4364. Mention that the integration already works in test mode.
2. **The client applies.** MP Group signs the client up with its processor, Priority, and the client submits the business details and documents Priority asks for.
3. **Approval.** Priority reviews and approves the business. Timing depends on the business; MP Group will give an estimate.
4. **MP Group sets up the live account.** Once approved, MP Group creates the client's live NMI gateway account.
5. **The client or developer creates live keys** in the live merchant portal (Settings → Security Keys): a public tokenization key and a private API key.
6. **Swap the settings in the hosting platform's secret store**, never in code or a committed file:
   - `NMI_PUBLIC_TOKENIZATION_KEY` and `NMI_PRIVATE_API_KEY` → the live keys
   - `NMI_API_BASE_URL` → `https://secure.nmi.com`
   - `NMI_ENV` → `live` (the code must be updated to allow live mode; MP Group can review that change)
   - remove `NMI_TEST_FORCE_DECLINE`
7. **First real transaction:** a person runs one small real payment, checks it in the live merchant portal, and refunds it from the portal.

## Before go-live, also confirm with MP Group

- That the Payment Component and v5 API are enabled on the client's live account.
- HTTPS is on for the whole site.
- Who the client calls for support.

## What not to do

- Don't change `NMI_ENV`, keys, or the base URL yourself.
- Don't remove the test-mode guard from the server code without the user explicitly asking and MP Group reviewing it.
- Don't ask for or use an MP Group partner API key.
