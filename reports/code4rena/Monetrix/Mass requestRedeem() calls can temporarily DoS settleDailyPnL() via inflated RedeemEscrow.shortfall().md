# Mass requestRedeem() calls can temporarily DoS settleDailyPnL() via inflated RedeemEscrow.shortfall()

**Judging Result:** ❌ **Not Judged Yet**

---

# Root + Impact

**Impact:** Medium **Likelihood:** High

Scope: .../src/core/RedeemEscrow.sol, .../src/core/MonetrixAccountant.sol

## Description

Normal user redemptions work as follows: any USDM holder calls `requestRedeem()` on `MonetrixVault`. The USDM is transferred to the Vault and RedeemEscrow.addObligation() is called immediately, increasing totalOwed. The Operator is later expected to fund the escrow with USDC so users can claim after the cooldown period.

*Note that `requestRedeem()` does not burn the USDM. Burning only occurs in `claimRedeem()` after the cooldown. Thus `usdm.totalSupply()` (used in `surplus()`) remains unchanged while `totalOwed` immediately spikes.*

This design choice creates an immediate spike in `totalOwed` (and thus `shortfall()`) without any corresponding increase in escrow balance or change in overall backing health.

The Accountant uses this `shortfall()` value (which can be arbitrarily large) when calculating `distributableSurplus()`:

```solidity
// MonetrixAccountant.sol
function distributableSurplus() public view returns (int256) {
    int256 s = surplus();
    address re = IMonetrixVaultReader(vault).redeemEscrow();
@>  uint256 sf = re == address(0) ? 0 : IRedeemEscrow(re).shortfall();
    return s - int256(sf);
}
```

`settleDailyPnL` (the only entrypoint for yield distribution) then enforces Gate 3:

```solidity
// MonetrixAccountant.sol
function settleDailyPnL(uint256 proposedYield) external onlyVault {
    // ... Gate 1 & 2 ...
    int256 ds = distributableSurplus();
@>  require(ds > 0, "Accountant: no distributable surplus"); // Gate 3
    // ...
}
```

Because `addObligation` fires on every `requestRedeem` (before any USDC reaches the escrow), a wave of redemptions can push `shortfall()` above `surplus()` even when the overall backing is healthy. This turns `distributableSurplus()` non-positive and blocks yield distribution for the full cooldown period (or longer if the Operator delays the top-up)

```solidity
// RedeemEscrow.sol
function addObligation(uint256 amount) external onlyVault {
@>  totalOwed += amount; // immediate on user requestRedeem
    emit ObligationAdded(amount, totalOwed);
}

function shortfall() external view returns (uint256) {
    uint256 bal = usdc.balanceOf(address(this));
@>  return totalOwed > bal ? totalOwed - bal : 0;
}
```

## Risk

**Likelihood**:

- Any USDM holder (or coordinated group) can call `requestRedeem` at any time with no extra restrictions.
- During market stress, FUD, or normal large redemptions, a significant portion of the USDM supply can be queued in a short window.

**Impact**:

- All yield distribution to sUSDM holders is frozen (no rate growth via `injectYield`).
- The freeze can last days (full 3-day cooldown) or indefinitely if the Operator is slow to top up.
- Breaks the core promise of continuous yield accrual and creates a user-triggered availability issue that is not covered by the known Operator-inaction issues.
- Affects *all* sUSDM holders by freezing yield accrual across the entire staked supply.

*This griefing vector is distinct from the publicly disclosed "Operator inaction" risks, as it can be triggered by any permissionless user(s) performing normal redemptions.*

## Proof of Concept

