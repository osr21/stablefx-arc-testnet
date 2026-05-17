# Architecture

## Overview

StableFX is a pnpm monorepo with two deployed artifacts:

- **`artifacts/stablefx`** — React + Vite frontend served as a static SPA
- **`artifacts/api-server`** — Express 5 REST API with PostgreSQL

Both are routed through a shared reverse proxy at the project root. The frontend calls `/api/*` which the proxy forwards to the API server.

## Monorepo layout

```
stablefx-arc-testnet/
├── artifacts/
│   ├── stablefx/           # React SPA (Vite)
│   └── api-server/         # Express 5 backend
├── lib/
│   ├── api-spec/           # OpenAPI 3.1 contract + Orval codegen
│   └── db/                 # Drizzle ORM schema
└── scripts/                # Utility scripts
```

## Frontend (`artifacts/stablefx`)

### Pages

| Route | Component | Purpose |
|---|---|---|
| `/` | `swap.tsx` | Stablecoin swap UI |
| `/liquidity` | `liquidity.tsx` | LP deposit/withdraw |
| `/admin` | `admin.tsx` | Analytics dashboard |
| `/history` | `history.tsx` | Trade history |
| `/setup` | `setup.tsx` | Circle API key setup |

### Key hooks

| Hook | File | Purpose |
|---|---|---|
| `useCurveSwap` | `use-curve-swap.ts` | Quote + settle swap via Curve |
| `useCurvePoolStats` | `use-curve-lp.ts` | Read pool reserves, LP balances |
| `useCurveAddLiquidity` | `use-curve-lp.ts` | Step-machine for LP deposit |
| `useCurveRemoveLiquidity` | `use-curve-lp.ts` | Step-machine for LP withdraw |
| `usePermit2` | `use-permit2.ts` | Circle RFQ + Permit2 settlement |
| `useWallet` | `use-wallet.ts` | MetaMask connect + Arc chain switch |

### Wagmi config

Chain ID 5042002 is defined in `lib/wagmi.ts` as a custom viem chain. `useReadContracts` and `writeContractAsync` from wagmi v2 are used throughout.

## Backend (`artifacts/api-server`)

### Routes (`src/routes/fx.ts`)

```
GET  /api/fx/rates          — EUR/USD rate (open.er-api.com, 60s cache)
POST /api/fx/quote          — Return quote (amount, pair validation)
POST /api/fx/trade          — Record completed trade in DB
GET  /api/fx/trades         — List trades (optional ?address filter)
GET  /api/fx/admin/stats    — Aggregate stats for analytics dashboard
```

### Database schema (`lib/db/src/schema.ts`)

```sql
trades (
  id          SERIAL PRIMARY KEY,
  address     TEXT NOT NULL,
  sell_token  TEXT NOT NULL,
  buy_token   TEXT NOT NULL,
  sell_amount TEXT NOT NULL,
  buy_amount  TEXT NOT NULL,
  tx_hash     TEXT NOT NULL,
  created_at  TIMESTAMPTZ DEFAULT now()
)
```

## On-chain settlement

### Curve path (default)

```
User wallet
  │
  ├─ ERC-20.approve(CURVE_POOL, netSellAmt)   [if allowance insufficient]
  ├─ ERC-20.transfer(FEE_WALLET, feeAmt)      [if platform fee configured]
  └─ CurvePool.exchange(i, j, dx, min_dy)
```

### Circle path (optional, requires API key)

```
User wallet
  │
  ├─ Permit2.permit(...)     [sign off-chain]
  └─ FxEscrow.swap(...)      [Circle settles via Permit2]
```

## Data flow diagram

```
User Input
    │
    ▼
get_dy (on-chain read, no gas)
    │
    ▼
Quote displayed (30s TTL)
    │  [user confirms]
    ▼
Check allowance (on-chain read)
    │
    ├─ [insufficient] → approve() → wait for receipt
    │
    ├─ [fee > 0] → transfer() → wait for receipt
    │
    └─ exchange() → wait for receipt
              │
              ▼
        POST /api/fx/trade (record in DB)
              │
              ▼
        Success card + Arcscan link
```

