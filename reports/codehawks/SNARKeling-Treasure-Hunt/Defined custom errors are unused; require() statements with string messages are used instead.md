# Defined custom errors are unused; require() statements with string messages are used instead

**Judging Result:** ❌ **Invalid**

---

# Root + Impact

**Impact:** Low **Likelihood:** High

Scope: .../src/TreasureHunt.sol

## Description

* The TreasureHunt contract defines a full set of custom errors at the top of the file (including multiple OnlyOwner... errors, HuntNotOver(), NoFundsToWithdraw(), TheContractMustBePaused(), etc.) and even includes an onlyOwner modifier.

* However, none of the custom errors are ever used. All owner-restricted functions and the onlyOwner modifier instead revert with the older, less gas-efficient require(condition, "ERROR\_STRING") pattern.

  solidity

```Solidity
// Root cause in the codebase with @> marks to highlight the relevant section
@> error OnlyOwnerCanFund();        // defined but never used
@> error OnlyOwnerCanPause();       // defined but never used
@> error OnlyOwnerCanUnpause();     // defined but never used
@> error OnlyOwnerCanUpdateVerifier(); // defined but never used
@> error OnlyOwnerCanEmergencyWithdraw(); // defined but never used
@> error HuntNotOver();             // defined but never used
@> error NoFundsToWithdraw();       // defined but never used
// ... many more custom errors defined but unused

modifier onlyOwner() {
    require(msg.sender == owner, "ONLY_OWNER");  // Replace with custom error
    _;
}

function pause() external {
    require(msg.sender == owner, "ONLY_OWNER_CAN_PAUSE");  // string instead of OnlyOwnerCanPause()
    ...
}

function fund() external payable {
    require(msg.sender==owner, "ONLY_OWNER_CAN_FUND");  // string instead of OnlyOwnerCanFund()
    ...
}
// same pattern repeats in unpause(), withdraw(), updateVerifier(), emergencyWithdraw()
```

## Risk

**Likelihood**:

* Every call to any owner-restricted function (fund, pause, unpause, withdraw, updateVerifier, emergencyWithdraw)

* The pattern is used consistently across the entire admin surface of the contract

**Impact**:

* Increased gas costs on every admin transaction (string errors are more expensive than custom errors)

* Larger deployed bytecode size due to embedded string data

* Inconsistent and outdated error handling (the contract already uses custom errors in claim() but not for owner checks)

## Proof of Concept

```Solidity
// No execution PoC needed - this is a static code review finding.
// All custom owner errors and several state-checking errors are defined but remain completely unused.
```

## Recommended Mitigation

```diff
- modifier onlyOwner() {
-     require(msg.sender == owner, "ONLY_OWNER");
-     _;
- }

+ modifier onlyOwner() {
+     if (msg.sender != owner) revert OnlyOwner(); // (add a general OnlyOwner error or reuse a specific one)
+     _;
+ }

 function pause() external {
-     require(msg.sender == owner, "ONLY_OWNER_CAN_PAUSE");
+     if (msg.sender != owner) revert OnlyOwnerCanPause();
     paused = true;
     ...
 }

 function fund() external payable {
-     require(msg.sender==owner, "ONLY_OWNER_CAN_FUND");
+     if (msg.sender != owner) revert OnlyOwnerCanFund();
     ...
 }

 // Apply the same pattern to unpause, withdraw, updateVerifier, emergencyWithdraw, etc.
 // Also replace the other string requires (e.g. "HUNT_NOT_OVER", "NO_FUNDS_TO_WITHDRAW") with their matching custom errors.
```
