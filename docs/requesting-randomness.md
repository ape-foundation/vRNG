# Requesting randomness

`NativeVRNG` exposes three request entry points and one finalization method, all on the `IVRNGProvider` interface. All request methods are `payable` and return a `bytes32 requestId`.

| Method | Flow | Callback? | Use when |
|---|---|---|---|
| [`requestRandomness`](#instant-flow) | Instant (1 tx) | Yes | Default. Simplest; one transaction. |
| [`requestCommitReveal`](#commit-reveal-flow) | Commit-reveal (2 tx) | Yes | You want the requester's *secret* mixed in and a user-driven finalization. |
| [`requestRandom`](#pop-compatibility) | Proof-of-Play compat (2 tx) | Yes | Migrating from a PoP-style integration. |
| [`reveal`](#finalizing-with-reveal) | Finalize a commit-reveal request | — | To deliver a commit-reveal result without waiting for the relayer. |

> Always read the fee with [`getFee`](fees-and-limits.md) and forward it as `msg.value`. Sending less reverts; sending more is your contract's responsibility to refund.

> **Compatibility — migrating from Pyth Entropy or Proof-of-Play?** The surface is intentionally dual: the **Pyth Entropy** pattern maps to `requestCommitReveal` + [`reveal`](#finalizing-with-reveal); the **Proof-of-Play** pattern maps to `requestRandom` + the `fulfillRandomness` callback. Keep your existing flow shape. New, ApeChain-native code should prefer the one-step `requestRandomness`. Full step-by-step in the [Migration guide](migration-guide.md).

---

## Instant flow

```solidity
function requestRandomness(bytes32 userCommitment)
    external payable returns (bytes32 requestId);
```

The default, single-transaction flow. The protocol fulfils the request automatically and calls your [`fulfillRandomness`](receiving-randomness.md) back. Uses the **default callback gas budget** (250,000 — see [Fees & rate limits](fees-and-limits.md#callback-gas)).

- **`userCommitment`** — caller-supplied entropy mixed into the derivation. For the instant flow it need not be secret; use it to bind a request to context (e.g. `keccak256(abi.encode(player, round))`).
- **Returns** `requestId` — store it to correlate the callback.

```solidity
bytes32 id = vrng.requestRandomness{ value: vrng.getFee(address(this)) }(
    keccak256(abi.encode(msg.sender, block.number))
);
```

## Commit-reveal flow

```solidity
function requestCommitReveal(
    bytes32 userCommitment,   // keccak256(secret)
    address callbackContract, // contract to receive fulfillRandomness
    uint32  callbackGasLimit  // gas forwarded to the callback
) external payable returns (bytes32 requestId);
```

A two-transaction flow where `userCommitment` is the **hash of a secret** the requester holds. The request can then be finalized either automatically (relayer) or by the requester via [`reveal`](#finalizing-with-reveal). Lets you choose the callback target and gas budget explicitly.

- **`callbackGasLimit`** is forwarded to your callback; passing `0` uses the default (250,000). It is capped by the protocol's `maxCallbackGas` — see [callback gas](fees-and-limits.md#callback-gas).
- A **reveal grace window** of `REVEAL_GRACE = 64` blocks after the request protects the requester: during the grace, only the original requester may `reveal`; after it, anyone holding the secret may finalize.

## PoP compatibility

```solidity
function requestRandom(
    bytes32 userCommitment,
    address callbackContract,
    uint32  callbackGasLimit
) external payable returns (bytes32 requestId);
```

Behaviorally equivalent to `requestCommitReveal`, provided for Proof-of-Play–style integrations migrating to native vRNG. Prefer `requestRandomness` (instant) or `requestCommitReveal` for new code.

## Finalizing with `reveal`

```solidity
function reveal(
    bytes32 requestId,
    bytes32 userRandomness,        // the pre-image of your userCommitment
    bytes  calldata providerRandomness
) external;
```

Pyth Entropy–compatible finalization for a commit-reveal request. Delivers the random value to the consumer callback without waiting for the relayer. The result is **identical** whether a request is fulfilled by the relayer or by `reveal` (path-equivalence) — see [How it works](how-it-works.md). `userRandomness` must be the pre-image whose `keccak256` equals the `userCommitment` you committed.

---

## The `requestId`

`requestId` is deterministic and unique per request:

```
requestId = keccak256(abi.encode(nativeVrngAddress, requester, nonce, requestBlock))
```

Use it as the key for your per-request bookkeeping. You can read a pending request's origin block with `pendingRequestBlock(requestId)`.

## What happens after a request

1. `NativeVRNG` emits **`RandomnessRequested(requestId, requester, requestBlock, userRandomness)`** and records the request.
2. The request becomes fulfillable by the **first beacon written strictly after `requestBlock`**.
3. On fulfillment, the protocol emits **`RandomnessFulfilled(...)`**, calls your **`fulfillRandomness`** callback, and emits **`CallbackDispatched(requestId, target, success)`**.

A request that is never fulfilled by traffic is **always eventually fulfillable** by a later beacon — there is no "stranded" terminal state. If an operator cancels a request, your fee is refunded (`cancelAndRefund` → `RequestCancelled`; pull a deferred refund with `claimRefund`). See [Fees & rate limits](fees-and-limits.md#refunds).

## Next

- [Receiving randomness](receiving-randomness.md) — implement the callback.
- [Verifying randomness](verifying-randomness.md) — prove a result was fair.
