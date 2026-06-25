# Receiving randomness

When a request is fulfilled, `NativeVRNG` calls a single function on the contract you nominated. Your contract must implement it.

```solidity
/// Caller MUST be the trusted vRNG provider (NativeVRNG).
/// randomValue = keccak256(beaconValue, requestId, userRandomness, effectiveSeq, requester)
function fulfillRandomness(bytes32 requestId, bytes32 randomValue) external;
```

- **`requestId`** — the id returned by your original request; use it to look up your per-request state.
- **`randomValue`** — a uniformly-distributed 256-bit value derived from the beacon. Map it into your range (`uint256(randomValue) % N`, or rejection sampling for exact uniformity).

---

## The three rules of a safe callback

### 1. Only the provider may call it

Anyone can call a public function. If you don't gate `fulfillRandomness`, an attacker could feed you a chosen `randomValue`. Always check the caller:

```solidity
if (msg.sender != address(vrng)) revert UnauthorizedCallback(msg.sender);
```

### 2. Recognize the request, then clear it

Look the `requestId` up in your own bookkeeping; reject unknown or already-handled ids; delete the entry before acting (prevents replay and refunds gas):

```solidity
address player = requesterOf[requestId];
if (player == address(0)) revert UnknownRequest(requestId);
delete requesterOf[requestId];
```

### 3. Keep it cheap and non-reverting

The callback runs inside the protocol's fulfillment transaction under a **bounded gas budget** (your `callbackGasLimit`, default 250,000; capped by the protocol's `maxCallbackGas`). Two consequences:

- **Stay within budget.** Typical loot-box / lottery callbacks spend well under 200,000 gas. Do the minimum on the hot path: record the result, emit an event, update light state.
- **A reverting or out-of-gas callback does not roll back the fulfillment.** The protocol records the result on-chain and emits `CallbackDispatched(requestId, target, success)` with `success = false`; it does **not** retry. This is intentional (non-blocking fulfillment) — a buggy consumer can't stall the service or trap its own request. **Design for it:** if your callback can fail, fall back to a *pull* pattern — store `randomValue` against `requestId` and let the user trigger the heavy work in a follow-up transaction.

```solidity
// Heavy work? Don't do it in the callback — store and let the user pull.
function fulfillRandomness(bytes32 requestId, bytes32 randomValue) external {
    if (msg.sender != address(vrng)) revert UnauthorizedCallback(msg.sender);
    results[requestId] = randomValue;        // cheap
    emit Fulfilled(requestId, randomValue);
}
function claimReward(bytes32 requestId) external { /* heavy work here */ }
```

---

## Deriving an outcome

`randomValue` is a full 256-bit value. Common patterns:

```solidity
uint256 roll  = uint256(randomValue) % 100;          // 0..99 (slight modulo bias for non-divisors of 2^256)
bool     coin = uint256(randomValue) & 1 == 1;        // fair coin
uint256 pick  = uint256(keccak256(abi.encode(randomValue, i))); // many independent draws from one value
```

For many draws from a single request, hash `randomValue` with an index/domain tag rather than requesting repeatedly.

## Events you can index

| Event | Meaning |
|---|---|
| `RandomnessRequested(requestId, requester, requestBlock, userRandomness)` | Request accepted. |
| `RandomnessFulfilled(requestId, requester, randomValue, beaconValue, requestBlock, effectiveSeq)` | Result derived and recorded. |
| `CallbackDispatched(requestId, target, success)` | Your callback was attempted; `success` reflects whether it returned without reverting. |

Full list in the [Contract reference](contract-reference.md#nativevrng--events).

## Next

- [Verifying randomness](verifying-randomness.md) — confirm a `randomValue` was honestly derived.
- [Example: Loot Box](../examples/lootbox.md) — the callback in a complete contract.
