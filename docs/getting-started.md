# Getting started

Integrate ApeChain Native vRNG into a consumer contract in three steps. This page is the quickstart; each step links to a deeper page.

**Prerequisites:** intermediate Solidity, a deployed `NativeVRNG` address ([Networks & addresses](networks.md)), and some $APE to pay the per-request fee.

---

## The model

vRNG is **request → callback** (asynchronous), like Chainlink VRF or Pyth Entropy:

1. Your contract **requests** randomness and pays a fee.
2. The protocol fulfils the request with the first beacon written after it.
3. The protocol **calls your contract back** with the random value.

Your contract therefore needs to do two things: **send a request**, and **implement a callback**.

---

## Step 1 — Import the interface

`NativeVRNG` implements `IVRNGProvider`. Copy that interface from [Interfaces & ABIs](interfaces-and-abis.md) into your project, import it, and store the provider address — `0xB9B0d73104BE5e286142258204AA497435a97415` on ApeChain mainnet ([all networks](networks.md)).

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import { IVRNGProvider } from "./interfaces/IVRNGProvider.sol"; // copy from docs → Interfaces & ABIs

contract MyGame {
    IVRNGProvider public immutable vrng;

    constructor(address vrngProvider) {
        vrng = IVRNGProvider(vrngProvider);
    }
}
```

## Step 2 — Request randomness

Read the fee with `getFee`, then call `requestRandomness` forwarding that fee. Store the returned `requestId` so you can recognize the request when it comes back.

```solidity
mapping(bytes32 => address) public requesterOf;

function roll(bytes32 userCommitment) external payable returns (bytes32 requestId) {
    uint256 fee = vrng.getFee(address(this));
    require(msg.value >= fee, "fee too low");

    requestId = vrng.requestRandomness{ value: fee }(userCommitment);
    requesterOf[requestId] = msg.sender;
    // refund msg.value - fee if you collected extra
}
```

- `userCommitment` is your own entropy (e.g. `keccak256(player, salt)`), mixed into the derivation — it does **not** need to be secret for the instant flow.
- `requestId` uniquely identifies this request. Details: [Requesting randomness](requesting-randomness.md).

## Step 3 — Implement the callback

The protocol calls `fulfillRandomness(requestId, randomValue)` on the contract you requested from. **Guard it so only `NativeVRNG` can call it**, then use the value.

```solidity
function fulfillRandomness(bytes32 requestId, bytes32 randomValue) external {
    require(msg.sender == address(vrng), "only vRNG");

    address player = requesterOf[requestId];
    require(player != address(0), "unknown request");
    delete requesterOf[requestId];

    uint256 outcome = uint256(randomValue) % 100; // map to your range
    // ... apply outcome ...
}
```

That's the whole integration. Keep the callback **cheap and non-reverting** — see [Receiving randomness](receiving-randomness.md) for why and how.

---

## Full working example

See [examples/lootbox.md](../examples/lootbox.md) for a complete, annotated consumer (`LootBoxDemo`) that implements both the instant and commit-reveal flows, handles fee refunds, and derives a rarity tier from the random value.

## Next

- **Next: [Requesting randomness](requesting-randomness.md)** — the request flows in detail.
- [Receiving randomness](receiving-randomness.md) — make your callback safe.
- See also: [Fees & rate limits](fees-and-limits.md) · [Verifying randomness](verifying-randomness.md)
