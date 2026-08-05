# Undocumented API Behavior

This file is NOT an official schema source. It records behavior verified through
live sandbox/production testing that is absent from `docs.mayar.id` at the time
of writing. Treat every entry here as tentative — Mayar may document, change, or
remove this behavior without notice.

Use `references/api-sources.md` first. Only consult this file when the official
documentation does not define a field or endpoint you need, and you have
confirmed it truly is missing (not just under a different page). Do not silently
skip a documentation search and jump here.

If behavior recorded here conflicts with what you observe live, trust the live
API response and update this file — do not trust this file blindly either.

## Environment domains

- Sandbox base URL is `https://api.mayar.io/hl/v2` (dashboard `web.mayar.io`,
  keys from `web.mayar.io/api-keys`). This replaced `api.mayar.club` around
  Jul–Aug 2026. The old domain still resolves but `GET /payment-channels`
  always returns `data: null` there — do not use it.

## Sandbox invoice creation requires a session cookie

`POST /invoices/create` on `api.mayar.io` (sandbox) rejects QRIS/VA/e-wallet
creation with a `502 Bad Gateway` unless the request also carries a
`Cookie: connect.sid=...` header alongside the Bearer token. Production
(`api.mayar.id`) does not have this requirement.

To obtain the cookie: log into `web.mayar.io` in a browser, open DevTools →
Application → Cookies, copy `connect.sid`. Store it in an env var
(e.g. `MAYAR_SANDBOX_COOKIE`) and attach it only when the target environment is
sandbox. Sandbox `qr_string` returns a mock literal `"some-random-qr-string"` —
this is expected, not a bug.

## `paymentMethod` values for invoice creation

Verified against `POST /invoices/create` on both `api.mayar.io` and
`api.mayar.id`. None of these values appear on the Create Invoice documentation
page as of writing.

| Method | Value |
|---|---|
| QRIS | `qris` (lowercase) |
| VA — any bank | `va/{bank_code_lowercase}` (e.g. `va/bni`, `va/mandiri`, `va/bri`, `va/bsi`, `va/cimb`, `va/permata`) |
| E-wallet — any provider | `ewallet/{provider_lowercase}` (e.g. `ewallet/dana`, `ewallet/ovo`, `ewallet/shopeepay`, `ewallet/linkaja`) |
| E-wallet GoPay | `ewallet/gopay` — requires separate account-level activation beyond the dashboard toggle |

Minimum string length for `paymentMethod` is 3 characters (`"va"` alone fails
validation).

`paymentDetail` is present directly in the create response — no follow-up GET is
needed.

VA response parsing:
`data.paymentDetail.virtual_account.channel_properties.virtual_account_number`
and `.customer_name`.

E-wallet response shape differs from QRIS/VA — there is no QR string or account
number. Instead `paymentDetail.actions[]` holds redirect targets:
`url_type: "WEB"` for browser and `url_type: "MOBILE"` for an app deep link.
Parse with `data.paymentDetail.actions.find(a => a.url_type === "MOBILE").url`.

## Membership/SaaS product — custom native checkout

The public membership docs describe dashboard-driven setup and the hosted
`membershipBillUrl` checkout. The flow below produces the same `paymentDetail`
shape as a regular invoice, enabling a custom checkout UI for membership
products. Verified in sandbox.

1. Create the product:
   `membership product create --data '{"name":"...","membershipInfo":{"type":"SAAS"}}'`.
   `membershipInfo.type` accepts `MEMBERSHIP`, `SAAS`, or `CREDIT` — uppercase,
   not documented publicly.
2. Create a priced tier:
   `membership tier create --data '{"productId":"...","name":"...","periods":[{"monthPeriod":1,"amount":50000}]}'`.
   The request field is `periods`; the response echoes it back as
   `membershipTierPeriod` — do not send `membershipTierPeriod` in the request,
   it is rejected.
3. Register a member:
   `membership register --data '{"productId":"...","membershipTierId":"...","membershipMonthlyPeriod":1,"customerInfo":{"email":"...","name":"...","mobile":"..."}}'`.
4. Create the billing invoice:
   `membership create-invoice <memberId> --productId <id>`. This only returns
   `membershipBillUrl` (a hosted page) — no `paymentDetail`.
5. To get a native-renderable `paymentDetail`, follow up with:
   `invoice edit <invoiceId> --data '{"items":[{"quantity":1,"rate":<amount>,"description":"..."}],"paymentMethod":"qris"}'`
   (or any `va/{bank}` / `ewallet/{provider}` value from the table above). The
   `items` array must be repeated on every edit call — it is not preserved from
   invoice creation. The response then contains the same `paymentDetail` shape
   used for regular invoices.