```solidity
function test_submissionValidity() public {
    // 1. Establish healthy state with positive surplus (simulating accrued yield)
    uint256 depositPerUser = 800_000e6; // 800k USDC - safely under maxDepositAmount = 1M

    _deposit(user1, depositPerUser);
    _deposit(user2, depositPerUser);

    // Fast-forward past minSettlementInterval so Gate 1 passes
    vm.warp(block.timestamp + 25 hours);

    // Mock L1 backing high enough for positive surplus, but low enough that mass redemptions exceed it
    _mockVaultL1SpotUsdc(1_200_000_00000000); // 1.2M USDC spot (8-decimal HL internal)
    _mockVaultL1SuppliedUsdc(700_000_00000000); // 700k USDC supplied

    int256 surplusBefore = accountant.surplus();
    require(surplusBefore > 0, "Precondition: positive surplus required");

    // 2. Mass user-triggered redemptions (perfectly normal, permissionless)
    // Each call immediately does addObligation() -> totalOwed spikes
    _requestRedeem(user1, depositPerUser);
    _requestRedeem(user2, depositPerUser);

    // 3. Verify the broken state
    uint256 currentShortfall = redeemEscrow.shortfall();
    int256 surplusAfter = accountant.surplus();
    int256 distributable = accountant.distributableSurplus();

    assertGt(currentShortfall, uint256(surplusAfter), "shortfall() now exceeds surplus");
    assertLe(distributable, 0, "distributableSurplus() is now non-positive");

    // 4. Prove Gate 3 is blocked (yield distribution frozen)
    vm.prank(address(vault));
    vm.expectRevert("Accountant: no distributable surplus");
    accountant.settleDailyPnL(100_000e6); // any positive yield is now impossible
}
```

The test reproduces a realistic redemption wave. After the calls, `shortfall()` jumps because `totalOwed` increased immediately while the RedeemEscrow balance did not. `distributableSurplus()` turns non-positive even though overall peg health (`surplus()`) remains fine. Yield settlement stays blocked until either:
- (a) users finish the cooldown and `claimRedeem()` (reducing `totalOwed`)
- (b) the Operator calls `fundRedemptions()` to top up the escrow. In a stress event this creates a freeze lasting up to the full `redeemCooldown()` period (currently 3 days).

## Recommended Mitigation

Only consider *claimable* obligations (those past the cooldown period) when calculating the shortfall that affects yield distribution. This keeps the peg-protection intent for immediate claims while preventing in-cooldown redemptions from blocking daily yield settlement. 
Alternatively (and preferably), decouple the temporary redemption accounting spike from the yield gate entirely.

### Option 1

```diff
// RedeemEscrow.sol
function shortfall() external view returns (uint256) {
    uint256 bal = usdc.balanceOf(address(this));
-   return totalOwed > bal ? totalOwed - bal : 0;
+   return claimableOwed > bal ? claimableOwed - bal : 0; // only matured (post-cooldown) obligations
}
```

**Add a new storage variable and update logic** (example):

```solidity
uint256 public claimableOwed; // new

// Called lazily on claimRedeem (or via a keeper cron)
function _matureObligation(uint256 amount) internal {
    claimableOwed += amount;
}

// In payout / claim flow, also reduce claimableOwed
```

**Note:** This requires more changes than shown (cooldown logic lives in `MonetrixVault.redeemRequests`, not in `RedeemEscrow`), plus new storage and upgrade considerations.

### Option 2

**OR (cleaner decoupling alternative - my preferred option)**

```diff
// MonetrixAccountant.sol
function distributableSurplus() public view returns (int256) {
    int256 s = surplus();
    address re = IMonetrixVaultReader(vault).redeemEscrow();
    uint256 sf = re == address(0) ? 0 : IRedeemEscrow(re).shortfall();
-   return s - int256(sf);
+   if (s > 0) {
+       return s; // solvent overall → ignore temporary in-cooldown shortfall for yield gate
+   }
+   return s - int256(sf); // preserve hard insolvency signal
}
```

The root problem is that `shortfall()` (and by extension `distributableSurplus()`) treats **pending redemption requests still in their cooldown period** as immediately claimable obligations.

- **Option 1 (targeted):** Track `claimableOwed` separately and only let matured obligations affect the yield gate. This preserves the original peg-protection intent for users who can actually claim today.
- **Option 2 (simpler, my recommendation):** Decouple the redemption safety check from the yield gate entirely. Yield distribution is only blocked when the protocol is actually insolvent (`surplus() < 0`). This eliminates the griefing vector while keeping the Accountant’s core safety guarantees.

> Option 2 is the cleanest and least invasive change for v1. It eliminates the griefing vector while maintaining the core insolvency protection of the Accountant (no new storage, no extra gas on the hot path).

Either change removes the user-triggered DoS without introducing new trust assumptions or complexity. Option 2 is especially clean because `surplus()` already includes `RedeemEscrow` balance and all L1 assets.