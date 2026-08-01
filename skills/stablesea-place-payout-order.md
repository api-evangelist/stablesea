---
name: place-payout-order
description: Place a stablecoin payout order on the Stable Sea Terminal API — from organization setup through offering, quote, and order.
api: Stable Sea Terminal API
base_url: https://api-sandbox.stablesea.com/v1
auth: HTTP bearer — send `Authorization: Bearer <api_key>`
operations:
  - GET /liquidity-providers
  - GET /liquidity-providers/{provider_name}/exchange-rate
  - POST /organizations
  - POST /organizations/{organization_id}/external_payment_instruments
  - POST /organizations/{organization_id}/offerings
  - POST /organizations/{organization_id}/quotes
  - POST /organizations/{organization_id}/orders
  - GET /organizations/{organization_id}/orders/{order_id}
---

# Place a payout order (Stable Sea Terminal API)

All requests go to `https://api-sandbox.stablesea.com/v1` and must include
`Authorization: Bearer <api_key>`. Every create (POST) call below accepts an
`Idempotency-Key` header — send a stable UUID per logical action so retries are safe.

## Steps

1. **Create the organization.** `POST /organizations` with the org name and contact.
   Capture the returned `id` as `organization_id`.

2. **Add a payout destination.** `POST /organizations/{organization_id}/external_payment_instruments`
   with the `alias`, `currency`, and `method` details (e.g. a Solana address, a COP
   bank account, or NGN funding details). Capture its `id`.

3. **Pick a liquidity provider.** `GET /liquidity-providers` to list providers, then
   `GET /liquidity-providers/{provider_name}/exchange-rate` to check the current rate
   for your trading pair.

4. **Create an offering.** `POST /organizations/{organization_id}/offerings` describing
   the `payin`/`payout` currencies and order type. Capture the `offering_id`.

5. **Quote it.** `POST /organizations/{organization_id}/quotes` with `offering_id`.
   The response includes `exchange_rate`, `payin`, `payout`, and expiry timestamps
   (`submission_expires_at`, `funding_expires_at`) — the quote must be used before it
   expires. Capture the `quote_id`.

6. **Place the order.** `POST /organizations/{organization_id}/orders` with `quote_id`.
   The response includes `reference_number`, `status`, and `funding_instructions`.

7. **Track it.** `GET /organizations/{organization_id}/orders/{order_id}` to poll
   `status`. If an order fails, inspect `retry_order_id` for the retry chain.

## Conventions & errors

- **Idempotency:** put an `Idempotency-Key` header on every POST (create org, EPI,
  offering, quote, order). See `conventions/stablesea-conventions.yml`.
- **Errors:** failures return a JSON envelope `{ "error": <int>, "message": <string> }`
  (not RFC 9457). See `errors/stablesea-problem-types.yml`.
- **Expiry:** honor `submission_expires_at` / `funding_expires_at` on quotes; re-quote
  if expired rather than reusing a stale quote.
