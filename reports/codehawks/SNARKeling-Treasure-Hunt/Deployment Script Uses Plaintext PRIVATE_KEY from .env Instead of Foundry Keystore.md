# Deployment Script Uses Plaintext PRIVATE_KEY from .env Instead of Foundry Keystore

**Judging Result:** ✅ **Valid**

---

# Root + Impact

**Impact:** High **Likelihood:** Medium

Scope: .../scripts/Deploy.s.sol

## Description

* The normal behavior is that Deploy.s.sol deploys the HonkVerifier and TreasureHunt contracts by broadcasting the deployment transactions from the deployer account using vm.startBroadcast.

* The specific issue is that the script loads the deployer’s private key via vm.envUint("PRIVATE\_KEY"). This forces users to store their private key in plaintext in a .env file - a severe security anti-pattern. Private keys can never be rotated like API keys, and any exposure (AI agent reading the file, accidental git commit, etc.) permanently compromises the entire wallet.

```Solidity
// Root cause in the codebase with @> marks to highlight the relevant section
contract Deploy is Script {

    function run() external {
        
@>      uint256 deployerKey = vm.envUint("PRIVATE_KEY"); // Shouldn't be reading private keys from a .env file
        uint256 initialFunding = vm.envOr("INITIAL_FUNDING", DEFAULT_INITIAL_FUNDING);

        address deployer = vm.addr(deployerKey);

        console2.log("Chain ID:", block.chainid);
        console2.log("Deployer:", deployer);

        vm.startBroadcast(deployerKey); // This needs to be changed as well

        verifier = new HonkVerifier();
        hunt = new TreasureHunt{value: initialFunding}(address(verifier));

        vm.stopBroadcast();
```

## Risk

**Likelihood**:

* Developers follow the script exactly as written (and as documented in the code comments), which requires creating a .env file with PRIVATE\_KEY=0x…

* .env files are frequently accidentally committed, shared, or read by AI agents/tools that scan project files

**Impact**:

* Complete and permanent compromise of the deployer wallet (private keys cannot be rotated)

* Full loss of any funds or assets controlled by that address if the key is ever exposed

## Proof of Concept

```Solidity
// No on-chain PoC required - the vulnerability is in the deployment script itself.
// 1. Create .env with PRIVATE_KEY=0x<your_key>
// 2. Run: forge script Deploy.s.sol --rpc-url <url> --broadcast
// Any AI agent or process with filesystem access can now read the plaintext key.
```

## Recommended Mitigation

```diff
 contract Deploy is Script {

     function run() external {
-        uint256 deployerKey = vm.envUint("PRIVATE_KEY");
+        // Use Foundry's encrypted keystore (cast wallet) instead of plaintext .env
+        // Run script with: forge script Deploy.s.sol --rpc-url <url> --account <keystore-name> --sender <sender-address> --broadcast

         uint256 initialFunding = vm.envOr("INITIAL_FUNDING", DEFAULT_INITIAL_FUNDING);

-        address deployer = vm.addr(deployerKey);
+        // Deployer address is now msg.sender when using --sender flag

         console2.log("Chain ID:", block.chainid);
-        console2.log("Deployer:", deployer);
+        console2.log("Deployer:", msg.sender);

-        vm.startBroadcast(deployerKey);
+        vm.startBroadcast();  // No private key argument when using --account

         verifier = new HonkVerifier();
         hunt = new TreasureHunt{value: initialFunding}(address(verifier));

         vm.stopBroadcast();
```

To run this script. Use:

```bash
# Secure keystore setup (recommended by Cyfrin/Foundry)
cast wallet import deployer --interactive
forge script Deploy.s.sol --rpc-url <url> --account deployer --sender deployer_address --broadcast
```

This fully eliminates plaintext private key storage and aligns with the exact security practices taught in Cyfrin Updraft courses.
Don't forget to update your README as well.

Note: The reason we pass --sender and --account is because --account is used for signing the transaction. Without --sender foundry will default back to the default address so startBroadcast() won't be sent from the correct address that you want.
