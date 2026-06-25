# Operations & governance

For the teams that **run** an ApeChain Native vRNG deployment (not integrate with it). It covers who can change what, how, and how to monitor health. Integrators can skip this page.

> **Governance is direct, not proxied.** Privileged changes are submitted straight to the contracts — there is no backend that signs or proxies them. Parameter changes are typically made from a **Gnosis Safe** via the **Safe{Wallet} app**; the emergency pause role may be a Safe or a plain admin EOA. The operator dashboard *reads* on-chain state and *links* to the right holder; it never executes a change itself.

---

## Two kinds of control

| | **Operate** | **Govern** |
|---|---|---|
| What | Day-to-day service actions + monitoring | On-chain contract parameter changes |
| Who acts | Operators (via the dashboard / services) | **Gnosis Safe signers**, in Safe{Wallet} |
| Examples | Watch health, drain/resume the relayer, read metrics | Pause, set fees/tiers/limits/callback-gas, withdraw fees, unpause |
| Trust | Backend services | The role holders — the contracts hold no privileged backend |

The rest of this page is **Govern** (the on-chain controls). The dashboard is your read + guide surface for all of it.

## Roles & holders

| Role (on-chain) | Held by | Powers |
|---|---|---|
| `ADMIN_ROLE` | **Governance Safe**, optionally behind a Timelock | All parameter changes, unpause, fee withdrawal, cancel-and-refund |
| `PAUSER_ROLE` | **A Safe or an admin EOA** (your choice) | Emergency `pause()` — takes effect immediately |
| `RATE_LIMITER_ROLE` | `NativeVRNG` | Internal — ticks the rate limiter |
| `FULFILLER_ROLE` | The relayer signer | Internal — submits fulfillments |

Addresses for the role holders are recorded in the deployment and shown on the dashboard.

---

## The controls

All changes below are **Governance Safe** transactions on the named contract (pause is the exception — see §Pause). If a Timelock is enabled, parameter changes are *schedule → wait → execute*.

### Pause (emergency stop)

| To do this | Call | From |
|---|---|---|
| Pause randomness | `NativeVRNG.pause()` | **`PAUSER_ROLE` holder** (Safe or EOA) — immediate |
| Pause the fee/rate-limit module | `FeeManager.pause()` | **`PAUSER_ROLE` holder** — immediate |
| Unpause | `NativeVRNG.unpause()` / `FeeManager.unpause()` | **Governance Safe** (admin) |

Pausing is fast (a single action, no Timelock delay) so you can stop the service instantly in an incident; unpausing is deliberate (admin, and any Timelock delay applies). `RandomBeaconHistory` is **not** pausable.

### Fees (`FeeManager`)

| To do this | Call |
|---|---|
| Set the standard fee | `setDefaultFee(newFee)` |
| Set the discounted (REDUCED) fee | `setReducedFee(newFee)` |
| Set the hard ceiling | `setMaxFee(newMaxFee)` |
| Set a per-app fee | `setFeeOverride(consumer, newFee)` |
| Set an app's tier | `setWhitelist(consumer, tier)` — `NONE` / `REDUCED` / `WAIVER` |
| Tag who pays (accounting) | `setFeeMode(consumer, mode)` — `USER_PAID` / `DEV_PAID` / `PROTOCOL_SUBSIDISED` |

`setMaxFee` is a guardrail: no fee can exceed it. See [Fees & rate limits](fees-and-limits.md).

### Rate limits (`FeeManager`)

| To do this | Call |
|---|---|
| Set per-subject limit | `setRateLimitParams(windowSeconds, maxRequests)` |

`maxRequests` requests per rolling `windowSeconds` per address; either `0` disables limiting.

### Callback gas (`NativeVRNG`)

| To do this | Call |
|---|---|
| Set the max callback gas a consumer may request | `setMaxCallbackGas(newMax)` |

The lever between per-consumer callback weight and batch throughput. Recommended band: floor ≈ 100,000 / default 250,000 / ceiling ≈ 500,000. Keep it as low as your consumers tolerate. See [callback gas](fees-and-limits.md#callback-gas).

### Protocol fees & refunds (`NativeVRNG`)

| To do this | Call |
|---|---|
| Sweep collected fees to a recipient | `withdrawFees(to, amount)` (bounded by `withdrawableFees()`) |
| Cancel a pending request and refund it | `cancelAndRefund(requestId)` |

---

## Health monitoring

The operator dashboard surfaces, in plain view:

- **Beacon freshness** — is the beacon being produced on time (Fresh / Syncing / Stale)? Freshness reflects *writer liveness* (last beacon ingested), not indexer backfill position.
- **Writer health** — signer, balance, cadence.
- **Request metrics** — request rate, fulfillment success rate (24h), p99 latency.
- **Pause status** — per contract (read on-chain).
- **Security/audit feed** — pauses, role changes, notable events.

On-chain spot checks: `RandomBeaconHistory.latestSeq()` (steadily increasing = beacon live), `NativeVRNG.paused()` / `FeeManager.paused()`, `NativeVRNG.maxCallbackGas()`, `FeeManager.getFee(app)`.

> **Quick health rule:** beacon Fresh + nothing Paused = healthy. A genuinely Stale beacon (the chain's own beacon head isn't advancing) or an unexpected Paused = page the on-call operator. A "Syncing" beacon after a restart is normal (the indexer is catching up) and self-resolves.

---

## Optional: Timelock

A Timelock adds an enforced waiting period between *approving* a parameter change and it *taking effect* — a safety net against rushed or mistaken changes. It is **optional** and **off by default**; emergency pause is **never** delayed.

When enabled, parameter changes become *schedule → wait → execute*, and admin roles on all three contracts are held by the Timelock (proposer/executor = Governance Safe). To enable: deploy an OpenZeppelin `TimelockController` with your chosen delay, transfer the admin roles to it, and renounce the old admin. Keep the admin healthy — if you use a Governance Safe, keep its signer set reachable; once the Timelock is sole admin, rotating it requires a working admin.

---

## How to submit a change (recap)

1. Open **Safe{Wallet}** for the Governance Safe (parameter changes), or act as the `PAUSER_ROLE` holder (Safe or EOA) for emergency pause.
2. **New transaction → Contract interaction**, enter the contract address, pick the method.
3. Collect the required signatures.
4. If a Timelock is enabled, *schedule*, wait the delay, then *execute*.
5. Confirm the change on the dashboard.

*For copy-paste calldata per method, see the operator's parameter-change runbook. Exact signatures are in the [Contract reference](contract-reference.md).*
