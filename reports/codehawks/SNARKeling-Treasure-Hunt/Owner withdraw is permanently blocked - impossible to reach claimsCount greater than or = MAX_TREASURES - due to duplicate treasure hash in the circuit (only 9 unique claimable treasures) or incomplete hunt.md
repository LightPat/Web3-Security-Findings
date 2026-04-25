# Owner withdraw() is permanently blocked (impossible to reach claimsCount >= MAX_TREASURES) due to duplicate treasure hash in the circuit (only 9 unique claimable treasures) or incomplete hunt

**Judging Result:** ✅ **Valid**

---

# Root + Impact

**Impact:** High **Likelihood:** High

Scope: .../src/TreasureHunt.sol

## Description

* Normally, after the treasure hunt concludes, the owner is expected to call `withdraw()` to retrieve any leftover/unclaimed ETH once `claimsCount >= MAX_TREASURES` (10), at which point the hunt is considered "over".

* The specific issue is that `withdraw()` has a hard-coded dependency on **all 10 treasures being successfully claimed**. However, the Noir circuit hard-codes `ALLOWED_TREASURE_HASHES` with a duplicate entry (the last two hashes are identical), so at most 9 unique treasures can ever be claimed. Even without the duplicate, a real-world treasure hunt may naturally result in fewer than 10 claims (not every treasure is found). In either case, `claimsCount` can never reach 10, making the only intended withdrawal path for the owner permanently impossible.

```Solidity
uint256 public constant MAX_TREASURES = 10;

function withdraw() external {
@>  require(claimsCount >= MAX_TREASURES, "HUNT_NOT_OVER");  // this line can never be satisfied
    ...
}
```

(Note: the duplicate is in circuits/src/main.nr, but the vulnerability manifests directly in TreasureHunt.sol.)

## Risk

**Likelihood**:

* The duplicate hash is permanently baked into the deployed circuit, guaranteeing claimsCount can never exceed 9

* Even if the circuit were fixed, treasure hunts are inherently incomplete by nature. Some treasures may simply never be found/claimed

**Impact**:

* Owner cannot recover any remaining ETH (e.g. the unclaimed portion of the initial 100 ETH funding or any extra fund() calls)

* Funds are permanently locked in the contract unless the owner uses the restricted emergencyWithdraw (which requires pausing the entire contract first, halting all future claims)

## Proof of Concept

```Solidity
// After deploying the contract and funding it (e.g. 100 ETH)
// Claim all 9 unique valid treasures (possible because of the duplicate in ALLOWED_TREASURE_HASHES)
assert(treasureHunt.getClaimsCount() == 9);  // maximum possible

// Attempt normal owner withdrawal
treasureHunt.withdraw();  // reverts with "HUNT_NOT_OVER"

// Even after the physical hunt ends, owner is stuck; emergencyWithdraw requires pause() first
```

Because the ALLOWED\_TREASURE\_HASHES array in the Noir circuit contains a duplicate value (the last two entries are identical), there are only 9 distinct treasures that can ever be successfully claimed. The contract tracks claims via a mapping, so attempting to claim the duplicate hash reverts or is ignored. Even after claiming every possible unique treasure, claimsCount remains stuck at 9. The withdraw() function strictly requires claimsCount >= 10 (which is mathematically impossible), permanently blocking the owner from recovering any leftover ETH through the normal path. This reproduces the bug in any testing environment (Foundry/Hardhat) or on a live deployment.

## Recommended Mitigation

```diff
 function withdraw() external {
-    require(claimsCount >= MAX_TREASURES, "HUNT_NOT_OVER");     
+    // Remove the strict requirement. Owner should always be able to recover remaining funds
+    // (if all 10 are claimed, balance should already be 0 anyway)
 
     uint256 balance = address(this).balance;
     require(balance > 0, "NO_FUNDS_TO_WITHDRAW");
     (bool sent, ) = owner.call{value: balance}("");
     require(sent, "ETH_TRANSFER_FAILED");
 
     emit Withdrawn(balance, address(this).balance);
 }
```

Alternatively, add a time-based unlock (e.g. huntEndTime set in constructor) or make emergencyWithdraw usable without requiring the contract to be paused.
