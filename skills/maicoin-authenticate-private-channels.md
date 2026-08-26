---
name: maicoin-authenticate-private-channels
description: Authenticate to MAX Exchange private WebSocket channels and stream your own order, trade and balance updates — including the signing scheme that differs from the REST API.
api: MAX Exchange WebSocket API
generated: '2026-08-25'
method: generated
source: https://maicoin.github.io/max-websocket-docs/authentication.md + private_channels.md + private_channels_mwallet.md
operations:
  - auth
---

# Authenticate to MAX private WebSocket channels

Endpoint: `wss://max-stream.maicoin.com/ws`

## Step 0 — The signing scheme is NOT the same as REST

This is the single most common failure on this surface, and MaiCoin does not flag it as a
difference anywhere.

- **REST**: signature = HMAC-SHA256 of the **base64 payload string** (parameters + `path` + `nonce`).
- **WebSocket**: signature = HMAC-SHA256 of the **nonce string alone**.

Reusing your REST signing routine here returns error `1007: authentication failed`.

```javascript
const crypto = require("crypto");
const nonce = Date.now();
const signature = crypto.createHmac("sha256", API_SECRET).update("" + nonce).digest("hex");
```

## Step 1 — Check your key permissions

Get keys from https://max.maicoin.com/api_tokens. Two grants matter here:

- **read permission for Order / Trade** — required for order and trade channels
- **read permission for Account & Personal Information** — required for balance updates

Without the right grant the auth succeeds but the events never arrive, which is easy to misdiagnose
as a connection problem.

## Step 2 — Send the auth command

```json
{
  "action": "auth",
  "apiKey": "...",
  "nonce": 1591690054859,
  "signature": "...",
  "filters": ["order", "trade"],
  "id": "client-id"
}
```

Nonce is milliseconds since epoch, must be within 30 seconds of server time, and is single-use.

Success looks like `{"e": "authenticated", "i": "client-id", "T": 1591686735192}`.

## Step 3 — Choose your filters deliberately

If `filters` is omitted or empty you get `order`, `trade` and `account` by default.

| Filter | Effect |
|---|---|
| `order` | Order snapshot on connect, then `order_update` events |
| `trade` | Last 100 trades as a snapshot, then `trade_update` events |
| `trade_update` | Updates only — **no snapshot**. Use this if you do not want 100 historical trades on every reconnect |
| `fast_trade_update` | Lower-latency trade updates |
| `account` | Balance updates |
| `mwallet_order`, `mwallet_trade`, `mwallet_fast_trade_update`, `mwallet_account`, `ad_ratio`, `borrowing` | Margin (m-wallet) equivalents — must be requested explicitly |

On a reconnect loop, `trade` re-sends the 100-trade snapshot every time. Prefer `trade_update`
unless you genuinely need the backfill.

## Step 4 — Read the events

All private events arrive on channel `c: "user"`.

Order events (`e`): `order_snapshot`, `order_update`, `mwallet_order_snapshot`,
`mwallet_order_update`. Orders are under `o[]` with `i` (id), `sd` (side: bid/ask), `ot` (order
type), `p` (price), `v` (volume), `rv` (remaining), `ev` (executed), `S` (state), `M` (market),
`ci` (your client_oid), `gi` (group id).

Trade events (`e`): `trade_snapshot`, `trade_update`, and the mwallet equivalents. Trades are under
`t[]` with `i` (id), `M` (market), `p` (price), `v` (volume), `f` (fee), `fc` (fee currency),
`fd` (fee discounted), `fn` (funds), `m` (maker), `oi` (order id).

Match `oi` on a trade back to `i` on an order to attribute fills.

## Step 5 — Prefer this over polling

MaiCoin explicitly recommends the WebSocket order channel over polling `GET /api/v3/order`,
because REST order state can lag under load and REST has no rate-limit headers to pace against.

## Reconnection

The status endpoint at `https://status-api-max.maicoin.com/api/status/max-api` may lag up to 30
seconds and may never reflect a brief WebSocket drop. MaiCoin states plainly that trading programs
must implement their own automatic reconnection. Do not rely on the status page to detect a
dropped stream.
