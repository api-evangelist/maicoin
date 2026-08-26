---
name: maicoin-stream-market-data
description: Subscribe to MAX Exchange real-time market data over WebSocket — order book, ticker, k-line, trades and market status — including the abbreviated key decoding and connection keep-alive that clients get wrong.
api: MAX Exchange WebSocket API
generated: '2026-08-25'
method: generated
source: https://maicoin.github.io/max-websocket-docs/ + asyncapi/maicoin-max-websocket-asyncapi.yml
operations:
  - sub
  - unsub
---

# Stream MAX market data over WebSocket

Endpoint: `wss://max-stream.maicoin.com/ws`

Public market-data channels need no authentication.

## Step 1 — Connect and keep the connection alive

You must send a **ping frame** at least every 130 seconds or the server closes the connection.
Send one every 30 seconds. Some WebSocket libraries do this for you — check yours before relying
on it, because a silent disconnect looks identical to a quiet market.

## Step 2 — Subscribe

```json
{
  "action": "sub",
  "subscriptions": [
    {"channel": "book", "market": "btctwd", "depth": 1},
    {"channel": "trade", "market": "btctwd"}
  ],
  "id": "client1"
}
```

The `id` is yours; it comes back on the response so you can correlate.
The server replies with an `e: "subscribed"` event echoing the subscription list.

Available channels:

| Channel | Parameters | Carries |
|---|---|---|
| `book` | `market`, `depth` (1, 5, 10, 20, 50 — default 50) | Order book snapshot then updates |
| `trade` | `market` | Public trades |
| `ticker` | `market` | Open/high/low/close/volume |
| `kline` | `market`, `resolution` (1m, 5m, 15m, 30m, 1h, 2h, 4h, 6h, 12h, 1d — default 1m) | Candlesticks |
| `market_status` | none | Per-market status, precision, minimum amounts |
| `pool_quota` | `currency` | M-wallet available loan quota |

## Step 3 — Decode the abbreviated keys

Every field name is abbreviated to save bandwidth. Nothing is self-describing. The ones you need
most:

`e`=event, `c`=channel, `M`=market, `T`=timestamp, `p`=price, `v`=volume, `a`=asks, `b`=bids,
`O`=open, `H`=high, `L`=low, `C`=close, `k`=kline, `tk`=ticker, `t`=trades, `sd`=side,
`R`=resolution, `ST`=start time, `ET`=end time, `x`=closed, `fi`=first id, `li`=last id.

The full table is in the WebSocket docs README and in
`conventions/maicoin-conventions.yml`.

## Step 4 — Handle book continuity

Each `book` event carries `fi` (first update id), `li` (last update id) and `v` (version).
Use them to verify you have not missed an update. If the sequence breaks, resubscribe to get a
fresh snapshot rather than continuing from a gapped book.

The first event after subscribing is `e: "snapshot"`; everything after is `e: "update"`.

## Step 5 — Respect the rate limits

- 20 request messages per second
- 200 request messages per minute
- 600 connections per hour per IP
- 1440 connections per day per IP

A 429 causes an **automatic IP ban**. The `Retry-After` header holds a **Unix timestamp in
seconds**, not delta-seconds — do not feed it to a standard RFC 9110 Retry-After handler, which
would sleep for decades. Compute `retry_after - now` yourself.

Requesting again during a penalty resets and extends the ban. Wait the full period.

## Step 6 — Unsubscribe

```json
{"action": "unsub", "subscription": [{"channel": "trade", "market": "btctwd"}], "id": "client1"}
```

## Errors

Errors arrive as an array under `E`, formatted `"E-<code>: <message>"`.
See `errors/maicoin-error-codes.yml` — 1004 invalid action, 1005 invalid json,
1006 nonce skew, 1007 auth failed, 1012 nonce reused.
