# Double-Spend Protection Broken

**Judging Result:** ✅ **Valid**

---

**Impact:** High **Likelihood:** High

Scope: .../src/TreasureHunt.sol

# Root Cause

The immutable state variable `_treasureHash` is declared but never initialized in the constructor (or anywhere else). The `claim()` function checks `claimed[_treasureHash]` instead of the `treasureHash` parameter that the caller actually supplied.

# Impact

The `AlreadyClaimed` protection is completely bypassed for every call. Any treasure can be claimed multiple times (or the check always fails), allowing users or attackers to drain more `REWARD` funds from the contract than intended.

## Description

* Normal Behavior: The `claim()` function is intended to allow any user to claim a unique treasure/reward by supplying a `treasureHash` (along with a valid proof). Before paying out the `REWARD`, the contract must check the `claimed` mapping using the caller-provided `treasureHash` to ensure that the same treasure has not already been claimed.

* Specific Issue: Because the immutable state variable `_treasureHash` is declared but never initialized, the `AlreadyClaimed` check on line 88 (and the duplicate line inside the function) always reads from `claimed[_treasureHash]` instead of the `treasureHash` parameter that was passed in. This makes the duplicate-claim protection completely ineffective.

```Solidity
// Line 35: Declared but never initialized
bytes32 private immutable _treasureHash;

// Inside claim() function
function claim(bytes calldata proof, bytes32 treasureHash, address payable recipient) external nonReentrant() {
        if (paused) revert ContractPaused();
        if (address(this).balance < REWARD) revert NotEnoughFunds();
        if (recipient == address(0) || recipient == address(this) || recipient == owner || recipient == msg.sender) revert InvalidRecipient();
        if (claimsCount >= MAX_TREASURES) revert AllTreasuresClaimed();
@>      if (claimed[_treasureHash]) revert AlreadyClaimed(treasureHash); // Line 88: Checks wrong slot (uses uninitialized _treasureHash instead of parameter treasureHash)
        if (msg.sender == owner) revert OwnerCannotClaim();
        // ... rest of function
}
```

## Risk

**Likelihood**:

* The bug is present in every deployment of the contract because `_treasureHash` is declared as immutable but is never assigned a value in the constructor or anywhere else in the codebase.

* Every single call to `claim()` executes the broken `AlreadyClaimed` check using the uninitialized `_treasureHash` slot instead of the supplied `treasureHash` parameter.

**Impact**:

* Any user can claim the same treasure (or any treasure) an unlimited number of times.

* The contract will pay out the full `REWARD` amount on every successful call, allowing the entire contract balance to be drained far beyond the intended `MAX_TREASURES` limit.

## Proof of Concept

```Solidity
1. Deploy the contract normally (the constructor never initializes the immutable `_treasureHash`, so it remains `bytes32(0)`).
2. Fund the contract with at least `2 * REWARD` ETH.
3. Obtain a valid proof for any `treasureHash`.
4. Call `claim(validProof, treasureHash, recipient1)` the transaction succeeds and pays out `REWARD`.
5. Call `claim(validProof, treasureHash, recipient2)` a second time using the exact same `treasureHash` and proof - the transaction also succeeds without reverting.

Expected behavior: The second call should revert with `AlreadyClaimed(treasureHash)`.  
Actual behavior: The `AlreadyClaimed` check is completely bypassed, so the same treasure can be claimed an unlimited number of times.
```

This PoC demonstrates that the `AlreadyClaimed` protection is completely ineffective. The check always uses the uninitialized `_treasureHash` (which is always `bytes32(0)`) instead of the caller-supplied `treasureHash`. As a result, the same treasure can be claimed repeatedly, bypassing the intended double-spend protection.

## Recommended Mitigation

```diff
- bytes32 private immutable _treasureHash; // Remove the unused storage variable

- if (claimed[_treasureHash]) revert AlreadyClaimed(treasureHash);
+ if (claimed[treasureHash]) revert AlreadyClaimed(treasureHash);
```

Replace the incorrect check that uses the uninitialized immutable variable with a check against the `treasureHash` parameter supplied by the caller.

There is also a test in TreasureHunt.t.sol called testClaimDoubleSpendReverts that has a bug. On line 144 there is a vm.expectRevert(); call that is commented out. This is wrong because hunt.claim() being called a second time should revert.

```diff
function testClaimDoubleSpendReverts() public {
        (
            bytes memory proof,
            bytes32 treasureHash,
            address payable recipient
        ) = _loadFixture();

        vm.startPrank(participant);
        hunt.claim(proof, treasureHash, recipient);

-       //vm.expectRevert();
+       vm.expectRevert();

        hunt.claim(proof, treasureHash, recipient);
        vm.stopPrank();
    }
```
