# API Reference

The StableFX API server runs on Express 5 and is served at `/api`. All routes are defined in `artifacts/api-server/src/routes/fx.ts` and validated against the OpenAPI spec in `lib/api-spec/openapi.yaml`.

Base URL: `/api/fx`

---

## GET /fx/rates

Returns the current EUR/USD exchange rate for the indicative rate ticker.

**Response**
```json
{
  "eurUsd": 1.0823,
  "source": "open.er-api.com",
  "cachedAt": "2026-05-17T14:00:00.000Z"
}
```

Rate is fetched from `open.er-api.com` and cached for 60 seconds.

---

## POST /fx/quote

Validates and returns a swap quote. Does not touch the chain — quote data comes from the client (Curve on-chain call is made in the browser).

**Request body**
```json
{
  "sellToken": "USDC",
  "buyToken": "EURC",
  "sellAmount": "100.000000",
  "buyAmount": "92.451234",
  "price": "0.924512"
}
```

**Response**
```json
{
  "quoteId": "q_abc123",
  "sellToken": "USDC",
  "buyToken": "EURC",
  "sellAmount": "100.000000",
  "buyAmount": "92.451234",
  "price": "0.924512",
  "expiresAt": "2026-05-17T14:00:30.000Z"
}
```

---

## POST /fx/trade

Records a completed on-chain trade in the database.

**Request body**
```json
{
  "address": "0xUserWalletAddress",
  "sellToken": "USDC",
  "buyToken": "EURC",
  "sellAmount": "100.000000",
  "buyAmount": "92.451234",
  "txHash": "0xabc..."
}
```

**Response**
```json
{
  "id": 42,
  "address": "0xUserWalletAddress",
  "sellToken": "USDC",
  "buyToken": "EURC",
  "sellAmount": "100.000000",
  "buyAmount": "92.451234",
  "txHash": "0xabc...",
  "createdAt": "2026-05-17T14:00:31.000Z"
}
```

---

## GET /fx/trades

Returns a list of recorded trades.

**Query parameters**

| Param | Type | Description |
|---|---|---|
| `address` | string (optional) | Filter by wallet address |
| `limit` | number (optional, default 50) | Max records to return |
| `offset` | number (optional, default 0) | Pagination offset |

**Response**
```json
{
  "trades": [
    {
      "id": 42,
      "address": "0xUserWalletAddress",
      "sellToken": "USDC",
      "buyToken": "EURC",
      "sellAmount": "100.000000",
      "buyAmount": "92.451234",
      "txHash": "0xabc...",
      "createdAt": "2026-05-17T14:00:31.000Z"
    }
  ],
  "total": 1
}
```

---

## GET /fx/admin/stats

Returns aggregate analytics for the dashboard.

**Response**
```json
{
  "totalTrades": 156,
  "totalVolumeUsdc": "48230.120000",
  "uniqueWallets": 23,
  "pairBreakdown": {
    "USDC_EURC": 89,
    "EURC_USDC": 67
  },
  "dailyVolume": [
    { "date": "2026-05-17", "volumeUsdc": "3420.50", "tradeCount": 12 }
  ],
  "estimatedFeesUsdc": "96.46"
}
```

---

## Error responses

All errors follow a consistent shape:

```json
{
  "error": "Description of the problem"
}
```

| Status | Meaning |
|---|---|
| 400 | Validation error (missing or invalid fields) |
| 404 | Resource not found |
| 500 | Internal server error |

---

## OpenAPI spec

The authoritative API contract is at [`lib/api-spec/openapi.yaml`](../lib/api-spec/openapi.yaml).

To regenerate client hooks and Zod schemas:

```bash
pnpm --filter @workspace/api-spec run codegen
```

This outputs:
- `lib/api-spec/src/generated/stableFX.ts` — React Query hooks
- `lib/api-spec/src/generated/stableFXSchemas.ts` — Zod validation schemas

