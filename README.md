# ApeChain Native vRNG — Documentation

**Native verifiable randomness for ApeChain.** Your contract requests a random value and pays a small **$APE** fee; the value is delivered to a callback and can be **independently verified on-chain**. No external oracle network, no off-chain subscription — randomness is produced and served natively.

> Use it for NFT mint reveals, on-chain games, loot boxes, lotteries, raffles, fair ordering — anywhere an outcome must be random *and* provably fair.

---

## How it works in one paragraph

Your contract calls **`requestRandomness`** on the `NativeVRNG` contract with a fee and some caller entropy. A monotonically-sequenced **randomness beacon** is written on-chain continuously. Your request is fulfilled by the **first beacon written strictly after your request block** — a value you could not have known when you requested, and that the provider cannot retroactively choose. The derived random value is delivered to your contract's **`fulfillRandomness`** callback, and anyone can later re-derive and **verify** it from the published beacon. See [How it works](docs/how-it-works.md).

---

## Documentation

### For integrators (consume randomness)

> **Already using Pyth Entropy or Proof-of-Play?** Start with the [Migration guide](docs/migration-guide.md) — it's a ~3-line change.

1. [Getting started](docs/getting-started.md) — integrate in three steps, with a minimal contract.
2. [Requesting randomness](docs/requesting-randomness.md) — the instant and commit-reveal flows, fees, and `requestId`.
3. [Receiving randomness](docs/receiving-randomness.md) — implementing the `fulfillRandomness` callback safely.
4. [Fees & rate limits](docs/fees-and-limits.md) — `getFee`, tiers, fee modes, rate limits, refunds.
5. [Verifying randomness](docs/verifying-randomness.md) — `getProof` and independent verification.
6. [Contract reference](docs/contract-reference.md) — full interface, events, errors, constants.
7. [Interfaces & ABIs](docs/interfaces-and-abis.md) — copy-paste interfaces + JSON ABIs (the contracts repo is private; everything you need is here).
8. [Networks & addresses](docs/networks.md) — deployed contract addresses.
9. [Example: Loot Box](examples/lootbox.md) — a complete, annotated consumer contract.

### Under the hood
- [How it works](docs/how-it-works.md) — the beacon, fulfillment, and front-running resistance.

### For protocol operators
- [Operations & governance](docs/operations.md) — pause, fees, rate limits, callback-gas, health, and Safe-based governance.

---

## The three contracts

| Contract | What it is | You interact with it to… |
|---|---|---|
| **`NativeVRNG`** | The randomness service and sole entry point. | Request randomness, read fees/proofs (integrators); pause/configure (operators). |
| **`FeeManager`** | Fees, fee tiers, whitelist, and rate limits. | Read your fee; operators tune pricing/limits. |
| **`RandomBeaconHistory`** | The append-only beacon log (the randomness source). | Read/verify beacon values; mostly internal. |

Addresses for each network are in [Networks & addresses](docs/networks.md).

> **Terms.** *vRNG* — the product. *`NativeVRNG`* — the entry contract you call. *Integrator* / "you" — the developer integrating. *Consumer* — your deployed contract that calls vRNG and receives the callback. *Caller* — the `address` parameter in reads like `getFee` (usually your consumer contract).

---

*This documentation describes the final, native on-chain design (single direct entry point, time/sequence-keyed beacon, Gnosis-Safe governance), **live on ApeChain mainnet** (chain `33139`). The **deployed mainnet ABIs + addresses are the source of truth**; signatures here are taken verbatim from them. See [Networks & addresses](docs/networks.md).*
