# Liquidity Management

The `/liquidity` page lets users deposit into and withdraw from the Curve USDC/EURC StableSwap pool directly from the StableFX interface.

---

## Pool overview panel

Always visible (no wallet required). Auto-refreshes every 15 seconds via `useReadContracts`.

| Field | Source | Notes |
|---|---|---|
| USDC in pool | `balances(0)` | 6 decimals |
| EURC in pool | `balances(1)` | 6 decimals |
| Virtual price | `get_virtual_price()` | 18 decimals, starts at 1.0 and grows as fees accrue |
| Total LP supply | `totalSupply()` | 18 decimals |

---

## Your position panel

Visible when a wallet is connected on Arc Testnet.

| Field | Calculation |
|---|---|
| LP balance | `balanceOf(userAddress)` on the pool contract |
| Pool share % | `userLP / totalLP * 100` |
| Estimated USDC | `usdcReserve * sharePercent / 100` |
| Estimated EURC | `eurcReserve * sharePercent / 100` |

---

## Add liquidity

### Inputs
- USDC amount (optional)
- EURC amount (optional)
- At least one must be non-zero

### Preview
`calc_token_amount([usdcAmt, eurcAmt], true)` is called with a 400ms debounce to show estimated LP tokens.

### Transaction steps

```
1. Check USDC allowance → if insufficient: approve(CURVE_POOL, usdcAmt)
2. Check EURC allowance → if insufficient: approve(CURVE_POOL, eurcAmt)
3. add_liquidity([usdcAmt, eurcAmt], minLpOut)
```

`minLpOut = previewedLP * (10000 - 50) / 10000`  (0.5% slippage tolerance)

### One-sided deposits
Curve StableSwap supports depositing only one token. If the pool is imbalanced, depositing the underweighted token (currently EURC) gives a small bonus due to the pool rebalancing effect.

---

## Remove liquidity

### Withdrawal modes

| Mode | Contract call | Use case |
|---|---|---|
| Balanced | `remove_liquidity(lpAmt, [minUsdc, minEurc])` | Receive both tokens proportionally |
| USDC only | `remove_liquidity_one_coin(lpAmt, 0, minOut)` | Receive only USDC |
| EURC only | `remove_liquidity_one_coin(lpAmt, 1, minOut)` | Receive only EURC |

### Preview
- **Balanced**: estimated from `(lpAmt / totalSupply) * reserve` (approximate, no chain call)
- **Single-coin**: `calc_withdraw_one_coin(lpAmt, coinIndex)` with 400ms debounce

### Quick-select buttons
25% / 50% / 75% / Max buttons calculate `userLpBalance * pct / 100`.

### Slippage
All outputs are protected with 0.5% slippage: `minOut = preview * 9950 / 10000`.

---

## LP token economics

- LP tokens are minted by the Curve pool itself (same contract address)
- LP tokens are 18 decimals regardless of the underlying token decimals
- The virtual price starts at 1.0 and monotonically increases as swap fees (0.01% per swap) are collected
- There are no additional staking or reward contracts on Arc Testnet — fee income is purely via virtual price appreciation

