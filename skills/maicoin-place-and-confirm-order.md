---
name: maicoin-place-and-confirm-order
description: Place a spot order on MAX Exchange (MaiCoin) and confirm it actually landed, using client_oid for safe retry and respecting per-market precision rules.
api: MAX Exchange REST API v3
generated: '2026-08-25'
method: generated
source: openapi/maicoin-max-v3-openapi.json + https://max-api.maicoin.com/api/doc/external/v3
operations:
  - getApiV3Markets
  - getApiV3Timestamp
  - postApiV3WalletPathWalletTypeOrder
  - getApiV3Order
  - deleteApiV3Order
---

# Place and confirm a spot order on MAX

Base URL: `https://max-api.maicoin.com`

## Before you start

Authentication is HMAC-SHA256 over a base64 payload. Every private call needs
`X-MAX-ACCESSKEY`, `X-MAX-PAYLOAD` and `X-MAX-SIGNATURE`. Build the payload by merging your
parameters with a `path` field and a millisecond `nonce`, JSON-serialising, then base64-encoding;
the signature is the hex HMAC-SHA256 of that payload string keyed by your secret. See
`authentication/maicoin-authentication.yml`.

The nonce must be within 30 seconds of server time and can be used only once.

## Step 1 — Read the market rules first

Call `getApiV3Markets` (`GET /api/v3/markets`, no auth required) and find your market.
You need four fields from it before you can compose a valid order:

- `base_unit_precision` — decimal places allowed on `volume`
- `quote_unit_precision` — decimal places allowed on `price`
- `min_base_amount` — smallest `volume` accepted
- `min_quote_amount` — smallest notional accepted

Rounding to the wrong precision is the most common rejection. Do not assume 8 decimals.

Also check `market_status` is `active`. A `suspended` or `cancel-only` market rejects new orders.

## Step 2 — Sync your clock

Call `getApiV3Timestamp` (`GET /api/v3/timestamp`, no auth). Compare it to your local clock and
carry the offset into every nonce you generate. A drift over 30 seconds fails every private call.

## Step 3 — Generate a client_oid

Generate a unique string of at most 36 characters and keep it. MAX enforces uniqueness per account
for 24 hours, which is what makes step 5 safe.

## Step 4 — Submit the order

Call `postApiV3WalletPathWalletTypeOrder`
(`POST /api/v3/wallet/{path_wallet_type}/order`, `path_wallet_type` = `spot`).

Body: `market`, `side` (`buy`/`sell`), `volume` — all required — plus `price` for a limit order,
`ord_type` (`limit`, `market`, `stop_limit`, `stop_market`, `post_only`, `ioc_limit`),
`stop_price` for stop orders, and your `client_oid`.

Send `price` and `volume` as STRINGS. MAX transports all monetary values as decimal strings.

**A 200 does not mean the order is on the book.** Order placement is asynchronous — a success
response means the request was accepted. Do not report success to the user yet.

## Step 5 — If the call failed or timed out, do NOT resubmit blindly

Retrying the same `client_oid` inside 24 hours is REJECTED, not deduplicated — you will get an
error, not the original order. That is the safe outcome, but you must handle it:

1. Retry the submit once with the SAME `client_oid`.
2. If it is rejected as a duplicate, the first attempt landed. Go to step 6 and read the state.
3. Never generate a fresh `client_oid` for a retry — that is how an agent double-fills.

## Step 6 — Confirm the real state

Call `getApiV3Order` (`GET /api/v3/order`) with either `id` or `client_oid`.
Read `state`, `remaining_volume` and `executed_volume`. This is the authoritative answer.

For anything long-running, subscribe to the WebSocket `user` channel with the `order` filter and
read `order_update` events instead of polling — MaiCoin recommends this explicitly.

## Step 7 — Cancelling

Call `deleteApiV3Order` (`DELETE /api/v3/order`) with `id` or `client_oid`.

Cancellation is asynchronous too, and an order being cancelled may still have unfilled trades in
flight. Always re-read the order state afterwards rather than assuming the cancel completed. Under
load, status updates can lag.

## Errors and limits

Errors come back as `{"success": false, "error": {"code": ..., "message": ..., "info": {...}}}`.
See `errors/maicoin-problem-types.yml`.

Rate limit is 1200 requests per minute per account on private endpoints. **No rate-limit headers
are returned** — you must count your own requests. See `rate-limits/maicoin-rate-limits.yml`.
