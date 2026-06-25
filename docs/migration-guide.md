# Migration guide

Already using **Pyth Entropy** (or a **Proof-of-Play**–style vRNG) on ApeChain? Moving to Native vRNG is a small, mechanical change — the `IVRNGProvider` interface is a superset of Pyth Entropy, so existing consumers migrate with **no more than three line changes and no game-logic rewrite**.

> **`NativeVRNG` is the sole entry point.** There is no router, adapter, or automated provider failover — you point your contract directly at `NativeVRNG` and gate your callback on its address. (If you've seen older docs mentioning a "router" or "failover," that design was dropped.)

## The change in one line

1. **Swap the import** → `IVRNGProvider`.
2. **Swap the address** → the `NativeVRNG` address ([Networks & addresses](networks.md)).
3. **Rename the calls** → `requestRandomness` for the request, `fulfillRandomness` for the callback (and re-gate the callback on the `NativeVRNG` address).

Pick your path: [Pyth Entropy → Native](#pyth-entropy--native) · [Proof-of-Play → Native](#proof-of-play--native).

---

## Pyth Entropy → Native

### What changes

| Step | Action |
|---|---|
| 1 | Import `IVRNGProvider` instead of Pyth's `IEntropy` / `IEntropyConsumer`. |
| 2 | Replace the stored Pyth address with the **`NativeVRNG`** address. |
| 3 | Replace `requestWithCallback` → `requestRandomness` (or use `requestCommitReveal` if you used Pyth's reveal flow). |
| 4 | Rename your `entropyCallback` → `fulfillRandomness`, and gate it on `msg.sender == address(vrng)`. |

### Before → after

**Before (Pyth):**

```solidity
import { IEntropyConsumer } from "@pythnetwork/entropy-sdk-solidity/IEntropyConsumer.sol";
import { IEntropy }         from "@pythnetwork/entropy-sdk-solidity/IEntropy.sol";

contract OldDice {
    IEntropy public immutable pyth;
    address  public immutable pythProvider;

    function roll(bytes32 userRandomNumber) external payable returns (uint64) {
        uint128 fee = pyth.getFee(pythProvider);
        return pyth.requestWithCallback{ value: fee }(pythProvider, userRandomNumber);
    }

    function entropyCallback(uint64 sequenceNumber, address provider, bytes32 randomNumber) internal override {
        require(provider == pythProvider, "wrong provider");
        // ... apply randomNumber ...
    }
}
```

**After (Native):**

```solidity
import { IVRNGProvider } from "./interfaces/IVRNGProvider.sol"; // copy from docs → Interfaces & ABIs

contract NewDice {
    IVRNGProvider public immutable vrng;   // the NativeVRNG address

    function roll(bytes32 userRandomness) external payable returns (bytes32) {
        uint256 fee = vrng.getFee(address(this));
        return vrng.requestRandomness{ value: fee }(userRandomness);
    }

    function fulfillRandomness(bytes32 requestId, bytes32 randomValue) external {
        require(msg.sender == address(vrng), "only vRNG");
        // ... apply randomValue ...
    }
}
```

### Field equivalents

| Pyth Entropy | Native vRNG |
|---|---|
| `entropy.getFee(provider)` → `uint128` | `vrng.getFee(consumer)` → `uint256` |
| `requestWithCallback(provider, userRandomNumber)` → `uint64 sequenceNumber` | `requestRandomness(userCommitment)` → `bytes32 requestId` |
| `revealWithCallback(provider, sequenceNumber, userRandomNumber, providerRandomNumber)` | `reveal(requestId, userRandomness, providerRandomness)` *(commit-reveal only)* |
| `IEntropyConsumer.entropyCallback(sequenceNumber, provider, randomNumber)` | `fulfillRandomness(requestId, randomValue)` — a plain external function on your consumer (interface `ILootBoxCallback`) |
| Gate on `provider` | Gate on `msg.sender == address(vrng)` |

### Two things to watch

- **Identifier width.** Pyth's handle is `uint64 sequenceNumber`; Native's is `bytes32 requestId`. If you key request state on `mapping(uint64 => …)`, change the key type to `bytes32`.
- **Reveal flow.** If you used Pyth's commit-reveal (`requestWithCallback` + `revealWithCallback`), use [`requestCommitReveal`](requesting-randomness.md#commit-reveal-flow) + [`reveal`](requesting-randomness.md#finalizing-with-reveal). For the simple one-shot case, `requestRandomness` + the callback is less code.

---

## Proof-of-Play → Native

Native exposes a Proof-of-Play–compatible request (`requestRandom`) and the same `fulfillRandomness` callback, so PoP-shaped consumers migrate the same way.

### What changes

| Step | Action |
|---|---|
| 1 | Import `IVRNGProvider`. |
| 2 | Point at the **`NativeVRNG`** address. |
| 3 | Use `requestRandom(userCommitment, callbackContract, callbackGasLimit)` (or switch to the simpler `requestRandomness`). |
| 4 | Rename your PoP callback → `fulfillRandomness(requestId, randomValue)`, gated on the `NativeVRNG` address. |

### Field equivalents

| Proof-of-Play | Native vRNG |
|---|---|
| off-chain / invoiced fee (no `getFee`) | `vrng.getFee(consumer)` → per-call $APE fee |
| `requestRandom(arg)` → `uint256 requestId` | `requestRandom(userCommitment, callbackContract, callbackGasLimit)` → `bytes32 requestId` |
| `popRandomCallback(requestId, randomValue)` | `fulfillRandomness(requestId, randomValue)` |

> **Note on `requestRandom`'s signature.** Unlike a bare PoP `requestRandom(arg)`, Native's takes `(userCommitment, callbackContract, callbackGasLimit)` — so this is a small signature change, not a pure rename. New code can just use [`requestRandomness`](requesting-randomness.md#instant-flow).

### Pricing

PoP integrations were typically billed off-chain. On Native the default is **user-paid** (the caller funds the fee). If you don't want end users to see a fee, ask your operator to set your consumer to **developer-paid** or **protocol-subsidized** (whitelist) — see [Fees & rate limits](fees-and-limits.md) and [Operations & governance](operations.md).

---

## Common gotchas

| Gotcha | Symptom | Fix |
|---|---|---|
| Callback still gated on the old provider address | Your callback reverts; the fulfillment still records, so the request is consumed. | Gate on `msg.sender == address(vrng)` (the `NativeVRNG` address). |
| `getFee` cached in the constructor | Requests revert after a governance fee change. | Read `getFee(address(this))` **per call**. |
| Request state keyed on `mapping(uint64 => …)` (from Pyth) | Compile error after the `bytes32` swap. | Change the key type to `bytes32`. |
| Heavy work inside the callback | Out-of-gas; fulfillment marked `success = false` and **not retried**. | Keep the callback light; move heavy work to a follow-up pull tx. See [Receiving randomness](receiving-randomness.md). |
| Hard-coding the implementation address | Breaks after a proxy upgrade. | Use the **proxy** address from [Networks & addresses](networks.md). |

## Test & roll back

- Re-run your suite against the rewritten contract, pointed at the `NativeVRNG` address ([Networks & addresses](networks.md)).
- Migration touches only which contract you call and your callback signature — there is no on-chain state to migrate. To roll back, revert the diff and redeploy against your previous provider. (Make the change on a fresh branch so rollback is one `git checkout`.)

## After migrating

- Verify a fulfilled result with [`getProof`](verifying-randomness.md).
- Read [How it works](how-it-works.md) for the front-running-resistance guarantee you now rely on.

## Next

- [Getting started](getting-started.md) · [Requesting randomness](requesting-randomness.md) · [Receiving randomness](receiving-randomness.md)
