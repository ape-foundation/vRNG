# Verifying randomness

The "v" in vRNG is **verifiable**: every fulfilled result can be independently re-derived from on-chain inputs, so neither the protocol nor the requester can fake or bias an outcome. You don't have to trust the event log — you can recompute it.

---

## Read the proof

```solidity
function getProof(bytes32 requestId)
    external view returns (Proof memory proof);

struct Proof {
    bytes32 requestId;
    address requester;
    uint64  effectiveSeq;    // sequence number of the beacon that fed the derivation
    bytes32 beaconValue;     // the beacon value stored at effectiveSeq
    uint64  beaconRound;     // the drand round for that beacon
    bytes32 userRandomness;  // the caller entropy mixed in
    bytes32 randomValue;     // the delivered result
}
```

`getProof` reverts for unknown or unfulfilled requests. It returns every input needed to reconstruct the result.

## Re-derive and check

The delivered `randomValue` is exactly:

```
randomValue == keccak256(abi.encode(
    beaconValue,      // from the proof / RandomBeaconHistory.getEntry(effectiveSeq)
    requestId,
    userRandomness,
    effectiveSeq,
    requester
))
```

Anyone can recompute this off-chain (or in a contract) and compare against the `randomValue` the consumer received. If they match, the result is honest.

```js
// pseudo-verification
const p = await nativeVRNG.getProof(requestId);
const expected = keccak256(abi.encode(
  p.beaconValue, p.requestId, p.userRandomness, p.effectiveSeq, p.requester));
assert(expected === p.randomValue);
```

## Why the inputs themselves are trustworthy

Re-deriving the hash only proves the value is consistent. The *un-manipulability* comes from two properties of the inputs:

1. **The beacon is an unforgeable BLS-VRF value.** Each `beaconValue` is a drand / BLS-VRF output verified when it is written to `RandomBeaconHistory`. No one — not the writer, not the requester — can choose or predict it. You can read it back with `RandomBeaconHistory.getBeaconValue(effectiveSeq)` / `getEntry(effectiveSeq)` and confirm it matches the proof.

2. **The beacon was published *after* your request.** A request made at block `requestBlock` is fulfilled only by a beacon whose contract-recorded `writtenAtBlock` is **strictly greater** than `requestBlock`. The selected beacon is the **canonical first-eligible** one (the protocol pins it; the relayer cannot grind for a favorable value). So the value feeding your result did not exist, and could not be known, when you committed to the request. See [How it works](how-it-works.md).

Together: the value is unforgeable *and* served strictly after you were committed — the two ingredients of bias-resistant randomness.

## Path-equivalence

A request fulfilled by the relayer and the same request finalized via [`reveal`](requesting-randomness.md#finalizing-with-reveal) produce the **identical** `randomValue` — the derivation depends only on the request and the first-eligible beacon, not on who triggers fulfillment. Verification is the same in both cases.

## Next

- [How it works](how-it-works.md) — the beacon, the first-eligible rule, and front-running resistance in depth.
- [Contract reference](contract-reference.md) — `getProof`, `RandomBeaconHistory` reads.
