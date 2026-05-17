# Smart Contracts

All contracts are deployed on **Arc Testnet** (Chain ID 5042002).
RPC: `https://rpc.testnet.arc.network`
Explorer: `https://testnet.arcscan.app`

---

## USDC

| Property | Value |
|---|---|
| Address | `0x3600000000000000000000000000000000000000` |
| Decimals | 6 |
| Standard | ERC-20 |

Circle USD Coin bridged to Arc Testnet. Used as coin[0] in the Curve pool.

---

## EURC

| Property | Value |
|---|---|
| Address | `0x89B50855Aa3bE2F677cD6303Cec089B5F319D72a` |
| Decimals | 6 |
| Standard | ERC-20 |

Circle Euro Coin bridged to Arc Testnet. Used as coin[1] in the Curve pool.

---

## Curve USDC/EURC StableSwap Pool

| Property | Value |
|---|---|
| Address | `0x2D84D79C852f6842AbE0304b70bBaA1506AdD457` |
| Type | Curve StableSwap v1 (plain pool) |
| Amplification (A) | 1000 |
| Swap fee | 0.01% (1 basis point) |
| LP token | Same contract (ERC-20, 18 decimals) |
| coin[0] | USDC |
| coin[1] | EURC |

### Key functions

```solidity
// Read a quote: how much of coin j you get for dx of coin i
function get_dy(int128 i, int128 j, uint256 dx) external view returns (uint256);

// Execute swap
function exchange(int128 i, int128 j, uint256 _dx, uint256 _min_dy) external returns (uint256);

// Preview LP tokens for a deposit
function calc_token_amount(uint256[2] amounts, bool is_deposit) external view returns (uint256);

// Preview tokens out for burning LP tokens (single coin)
function calc_withdraw_one_coin(uint256 _token_amount, int128 i) external view returns (uint256);

// Add liquidity (deposit both or one coin)
function add_liquidity(uint256[2] amounts, uint256 min_mint_amount) external returns (uint256);

// Remove liquidity proportionally
function remove_liquidity(uint256 _amount, uint256[2] min_amounts) external returns (uint256[2]);

// Remove liquidity single coin
function remove_liquidity_one_coin(uint256 _token_amount, int128 i, uint256 _min_amount) external returns (uint256);

// Pool reserves
function balances(uint256 i) external view returns (uint256);

// Accumulated fees reflected in LP value
function get_virtual_price() external view returns (uint256);
```

---

## Permit2

| Property | Value |
|---|---|
| Address | `0x000000000022D473030F116dDEE9F6B43aC78BA3` |
| Source | Uniswap Permit2 (canonical deployment) |

Used in the optional Circle RFQ path for gasless approvals via signed permits.

---

## FxEscrow

| Property | Value |
|---|---|
| Address | `0x43506849D7C04F9138D1A2050bbF3A0c054402dd` |

Circle StableFX escrow contract. Only used when Circle RFQ path is active.

---

## Pool state (as of May 2026)

| Metric | Value |
|---|---|
| USDC reserve | ~3.9M |
| EURC reserve | ~178K |
| Virtual price | ~$1.19 |
| Total LP supply | ~3.4M |

The pool is currently imbalanced (more USDC than EURC). This means:
- Single-sided EURC deposits receive a small bonus
- Single-sided USDC withdrawals may incur a small penalty
- EURC→USDC swaps give marginally better rates than USDC→EURC

