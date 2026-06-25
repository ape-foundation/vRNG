# Networks & addresses

ApeChain Native vRNG is **live on ApeChain mainnet**. As an integrator you normally only need the **`NativeVRNG`** address — it is the entry point; you read fees and proofs through it and it calls your callback.

> All contracts are deployed behind **transparent proxies** — integrate against the **proxy address** below (it is stable across implementation upgrades). The mainnet ABIs + addresses are the **final source of truth**; the values here are taken from the deployment.

---

## ApeChain (mainnet) — chain ID `33139`

| Field | Value |
|---|---|
| Chain ID | `33139` |
| Explorer | `https://apescan.io` |
| Native token | `$APE` |

| Contract | Address (proxy) |
|---|---|
| **`NativeVRNG`** (entry point) | `0xB9B0d73104BE5e286142258204AA497435a97415` |
| `RandomBeaconHistory` | `0x031a84F81a9505E48624936c274f4DfD497676Ae` |
| `FeeManager` | `0x2125EA37840c7f15d6b5D352049329a79e90803D` |

> **ApeChain Curtis (testnet):** a public deployment is **coming soon**. Until then, integrate against mainnet (above).

---

## Wiring it up

```solidity
// ApeChain mainnet
address constant VRNG = 0xB9B0d73104BE5e286142258204AA497435a97415;
IVRNGProvider vrng = IVRNGProvider(VRNG);
```

Make the address **configurable** (constructor arg or an owner-settable setter) rather than hard-coding it, so you can switch networks — or survive a redeploy — without redeploying your consumer. The example contract does this via `setVrngProvider` — see [examples/lootbox.md](../examples/lootbox.md).

## Verifying you have the right contract

- `NativeVRNG.getFee(yourAddress)` returns a non-reverting `uint256` fee.
- `RandomBeaconHistory.latestSeq()` returns a steadily-increasing sequence number (the beacon is live).

## ABIs

The interfaces and JSON ABIs are published in **[Interfaces & ABIs](interfaces-and-abis.md)** (the same ABIs are also on the verified contracts on the explorer). For day-to-day integration you only need the `IVRNGProvider` + `ILootBoxCallback` interfaces.

## Next

- [Getting started](getting-started.md) — integrate against these addresses.
- [Operations & governance](operations.md) — for operators running a deployment.
