# Axelar Fee Economics

## Scope and Modeling Boundaries

This note documents fee economics based on `axelar-core` code, plus explicitly marked assumptions from ecosystem-level flow descriptions.

- **In-scope (code-confirmed):** fee handling on Axelar chain (`x/distribution`), including community-pool allocation and burn.
- **Out-of-scope in this repo:** off-chain or cross-chain swap/conversion mechanics (for example, "Gas Services converts ETH/BNB fees into AXL"), which are not implemented in `axelar-core`.

---

## 1) Code-Confirmed Fee Handling on Axelar Chain

### Execution timing

Fee handling runs in distribution `BeginBlock`, which processes fees collected from the previous block.

- `x/distribution/abci.go` calls `AllocateTokens(...)` in `BeginBlocker`.
- `x/distribution/keeper/keeper.go` reads fee collector balances and applies custom allocation logic.

### Allocation rule

Let:

- $F_t$: total fees collected (processed at block $t$)
- $\phi_t^{\text{fee}}$: community tax rate (`communityTaxRate`)
- $C_t$: community-pool allocation
- $B_t$: burned amount

Then:

$$
C_t = \phi_t^{\text{fee}} \cdot F_t
$$

$$
B_t = F_t - C_t
$$

Implementation details:

1. Fees are moved from fee collector module account to distribution module account.
2. Community allocation is computed as `feesCollected * communityTaxRate`.
3. Decimal truncation remainder is added to community pool.
4. Remaining fees are burned with `BurnCoins`.
5. `FeesBurned` event is emitted.
6. A bookkeeping token `burned-<denom>` is minted and sent to `ZeroAddress` to track cumulative burned amounts.

### Validator impact

Validators do **not** receive transaction fees through this custom distribution path.  
Inflationary validator rewards are handled separately in the reward module and released via reward pools.

---

## 2) Fee Denomination and Origin

### What the code confirms

- The fee logic is denom-agnostic at this step: it processes whatever denoms are in fee collector balances.
- Burn/community split is applied to collected on-chain fee balances, not hardcoded to a single denom inside `AllocateTokens`.

### Practical interpretation for modeling

For tokenomics modeling, use two layers:

1. **Upstream fee origin layer (external to this repo):**
   - Users may pay fees in source-chain assets (for example ETH, BNB, etc.).
   - External systems may convert those fees into Axelar-chain fee denomination(s).
2. **Axelar-chain distribution layer (code-confirmed here):**
   - On collected Axelar-chain fees, apply:
    - community allocation $C_t$,
    - burn $B_t$.

If your external assumption is "fees become AXL on Axelar chain before distribution", then the effective economics are:

- $2\%$ to community pool (if $\phi_t^{\text{fee}}=2\%$),
- $98\%$ AXL burn,
- no validator share from tx fees.

---

## 3) Draft End-to-End Narrative (for documentation)

Use this as a copy-paste paragraph if you want the bridge-transaction intuition:

> Users initiate cross-chain actions and pay fees in source-chain tokens (for example ETH or BNB).  
> In the broader Axelar flow, those fees are converted into Axelar-chain fee balances (model assumption if conversion is handled outside `axelar-core`).  
> Once fees are in Axelar-chain balances, `x/distribution` applies the on-chain rule: a community-tax slice goes to the community pool, and the remainder is burned.  
> With a 2% tax parameter, this yields a 2% community allocation and 98% burn per processed fee flow, which contributes to recurring buy pressure (via conversion demand) plus supply destruction (via burn).

---

## 4) Recommended Modeling Equations

### Chain-internal (code-confirmed) layer

$$
C_t = \phi_t^{\text{fee}} \cdot F_t,\quad
B_t = (1-\phi_t^{\text{fee}})\cdot F_t
$$

### Optional upstream conversion layer (assumption)

If source-chain fees are converted to AXL:

- $Q_t^{\text{src}}$: source-chain fee quantity
- $P_t$: conversion rate into AXL
- $F_t^{\text{AXL}} = Q_t^{\text{src}} \cdot P_t$

Then apply:

$$
C_t^{\text{AXL}} = \phi_t^{\text{fee}} \cdot F_t^{\text{AXL}},\quad
B_t^{\text{AXL}} = (1-\phi_t^{\text{fee}})\cdot F_t^{\text{AXL}}
$$

---

## 5) Contradiction Note (Spreadsheet vs Code)

If a spreadsheet applies the community-tax factor to inflationary validator block rewards, that conflicts with the current `axelar-core` implementation.

- Community tax in this path applies to **transaction fees**.
- Inflationary rewards are released through reward pools to validators without this deduction path.

---

## 6) References (Code Paths)

- `x/distribution/abci.go`
- `x/distribution/keeper/keeper.go`
- `x/distribution/keeper/keeper_test.go`
- `x/distribution/types/types.go`
- `x/reward/keeper/reward_pool.go`
