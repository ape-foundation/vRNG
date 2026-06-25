# Example: Loot Box

A complete, annotated consumer that opens loot boxes with verifiable randomness and derives a rarity tier. It implements **both** the instant and commit-reveal flows, refunds fee overpayment, and shows the safe callback pattern. This mirrors the shipped `LootBoxDemo` contract.

> Read [Getting started](../docs/getting-started.md) first for the three-step model.

---

## The contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import { IVRNGProvider } from "./interfaces/IVRNGProvider.sol"; // copy from docs → Interfaces & ABIs

contract LootBoxDemo {
    enum ItemTier { Common, Uncommon, Rare, Epic, Legendary }

    struct PendingRequest { address requester; bool commitReveal; }

    /// Gas forwarded to the callback in the commit-reveal flow.
    uint32 public constant CALLBACK_GAS_LIMIT = 100_000;

    IVRNGProvider public vrng;                                   // the provider (configurable)
    mapping(bytes32 => PendingRequest) public pendingRequests;   // in-flight requests

    event LootBoxOpened(bytes32 indexed requestId, address indexed requester, bool commitReveal);
    event ItemDropped(bytes32 indexed requestId, address indexed requester, ItemTier tier, bytes32 randomValue);

    error UnauthorizedCallback(address caller);
    error UnknownRequest(bytes32 requestId);
    error InsufficientFee(uint256 required, uint256 provided);

    // --- Request: instant flow -------------------------------------------
    function openInstant(bytes32 userCommitment) external payable returns (bytes32 requestId) {
        uint256 requiredFee = vrng.getFee(address(this));
        if (msg.value < requiredFee) revert InsufficientFee(requiredFee, msg.value);

        requestId = vrng.requestRandomness{ value: requiredFee }(userCommitment);
        pendingRequests[requestId] = PendingRequest({ requester: msg.sender, commitReveal: false });

        if (msg.value > requiredFee) {                            // refund the excess
            (bool ok, ) = msg.sender.call{ value: msg.value - requiredFee }("");
            require(ok, "refund failed");
        }
        emit LootBoxOpened(requestId, msg.sender, false);
    }

    // --- Request: commit-reveal flow -------------------------------------
    function openCommit(bytes32 secretHash) external payable returns (bytes32 requestId) {
        uint256 requiredFee = vrng.getFee(address(this));
        if (msg.value < requiredFee) revert InsufficientFee(requiredFee, msg.value);

        requestId = vrng.requestCommitReveal{ value: requiredFee }(secretHash, address(this), CALLBACK_GAS_LIMIT);
        pendingRequests[requestId] = PendingRequest({ requester: msg.sender, commitReveal: true });

        if (msg.value > requiredFee) {
            (bool ok, ) = msg.sender.call{ value: msg.value - requiredFee }("");
            require(ok, "refund failed");
        }
        emit LootBoxOpened(requestId, msg.sender, true);
    }

    // --- Callback --------------------------------------------------------
    function fulfillRandomness(bytes32 requestId, bytes32 randomValue) external {
        if (msg.sender != address(vrng)) revert UnauthorizedCallback(msg.sender);   // rule 1

        PendingRequest memory req = pendingRequests[requestId];
        if (req.requester == address(0)) revert UnknownRequest(requestId);          // rule 2
        delete pendingRequests[requestId];

        ItemTier tier = _deriveTier(randomValue);                                   // rule 3: cheap
        emit ItemDropped(requestId, req.requester, tier, randomValue);
    }

    function _deriveTier(bytes32 randomValue) internal pure returns (ItemTier) {
        uint256 roll = uint256(randomValue) % 100;
        if (roll < 50) return ItemTier.Common;     // 50%
        if (roll < 75) return ItemTier.Uncommon;   // 25%
        if (roll < 90) return ItemTier.Rare;       // 15%
        if (roll < 98) return ItemTier.Epic;       //  8%
        return ItemTier.Legendary;                 //  2%
    }
}
```

*(The shipped contract is an upgradeable `UUPS`/`Ownable` variant with `initialize(_vrng, _owner)` and an owner-only `setVrngProvider`; trimmed here for clarity.)*

---

## What to notice

- **Fee handling.** It reads `getFee(address(this))` every call (the fee can change), forwards exactly that, and refunds the excess to the user. → [Fees & rate limits](../docs/fees-and-limits.md).
- **Correlation.** It stores `pendingRequests[requestId]` at request time and reads it in the callback. `requestId` is the join key.
- **The three callback rules** (numbered in the code): provider-only caller, recognize-then-delete the request, keep it cheap. → [Receiving randomness](../docs/receiving-randomness.md).
- **Outcome derivation.** A simple `% 100` weighted table. For exact uniformity use rejection sampling; for many draws, hash `randomValue` with an index.
- **Configurable provider.** Production code makes the `vrng` address settable (with an event) rather than hard-coded, so it survives a redeploy / mainnet move. → [Networks & addresses](../docs/networks.md).

## Verifying a drop

Anyone can confirm a drop was fair: take the `requestId`, call [`getProof`](../docs/verifying-randomness.md), re-derive `randomValue`, and re-run `_deriveTier`. If the tier matches the emitted `ItemDropped`, the drop was honest and tamper-proof.

## Next

- [Verifying randomness](../docs/verifying-randomness.md)
- [How it works](../docs/how-it-works.md)
