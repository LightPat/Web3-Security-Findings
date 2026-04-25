# Duplicate hash in ALLOWED_TREASURE_HASHES prevents treasure #9 from ever being claimed

**Judging Result:** ✅ **Valid**

---

# Root + Impact

**Impact:** Medium **Likelihood:** High

Scope: .../src/main.nr

## Description

* The Noir circuit is designed to let a finder prove knowledge of any one of the 10 intended treasures without revealing which one it is. It does this by checking that the public `treasure_hash` is present in the hardcoded `ALLOWED_TREASURE_HASHES` array **and** that it equals `pedersen_hash(treasure)`.

* The last two entries in `ALLOWED_TREASURE_HASHES` are identical. The array therefore only contains 9 unique hashes instead of the documented 10. Treasure #9’s hash is missing, so no valid ZK proof can ever be generated for it.

```Solidity
// Baked-in set of 10 allowed treasure hashes (pedersen hashes).
global ALLOWED_TREASURE_HASHES: [Field; 10] = [
    1505662313093145631275418581390771847921541863527840230091007112166041775502,
    -7876059170207639417138377068663245559360606207000570753582208706879316183353,
    -5602859741022561807370900516277986970516538128871954257532197637239594541050,
    2256689276847399345359792277406644462014723416398290212952821205940959307205,
    10311210168613568792124008431580767227982446451742366771285792060556636004770,
    -5697637861416433807484703347699404695743570043365849280798663758395067508,
    -2009295789879562882359281321158573810642695913475210803991480097462832104806,
    8931814952839857299896840311953754931787080333405300398787637512717059406908,
@>  -961435057317293580094826482786572873533235701183329831124091847635547871092,
@>  -961435057317293580094826482786572873533235701183329831124091847635547871092 // duplicate of previous entry
];
```

## Risk

**Likelihood**:

* The array is statically defined in the circuit source and is baked into the deployed verifier contract.

* The is\_allowed helper and the main function both rely on this exact array; the duplicate is present in every build and every deployment.

**Impact**:

* Treasure #9 becomes permanently unclaimable, even a finder who physically locates it and knows the correct secret cannot generate a valid proof.

* The real-world treasure hunt is left with only 9 working treasures instead of the intended 10, breaking the game design and fairness guarantees.

## Proof of Concept

```Solidity
// tests.nr explicitly skips treasure 9 and duplicates treasure 10:
let treasures = [1, 2, 3, 4, 5, 6, 7, 8, 10, 10];
for i in 0..10 {
    let treasure = treasures[i];
    let treasure_hash = ALLOWED_TREASURE_HASHES[i];
    main(treasure, treasure_hash, 2);   // passes only because test avoids #9
}
```

Simply inspecting the array shows ALLOWED\_TREASURE\_HASHES\[8] == ALLOWED\_TREASURE\_HASHES\[9]. Any attempt to claim treasure #9 with its real secret will fail the assert(is\_allowed(treasure\_hash)) check in main.

## Recommended Mitigation

```diff
 global ALLOWED_TREASURE_HASHES: [Field; 10] = [
     1505662313093145631275418581390771847921541863527840230091007112166041775502,
     ...,
     8931814952839857299896840311953754931787080333405300398787637512717059406908,
-    -961435057317293580094826482786572873533235701183329831124091847635547871092,
-    -961435057317293580094826482786572873533235701183329831124091847635547871092 
+    -961435057317293580094826482786572873533235701183329831124091847635547871092, // treasure #9
+    <correct_pedersen_hash_for_treasure_10>                                       // replace with the real hash
 ];

 // (or if only 9 treasures were ever intended)
- global ALLOWED_TREASURE_HASHES: [Field; 10] = [
+ global ALLOWED_TREASURE_HASHES: [Field; 9] = [
     ...
- for i in 0..10 {
+ for i in 0..9 {
```

Update the array size, loop bounds, all comments that say “10 allowed hashes / 10 treasures”, and the test suite accordingly.
