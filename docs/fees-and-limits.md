# Fees & rate limits

Pricing, rate limiting, and refunds are owned by the **`FeeManager`** contract. As an integrator you mostly need one call — `getFee` — and an awareness of rate limits and refunds. As an operator, see [Operations & governance](operations.md).

---

## Paying for a request

Always read the fee at call time and forward it:

```solidity
uint256 fee = vrng.getFee(address(this));   // wei of $APE
bytes32 id  = vrng.requestRandomness{ value: fee }(userCommitment);
```

```solidity
function getFee(address caller) external view returns (uint256 fee);
```

- `getFee` returns the fee **for a specific caller** — it already accounts for any per-app override or whitelist tier (below). Pass the address that will submit the request (usually `address(this)`).
- Send **at least** `fee`. Less reverts. More is accepted — refund the excess yourself (see the [example](../examples/lootbox.md)).

## How the fee is determined

`FeeManager.getFee(caller)` resolves the fee in this exact order — **first match wins**:

1. **`WAIVER` tier → `0`** — a waived consumer is always free, **even if a per-app override is set** (the waiver is checked first).
2. **Per-app override** — a non-zero `setFeeOverride(consumer, …)` value.
3. **`REDUCED` tier → `reducedFee`** (the discounted fee).
4. **Default → `defaultFee`** — everyone else.

| Tier | Fee (when no override applies) |
|---|---|
| `NONE` (default) | `defaultFee` |
| `REDUCED` | `reducedFee` (discount) |
| `WAIVER` | `0` (free — takes precedence over any override) |

A governance-set **`maxFee`** is a hard ceiling: no fee setter can push any fee above it, so even a misconfiguration can't overcharge beyond the cap.

**Read helpers:** `getTier(consumer)`, `getFeeOverride(consumer)`, `getFeeMode(consumer)`.

## Fee mode (accounting label)

```solidity
enum FeeMode { USER_PAID, DEV_PAID, PROTOCOL_SUBSIDISED }
```

`FeeMode` is **accounting metadata only** — it records *who* is footing the fee for analytics/billing and does **not** change the on-chain fee math. Set per consumer by governance (`setFeeMode`).

---

## Rate limits

`FeeManager` enforces a fixed-window, per-subject rate limit:

- Each subject (requesting address) may make up to **`maxRequests`** requests per rolling **`windowSeconds`** window.
- Setting either to `0` **disables** rate limiting.
- Inspect current usage with `getRateBucket(subject)` → `{ windowStart, count }`.

If you expect bursts (e.g. a mint), check with your operator that the configured window/limit accommodates them, or request a per-app adjustment.

---

## Refunds

A request fee can be returned to the requester in one case: an operator **cancels** a pending request (`cancelAndRefund`, admin-only). Then:

- The fee (`feeCharged`) is refunded and `RequestCancelled(requestId, requester, refundAmount)` is emitted.
- If the immediate refund transfer fails (e.g. the requester is a contract that rejects it), the amount is **credited for later pull** and `RefundDeferred(...)` is emitted. Claim it with:

  ```solidity
  function claimRefund() external;   // pulls any deferred refund owed to msg.sender
  ```

Normal fulfillment does **not** refund — the fee pays for the randomness produced.

<a id="callback-gas"></a>
## Callback gas

When you request, you (implicitly or explicitly) set how much gas your `fulfillRandomness` callback may use:

- **Instant flow** (`requestRandomness`) uses the **default budget: 250,000 gas** (`DEFAULT_CALLBACK_GAS`).
- **Commit-reveal / PoP** let you pass `callbackGasLimit`; `0` falls back to the default.
- The protocol caps any requested budget at **`maxCallbackGas`** (governance-settable; default 250,000). A request whose `callbackGasLimit` exceeds the cap is rejected.

Why the cap exists: fulfillments are **batched** for throughput, so each callback's budget must stay bounded — one heavy callback can't crowd out the batch. Keep callbacks light; push heavy work to a follow-up transaction (see [Receiving randomness](receiving-randomness.md)).

## Next

- [Operations & governance](operations.md) — how operators set fees, tiers, limits, and the cap.
- [Contract reference](contract-reference.md) — exact `FeeManager` signatures and events.
