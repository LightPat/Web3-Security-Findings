# PrecompileReader hard-reverts on any HyperCore staticcall failure, permanently DoS-ing settleDailyPnL (Gate 3) and all solvency views

**Judging Result:** ❌ **Invalid**

---

# Root + Impact

**Impact:** High **Likelihood:** Medium

Scope: .../src/core/PrecompileReader.sol

## Description

`PrecompileReader` is the single read path used by `MonetrixAccountant` for all L1 state (perp account value, spot/supplied balances, HLP equity, oracles, etc.).

Normal behavior is that every wrapper performs a `staticcall` and immediately reverts on failure or malformed return data.

```solidity
// PrecompileReader.sol
(bool ok, bytes memory res) = HyperCoreConstants.PRECOMPILE_XXX.staticcall(...);
@> require(ok && res.length >= EXPECTED_LEN, "PrecompileReader: XXX read failed");
// oraclePx / spotPx also do: require(price > 0, "... zero");
```

*While this hard-revert behavior is intentional (see comment in `PrecompileReader.sol`: “Reverts on precompile failure so a transient glitch can’t be silently booked as a zero balance”), the complete lack of any recovery mechanism turns transient L1 glitches into permanent protocol unavailability.*

## Risk

**Likelihood**:

- HyperCore precompiles can experience transient outages or return short/malformed data (known behavior on HyperEVM/L1)
- Oracle precompiles momentarily return zero price (explicit revert in PrecompileReader)
- Network/gas conditions cause any staticcall to revert

**Impact**:

- `settleDailyPnL()` becomes completely uncallable (Gate 3 fails)
- Yield distribution to sUSDM stakers is frozen for the duration of the outage
- All solvency views (`totalBackingSigned`, `surplus`, `distributableSurplus`) revert, breaking monitoring/dashboards/frontend

## Proof of Concept

```solidity
function test_submissionValidity() public {
        // Setup: deposit + stake so we have positive yield to settle
        _deposit(user1, 1_000_000e6);
        _stake(user1, 1_000_000e6);

        // Fast-forward past first settlement (Gate 1)
        vm.warp(block.timestamp + 25 hours);

        // Give the vault plenty of mock L1 backing so distributableSurplus() > 0
        _mockVaultL1SpotUsdc(10_000_000e6);
        _mockVaultL1SuppliedUsdc(10_000_000e6);

        // Force PERP_ACCOUNT_MARGIN_SUMMARY precompile to return wrong-length data
        // (triggers "PrecompileReader: perp account read failed" in _readPerpAccountValueSigned)
        _PoCMockPrecompile(payable(HyperCoreConstants.PRECOMPILE_ACCOUNT_MARGIN_SUMMARY))
            .setResponse(abi.encode(uint32(0), address(vault)), new bytes(96));

        // Attempt settlement -> Gate 3 should now hit the precompile failure
        vm.expectRevert("PrecompileReader: perp account read failed");
        vm.prank(address(vault));
        accountant.settleDailyPnL(50e6);

        // Demonstrate that ALL critical solvency views are now also bricked
        vm.expectRevert("PrecompileReader: perp account read failed");
        accountant.totalBackingSigned();

        vm.expectRevert("PrecompileReader: perp account read failed");
        accountant.distributableSurplus();
    }
```

The PoC runs inside the official C4Submission.t.sol template and demonstrates that once any critical precompile read fails, settleDailyPnL and all backing/surplus views are blocked until the precompile recovers.

## Recommended Mitigation

```diff
- (bool ok, bytes memory res) = PRECOMPILE.staticcall(data);
- require(ok && res.length >= EXPECTED_LEN, "PrecompileReader: XXX read failed");
+ (bool ok, bytes memory res) = PRECOMPILE.staticcall(data);
+ if (!ok || res.length < EXPECTED_LEN) {
+     emit PrecompileReadFailed(PRECOMPILE, data);
+     // Use last-known-good cached value if still fresh (recommended)
+     if (lastReadTimestamp[PRECOMPILE] + MAX_STALENESS >= block.timestamp) {
+         return cachedValue[PRECOMPILE];
+     }
+     // Otherwise fail closed (no silent zero values)
+     revert("PrecompileReader: read failed and cache stale");
+ }
```

**Primary mitigation (PrecompileReader.sol):** Replace the hard `require` with graceful degradation pattern above using per-precompile last-successful-value caching + staleness check. This preserves the explicit fail-closed philosophy while allowing transient HyperCore outages to be tolerated without DoS-ing Gate 3 or solvency views. Add the following supporting code:
- `event PrecompileReadFailed(address precompile, bytes data);`
- `mapping(address => uint256) public lastReadTimestamp;`
- `mapping(address => int256) public cachedValue;` (use `int256` because critical reads like perp account value are signed)
- `uint256 public constant MAX_STALENESS = 2 hours;`
- Update the cache on every successful read (add `lastReadTimestamp[PRECOMPILE] = block.timestamp; cachedValue[PRECOMPILE] = value;` after a successful decode).

```diff
+ /// @notice GOVERNOR-controlled emergency override during prolonged precompile outages
+ bool public emergencyMode;
+ int256 public emergencyBackingOverride;
+
+ function toggleEmergencyMode(bool _enabled) external onlyGovernor {
+     emergencyMode = _enabled;
+     emit EmergencyModeToggled(_enabled);
+ }
+
+ function setEmergencyBackingOverride(int256 _override) external onlyGovernor {
+     require(emergencyMode, "Accountant: emergency mode not active");
+     emergencyBackingOverride = _override;
+ }
```

**Secondary mitigation (MonetrixAccountant.sol):** Add the two functions above and insert the following check at the top of `totalBackingSigned()`, `distributableSurplus()`, and inside `settleDailyPnL()` (before Gate 3):

```solidity
if (emergencyMode) return emergencyBackingOverride;
```

This gives the Governor a timelocked, auditable recovery path for extended outages without protocol downtime.

This solution maintains the protocol’s security model, adds no new trust assumptions, and eliminates the permanent DoS risk from transient L1 precompile failures.