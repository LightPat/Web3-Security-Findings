# Incorrect Recipient in Claimed Event

**Judging Result:** ✅ **Valid**

---

# Root + Impact

**Impact:** Low **Likelihood:** High

Scope: .../src/TreasureHunt.sol

## Description

* The claim function allows a user to submit a valid ZK proof along with a recipient address. After successful verification, the contract transfers the REWARD (10 ETH) to that recipient and is supposed to emit a Claimed event that logs the treasureHash and the actual recipient of the funds.

* The Claimed event is emitted with msg.sender (the caller of the function) instead of the recipient parameter that actually receives the ETH.

```Solidity
event Claimed(bytes32 indexed treasureHash, address indexed recipient); // intended to record the real recipient

function claim(bytes calldata proof, bytes32 treasureHash, address payable recipient) external nonReentrant() {
    // ... validations & proof verification ...

    (bool sent, ) = recipient.call{value: REWARD}("");
    require(sent, "ETH_TRANSFER_FAILED");

@>  emit Claimed(treasureHash, msg.sender); // BUG: should be recipient
}
```

## Risk

**Likelihood**:

* This bug triggers on every successful claim() call

* No special conditions or attacker actions required. It is deterministic

**Impact**:

* Off-chain indexing/monitoring will track the wrong recipient

* Audit logs and historical data will be misleading

* Event consumers (block explorers, dashboards, bots, frontends) cannot determine who actually received the reward

## Proof of Concept

```Solidity
// Any successful call to claim() demonstrates the bug:
address caller = makeAddr("caller");
address recipient = makeAddr("recipient");

// caller != recipient
treasureHunt.claim(proof, treasureHash, payable(recipient));

// Emitted event will be: Claimed(treasureHash, caller)
// Correct event should be: Claimed(treasureHash, recipient)
```
The PoC creates a situation where the caller (msg.sender) is different from the recipient who actually receives the ETH. The contract correctly sends the funds to the recipient, but the emitted Claimed event incorrectly logs the caller’s address. This shows the event is unreliable for off-chain tracking.

## Recommended Mitigation

```diff
function claim(bytes calldata proof, bytes32 treasureHash, address payable recipient) external nonReentrant() {
    // ... validations & proof verification ...

    (bool sent, ) = recipient.call{value: REWARD}("");
    require(sent, "ETH_TRANSFER_FAILED");

-   emit Claimed(treasureHash, msg.sender);
+   emit Claimed(treasureHash, recipient);
}
```
Changing the event emission from msg.sender to the recipient parameter is a one-line fix. It ensures the Claimed event always accurately reflects who actually received the reward. This has zero risk to the contract logic, negligible gas impact, and fully resolves the off-chain indexing and logging issues.