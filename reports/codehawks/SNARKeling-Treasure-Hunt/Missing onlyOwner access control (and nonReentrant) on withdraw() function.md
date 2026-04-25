# Missing onlyOwner access control (and nonReentrant) on withdraw() function

**Judging Result:** ✅ **Valid**

---

# Root + Impact

**Impact:** Low **Likelihood:** Medium

Scope: .../src/TreasureHunt.sol

## Description

* The withdraw() function is intended to let the owner retrieve any leftover funds after the hunt has officially ended (i.e. once claimsCount >= MAX\_TREASURES).

* The function is missing the onlyOwner modifier (which already exists in the contract) and the nonReentrant modifier. Any address on the network can call withdraw() as soon as the hunt ends, draining the entire contract balance to the owner address.

```Solidity
function withdraw() external {
    require(claimsCount >= MAX_TREASURES, "HUNT_NOT_OVER");     

    uint256 balance = address(this).balance;
    require(balance > 0, "NO_FUNDS_TO_WITHDRAW");
@>  (bool sent, ) = owner.call{value: balance}("");  // anyone can trigger this
    require(sent, "ETH_TRANSFER_FAILED");

    emit Withdrawn(balance, address(this).balance);
}   
```

## Risk

**Likelihood**:

* The hunt will end once MAX\_TREASURES (10) are claimed.

* Any user (or bot) can monitor the contract state and call withdraw() the moment claimsCount >= MAX\_TREASURES.

**Impact**:

* Funds are still sent to the legitimate owner (no direct theft).

* Griefing / nuisance attack possible (anyone can force the withdrawal).

* Inconsistent with the rest of the admin functions (pause, fund, unpause, etc.) that properly restrict the caller.

* Theoretical reentrancy vector because ETH is sent via low-level call without nonReentrant.

## Proof of Concept

```Solidity
// After all 10 treasures have been successfully claimed:
address anyone = makeAddr("anyone");
vm.prank(anyone);
treasureHunt.withdraw();   // succeeds and sends entire balance to owner
```
Any address can call the withdraw() function once claimsCount >= MAX_TREASURES, even though it should be restricted to the owner only. This demonstrates the lack of access control. The transaction succeeds for any caller, allowing anyone to trigger the withdrawal as soon as the hunt ends.

## Recommended Mitigation

```diff
- function withdraw() external {
+ function withdraw() external onlyOwner nonReentrant {
    require(claimsCount >= MAX_TREASURES, "HUNT_NOT_OVER");     

    uint256 balance = address(this).balance;
    require(balance > 0, "NO_FUNDS_TO_WITHDRAW");
    (bool sent, ) = owner.call{value: balance}("");
    require(sent, "ETH_TRANSFER_FAILED");

    emit Withdrawn(balance, address(this).balance);
}   
```
The onlyOwner modifier (already defined in the contract) ensures only the legitimate owner can call withdraw(). The nonReentrant modifier (already implemented and used in claim()) prevents any potential reentrancy attacks during the Ether transfer.
