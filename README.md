# StableFX — Arc Testnet

> USDC ↔ EURC stablecoin swap dApp on Arc Testnet (Chain ID 5042002), powered by the Curve StableSwap pool.

[![Arc Testnet](https://img.shields.io/badge/network-Arc%20Testnet-blue)](https://testnet.arc.network)
[![Chain ID](https://img.shields.io/badge/chain%20id-5042002-informational)](https://rpc.testnet.arc.network)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## Overview

StableFX is a non-custodial stablecoin exchange and liquidity management dApp built on the Arc Testnet. It uses a deployed Curve Finance StableSwap pool (A=1000, fee=0.01%) as its settlement layer for low-slippage USDC↔EURC swaps, with an optional Circle StableFX RFQ path for institutional users.

### Key features

| Feature | Description |
|---|---|
| Swap | USDC ↔ EURC swaps with on-chain Curve quotes and real ERC-20 settlement |
| Liquidity | Add / remove liquidity to the Curve pool and earn swap fees |
| Analytics | Daily volume chart, pair breakdown, and platform fee tracker |
| History | Per-wallet trade history persisted in Postgres |
| Circle path | Optional Circle StableFX RFQ API + Permit2 for institutional flow |
| Platform fee | Configurable basis-point fee collected on every swap |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  Browser (React + wagmi v2 + viem)                                  │
│                                                                     │
│  ┌──────────┐  ┌──────────┐  ┌─────────────┐  ┌───────────────┐   │
│  │  Swap    │  │ Liquidity│  │  Analytics  │  │   History     │   │
│  │  Page    │  │  Page    │  │  Dashboard  │  │   Page        │   │
│  └────┬─────┘  └────┬─────┘  └──────┬──────┘  └───────┬───────┘   │
│       │              │               │                  │           │
│  ┌────▼──────────────▼──────┐  ┌────▼──────────────────▼──────┐   │
│  │  wagmi / viem hooks      │  │  React Query + Orval hooks    │   │
│  │  (on-chain reads/writes) │  │  (REST API calls)             │   │
│  └──────────┬───────────────┘  └────────────────┬─────────────┘   │
└─────────────┼────────────────────────────────────┼─────────────────┘
              │ Arc Testnet RPC                     │ /api/*
              ▼                                     ▼
   ┌─────────────────────┐           ┌──────────────────────────┐
   │  Curve USDC/EURC    │           │  Express 5 API Server    │
   │  StableSwap Pool    │           │  (rates, trades, stats)  │
   │  0x2D84…0457        │           │  + PostgreSQL (Drizzle)  │
   └─────────────────────┘           └──────────────────────────┘
```

### Swap flow (Curve path)

1. User enters sell amount → `get_dy` called on-chain for live quote
2. Platform fee deducted from sell amount (configurable bps)
3. ERC-20 `approve` → Curve `exchange` (2 wallet prompts max)
4. Trade recorded in Postgres; txHash shown with Arcscan link

### Swap flow (Circle path — optional)

1. Circle StableFX RFQ API called for quote (requires `stablefx_api_key` in `localStorage`)
2. Permit2 signature → Circle API settlement
3. Platform fee transferred via Permit2 witness

### Liquidity flow

1. Enter USDC/EURC amounts → `calc_token_amount` previews LP tokens received
2. ERC-20 `approve` (per token) → Curve `add_liquidity`
3. LP tokens held in wallet; virtual price accretes swap fees automatically
4. Withdraw balanced or single-coin via `remove_liquidity` / `remove_liquidity_one_coin`

---

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | React 18, TypeScript 5.9, Vite 7 |
| Styling | Tailwind CSS v4, shadcn/ui |
| Wallet | wagmi v2, viem, MetaMask |
| API client | Orval (OpenAPI codegen) + React Query |
| Backend | Express 5, Node.js 24, TypeScript |
| Database | PostgreSQL + Drizzle ORM |
| Validation | Zod (v4), drizzle-zod |
| Monorepo | pnpm workspaces |

---

## Contract addresses (Arc Testnet — Chain ID 5042002)

| Contract | Address |
|---|---|
| USDC | `0x3600000000000000000000000000000000000000` |
| EURC | `0x89B50855Aa3bE2F677cD6303Cec089B5F319D72a` |
| Curve USDC/EURC pool | `0x2D84D79C852f6842AbE0304b70bBaA1506AdD457` |
| Permit2 | `0x000000000022D473030F116dDEE9F6B43aC78BA3` |
| FxEscrow | `0x43506849D7C04F9138D1A2050bbF3A0c054402dd` |

Curve pool parameters: A=1000, fee=0.01% (1bps). USDC = coin[0], EURC = coin[1].

---

## Getting started

### Prerequisites

- Node.js 24+
- pnpm 9+
- MetaMask (or any EIP-1193 wallet)
- Arc Testnet added to your wallet (RPC: `https://rpc.testnet.arc.network`, Chain ID: 5042002)

### Setup

```bash
# Clone the repo
git clone https://github.com/osr21/stablefx-arc-testnet.git
cd stablefx-arc-testnet

# Install dependencies
pnpm install

# Copy environment variables
cp .env.example .env
# Fill in DATABASE_URL and optionally PLATFORM_FEE_WALLET / PLATFORM_FEE_BPS
```

### Environment variables

| Variable | Required | Description |
|---|---|---|
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `SESSION_SECRET` | Yes | Express session secret (any random string) |
| `PLATFORM_FEE_WALLET` | No | Address to receive platform fees |
| `PLATFORM_FEE_BPS` | No | Fee in basis points (e.g. `20` = 0.20%) |

### Running locally

```bash
# Start the API server (port 5000 by default)
pnpm --filter @workspace/api-server run dev

# Start the frontend (Vite dev server)
pnpm --filter @workspace/stablefx run dev

# Push DB schema (first time or after schema changes)
pnpm --filter @workspace/db run push
```

### Useful commands

```bash
# Full typecheck across all packages
pnpm run typecheck

# Regenerate API hooks and Zod schemas from OpenAPI spec
pnpm --filter @workspace/api-spec run codegen

# Build all packages
pnpm run build
```

---

## Project structure

```
stablefx-arc-testnet/
├── artifacts/
│   ├── stablefx/               # React frontend
│   │   └── src/
│   │       ├── pages/
│   │       │   ├── swap.tsx        # Main swap UI
│   │       │   ├── liquidity.tsx   # LP deposit/withdraw UI
│   │       │   ├── admin.tsx       # Analytics dashboard
│   │       │   └── history.tsx     # Trade history
│   │       ├── hooks/
│   │       │   ├── use-curve-swap.ts   # Curve quote + swap hook
│   │       │   ├── use-curve-lp.ts    # LP pool stats + liquidity hooks
│   │       │   ├── use-permit2.ts     # Circle/Permit2 hook
│   │       │   └── use-wallet.ts      # Wallet connection hook
│   │       └── lib/
│   │           ├── constants.ts    # Addresses, ABIs
│   │           └── wagmi.ts        # wagmi config (Arc Testnet)
│   └── api-server/             # Express backend
│       └── src/
│           └── routes/
│               └── fx.ts           # /fx/quote, /fx/trade, /fx/rates, /fx/admin/stats
├── lib/
│   ├── api-spec/               # OpenAPI spec + Orval codegen
│   │   └── openapi.yaml
│   └── db/                     # Drizzle schema + migrations
│       └── src/schema.ts
└── scripts/                    # Utility scripts
```

---

## Pages

### Swap (`/`)

- Select sell token (USDC or EURC)
- Enter amount and click **Get Quote** — fetches live `get_dy` from the Curve pool
- Quote expires after 30 seconds with a one-click refresh button
- **Confirm Swap** triggers:
  - (if needed) ERC-20 approval for the Curve pool
  - (if platform fee configured) ERC-20 transfer to fee wallet
  - Curve `exchange` call
- Success card shows exact amounts, platform fee taken, and Arcscan link

### Liquidity (`/liquidity`)

- **Pool overview**: live USDC reserve, EURC reserve, virtual price, total LP supply (auto-refreshes every 15s)
- **Your position**: LP balance, share %, estimated underlying tokens
- **Add liquidity tab**: deposit USDC and/or EURC (one-sided OK), preview LP tokens via `calc_token_amount`, step indicator, 0.5% slippage protection
- **Remove liquidity tab**: burn LP tokens to receive balanced (USDC+EURC), USDC-only, or EURC-only; 25/50/75/Max quick-select; preview via `calc_withdraw_one_coin`

### Analytics (`/admin`)

- Platform-wide stats: total trades, total volume, unique wallets
- Daily volume chart (last 30 days)
- USDC→EURC vs EURC→USDC pair breakdown
- Estimated platform fees collected

### History (`/history`)

- Per-wallet trade list (connect wallet to filter, or view all)
- Columns: time, pair, sell amount, buy amount, txHash

---

## API endpoints

All endpoints are prefixed with `/api/fx`.

| Method | Path | Description |
|---|---|---|
| `GET` | `/rates` | Live EUR/USD rate (cached 60s, from open.er-api.com) |
| `POST` | `/quote` | Get a swap quote (amount, pair) |
| `POST` | `/trade` | Record a completed trade |
| `GET` | `/trades` | List trades (optional `?address=` filter) |
| `GET` | `/admin/stats` | Aggregate analytics (volume, fees, daily breakdown) |

Full OpenAPI spec: [`lib/api-spec/openapi.yaml`](lib/api-spec/openapi.yaml)

---

## Platform fee

Configured via environment variables:

```
PLATFORM_FEE_WALLET=0xYourWalletAddress
PLATFORM_FEE_BPS=20   # 0.20%
```

On the **Curve path**: fee is deducted from the sell amount before calling `exchange`, then transferred directly to the fee wallet via ERC-20 `transfer`.

On the **Circle path**: fee is transferred via Permit2 witness alongside the main swap.

---

## Notes & gotchas

- Curve `get_dy` args are `int128` — viem encodes positive bigints correctly for this
- LP tokens are 18 decimals; pool reserves (USDC/EURC) are 6 decimals
- The pool is currently imbalanced (more USDC than EURC) — EURC deposits may receive a small bonus; single-sided USDC withdrawals may incur a small penalty
- Circle RFQ path requires a `stablefx_api_key` stored in `localStorage` — leave it empty to use Curve-only mode
- `get_dy` reverts if called with `0n` args via raw hex encoding — always call through viem

---

## License

MIT — see [LICENSE](LICENSE) for details.

> **Not for production use.** This dApp runs on Arc Testnet only. Do not send mainnet funds.
