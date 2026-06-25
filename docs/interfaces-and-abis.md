# Interfaces & ABIs

Everything you need to integrate, published here so you don't need access to the contracts repository. **Copy the interface(s) below into your project** and import them locally, and use the JSON ABIs for off-chain code (ethers / viem / wagmi).

- [Solidity interfaces](#solidity-interfaces) — copy into your contracts.
- [JSON ABIs](#json-abis) — for frontends / indexers / scripts.

---

## Solidity interfaces

### `IVRNGProvider` — the entry interface you call

Copy this into your project (e.g. `interfaces/IVRNGProvider.sol`) and import it.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

struct ProviderInfo {
    address provider;
    uint256 fee;
    uint256 totalRequests;
}

/// @title IVRNGProvider — ApeChain Native vRNG consumer interface.
interface IVRNGProvider {
    /// Instant, single-transaction request. `userCommitment` is your entropy
    /// mixed into the derivation. Returns the request id.
    function requestRandomness(bytes32 userCommitment) external payable returns (bytes32 requestId);

    /// Commit-reveal request. `userCommitment = keccak256(secret)`.
    function requestCommitReveal(
        bytes32 userCommitment,
        address callbackContract,
        uint32 callbackGasLimit
    ) external payable returns (bytes32 requestId);

    /// Proof-of-Play-compatible request.
    function requestRandom(
        bytes32 userCommitment,
        address callbackContract,
        uint32 callbackGasLimit
    ) external payable returns (bytes32 requestId);

    /// Pyth Entropy-compatible finalization of a commit-reveal request.
    function reveal(bytes32 requestId, bytes32 userRandomness, bytes calldata providerRandomness) external;

    /// The fee (wei of $APE) required for `caller` to submit a request.
    function getFee(address caller) external view returns (uint256 fee);

    /// Current provider fee + request-volume info.
    function getProviderInfo(address provider) external view returns (ProviderInfo memory info);
}
```

### `ILootBoxCallback` — the callback your contract implements

The provider delivers the result by calling `fulfillRandomness` on the contract you nominated. Copy this alongside `IVRNGProvider` (e.g. `interfaces/ILootBoxCallback.sol`), or just declare the function directly on your consumer — the name is historical; you only need the shape:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// Implement on your consumer to receive vRNG fulfilments.
interface ILootBoxCallback {
    /// @dev Caller MUST be the trusted vRNG provider (NativeVRNG) address.
    /// @param requestId   The id returned by your request.
    /// @param randomValue keccak256(beaconValue, requestId, userRandomness, effectiveSeq, requester).
    function fulfillRandomness(bytes32 requestId, bytes32 randomValue) external;
}
```

### `IProviderProof` — read a verifiable proof (optional)

For independent verification of a fulfilled request — see [Verifying randomness](verifying-randomness.md).

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface IProviderProof {
    struct Proof {
        bytes32 requestId;
        address requester;
        uint64  effectiveSeq;
        bytes32 beaconValue;
        uint64  beaconRound;
        bytes32 userRandomness;
        bytes32 randomValue;
    }

    /// Reverts for unknown or unfulfilled requestIds.
    function getProof(bytes32 requestId) external view returns (Proof memory proof);
}
```

`NativeVRNG` also exposes a few extras beyond these (`pendingRequestBlock`, `REVEAL_GRACE`, `claimRefund`, `withdrawableFees`) — see [Contract reference](contract-reference.md).

---

## JSON ABIs

The deployed contract ABIs (use these in ethers / viem / wagmi). These are the authoritative mainnet ABIs:

| ABI | File |
|---|---|
| **NativeVRNG** (full) | [`abi/NativeVRNG.abi.json`](abi/NativeVRNG.abi.json) |
| **FeeManager** (full) | [`abi/FeeManager.abi.json`](abi/FeeManager.abi.json) |
| **RandomBeaconHistory** (full) | [`abi/RandomBeaconHistory.abi.json`](abi/RandomBeaconHistory.abi.json) |
| **Consumer subset** (the calls + events most integrators need) | [`abi/IVRNGProvider-consumer.abi.json`](abi/IVRNGProvider-consumer.abi.json) |

```ts
import { Contract } from "ethers";
import abi from "./abi/IVRNGProvider-consumer.abi.json";

const vrng = new Contract(NATIVE_VRNG_ADDRESS, abi, signer);
const fee  = await vrng.getFee(myAddress);
const tx   = await vrng.requestRandomness(userCommitment, { value: fee });
```

These JSON files ship with these docs. If you're reading the rendered docs (no repo access), you can also grab each ABI from the verified contract's *Code* tab on the explorer — see [Networks & addresses](networks.md).

## Next

- [Getting started](getting-started.md) — use these in a minimal contract.
- [Contract reference](contract-reference.md) — the full signature/event/error reference.
