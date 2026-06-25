# Contract reference

Authoritative API surface, taken verbatim from the contract source. Signatures are Solidity `^0.8.24`.

- [NativeVRNG — consumer surface](#nativevrng--consumer-surface)
- [Consumer callback](#consumer-callback)
- [NativeVRNG — events](#nativevrng--events)
- [NativeVRNG — operator surface](#nativevrng--operator-surface)
- [FeeManager](#feemanager)
- [RandomBeaconHistory](#randombeaconhistory)
- [Constants & enums](#constants--enums)

---

## NativeVRNG — consumer surface

From `IVRNGProvider`, `IProviderProof`, and `INativeVRNG`. All request methods are `payable`.

```solidity
function requestRandomness(bytes32 userCommitment) external payable returns (bytes32 requestId);

function requestCommitReveal(bytes32 userCommitment, address callbackContract, uint32 callbackGasLimit)
    external payable returns (bytes32 requestId);

function requestRandom(bytes32 userCommitment, address callbackContract, uint32 callbackGasLimit)
    external payable returns (bytes32 requestId);

function reveal(bytes32 requestId, bytes32 userRandomness, bytes calldata providerRandomness) external;

function getFee(address caller) external view returns (uint256 fee);
function getProviderInfo(address provider) external view returns (ProviderInfo memory info);
function getProof(bytes32 requestId) external view returns (Proof memory proof);
function pendingRequestBlock(bytes32 requestId) external view returns (uint64 requestBlock);
function REVEAL_GRACE() external view returns (uint64 graceBlocks);   // == 64
function withdrawableFees() external view returns (uint256 amount);
function claimRefund() external;                                      // pull a deferred refund
```

```solidity
struct ProviderInfo { address provider; uint256 fee; uint256 totalRequests; }

struct Proof {
    bytes32 requestId;
    address requester;
    uint64  effectiveSeq;
    bytes32 beaconValue;
    uint64  beaconRound;
    bytes32 userRandomness;
    bytes32 randomValue;
}
```

See [Requesting randomness](requesting-randomness.md) and [Verifying randomness](verifying-randomness.md).

## Consumer callback

Implemented **by your contract** — a plain external function matching this signature (canonically declared as `ILootBoxCallback`; you don't need to import a lootbox-named interface, just match the shape). Called by `NativeVRNG`.

```solidity
function fulfillRandomness(bytes32 requestId, bytes32 randomValue) external;
//   randomValue = keccak256(abi.encode(beaconValue, requestId, userRandomness, effectiveSeq, requester))
//   Guard: require(msg.sender == address(nativeVRNG))
```

> **About the name `fulfillRandomness`.** The protocol deliberately supports two established consumer interfaces so apps can migrate without reshaping their integration:
> - **Proof-of-Play style** — `requestRandom` + your `fulfillRandomness(bytes32 requestId, bytes32 randomValue)` callback (this function).
> - **Pyth Entropy style** — `requestCommitReveal` + finalization via [`reveal`](requesting-randomness.md#finalizing-with-reveal).
>
> The default instant flow (`requestRandomness`) also delivers through this `fulfillRandomness` callback.
>
> Separately, `NativeVRNG` itself exposes **relayer-only** `fulfillRandomness(requestId, seq)` (single) and `fulfillRandomness(requestId[], seq[])` (batch) entry points — these are the fulfillment *trigger* (`FULFILLER_ROLE`), not callbacks; you never call them. Same verb, different role: **yours receives the value; the provider's submits the fulfillment.**

## NativeVRNG — events

```solidity
event RandomnessRequested(bytes32 indexed requestId, address indexed requester, uint64 indexed requestBlock, bytes32 userRandomness);
event RandomnessFulfilled(bytes32 indexed requestId, address indexed requester, bytes32 indexed randomValue, bytes32 beaconValue, uint64 requestBlock, uint64 effectiveSeq);
event CallbackDispatched(bytes32 indexed requestId, address indexed target, bool indexed success);
event RequestCancelled(bytes32 indexed requestId, address indexed requester, uint256 indexed refundAmount);
event MaxCallbackGasUpdated(uint32 indexed oldValue, uint32 indexed newValue);
event FeesWithdrawn(address indexed to, uint256 indexed amount);
event RefundDeferred(bytes32 indexed requestId, address indexed requester, uint256 indexed amount);
event RefundClaimed(address indexed requester, uint256 indexed amount);
```

## NativeVRNG — operator surface

Role-gated; not for integrators. See [Operations & governance](operations.md).

```solidity
function fulfillRandomness(bytes32 requestId, uint64 seq) external;                       // FULFILLER_ROLE (relayer, single)
function fulfillRandomness(bytes32[] requestIds, uint64[] seqs) external returns (uint256); // FULFILLER_ROLE (relayer, batch)
function cancelAndRefund(bytes32 requestId) external;                      // ADMIN_ROLE
function withdrawFees(address to, uint256 amount) external;                // ADMIN_ROLE
function setMaxCallbackGas(uint32 newMax) external;                        // ADMIN_ROLE
function pause() external;                                                 // PAUSER_ROLE
function unpause() external;                                               // ADMIN_ROLE
function getPendingRequest(bytes32 requestId) external view returns (RequestCommitment memory);
```

---

## FeeManager

```solidity
// reads (integrators)
function getFee(address caller) external view returns (uint256 fee);
function getTier(address consumer) external view returns (Tier tier);
function getFeeMode(address consumer) external view returns (FeeMode mode);
function getFeeOverride(address consumer) external view returns (uint256 overrideFee);
function getRateBucket(address subject) external view returns (RateBucket memory bucket);

// public parameters
uint256 public defaultFee;
uint256 public maxFee;        // hard ceiling for every fee setter
uint256 public reducedFee;
uint32  public windowSeconds; // 0 disables rate limiting
uint32  public maxRequests;   // 0 disables rate limiting

// governance (ADMIN_ROLE; PAUSER_ROLE for pause)
function setDefaultFee(uint256 newFee) external;
function setReducedFee(uint256 newFee) external;
function setMaxFee(uint256 newMaxFee) external;
function setFeeOverride(address consumer, uint256 newFee) external;
function setWhitelist(address consumer, Tier tier) external;
function setFeeMode(address consumer, FeeMode mode) external;
function setRateLimitParams(uint32 newWindowSeconds, uint32 newMaxRequests) external;
function pause() external;     // PAUSER_ROLE
function unpause() external;    // ADMIN_ROLE

struct RateBucket { uint32 windowStart; uint32 count; }
```

```solidity
// events
event FeeChanged(uint256 indexed oldFee, uint256 indexed newFee, address indexed scope);
event ReducedFeeChanged(uint256 indexed oldFee, uint256 indexed newFee);
event MaxFeeChanged(uint256 indexed oldFee, uint256 indexed newFee);
event WhitelistUpdated(address indexed consumer, Tier indexed tier);
event FeeModeChanged(address indexed consumer, FeeMode indexed mode);
event RateLimitParamsChanged(uint32 indexed windowSeconds, uint32 indexed maxRequests);
event FeeCollected(bytes32 indexed requestId, address indexed payer, uint256 feeWei, FeeMode mode);
```

Roles: `ADMIN_ROLE` (governance — a Gnosis Safe, optionally behind a Timelock), `PAUSER_ROLE` (a Safe or an admin EOA), `RATE_LIMITER_ROLE` (granted to `NativeVRNG`).

---

## RandomBeaconHistory

Mostly internal (the writer appends; `NativeVRNG` reads). Useful reads for verification:

```solidity
function latestSeq() external view returns (uint64);
function getEntry(uint64 seq) external view returns (Entry memory entry);
function getBeaconValue(uint64 seq) external view returns (bytes32 beaconValue);

struct Entry {
    uint64     drandRound;
    bool       missed;
    uint64     writtenAtBlock;              // set by the contract via arbBlockNumber()
    uint64     prevNonMissedWrittenAtBlock;
    bytes32    drandValue;
    bytes32    beaconValue;                 // the value fed into randomness derivation
    uint256[2] vrfProof;
}
```

```solidity
// events
event BeaconStored(uint64 indexed seq, uint64 indexed drandRound, bytes32 indexed beaconValue, uint64 writtenAtBlock);
event BeaconMissed(uint64 indexed seq);
event SystemWriterChanged(address indexed oldWriter, address indexed newWriter);
event DaemonVrfPubKeyChanged(uint256[4] oldKey, uint256[4] newKey);
```

> `RandomBeaconHistory` is **not pausable** — it is a plain append-only store with no `paused()`. Only `NativeVRNG` and `FeeManager` can be paused.

---

## Constants & enums

```solidity
uint32  DEFAULT_CALLBACK_GAS = 250_000;  // callback budget when callbackGasLimit == 0
uint64  REVEAL_GRACE         = 64;        // blocks after requestBlock reserved to the original requester
// NativeVRNG.maxCallbackGas defaults to 250_000 (governance-settable cap; maxCallbackGas >= DEFAULT_CALLBACK_GAS)

enum Tier    { NONE, REDUCED, WAIVER }                 // FeeManager
enum FeeMode { USER_PAID, DEV_PAID, PROTOCOL_SUBSIDISED } // FeeManager (accounting metadata only)
```

## Common reverts

- Sending `msg.value < getFee(caller)` → request reverts (insufficient fee).
- `callbackGasLimit > maxCallbackGas` → request rejected.
- `reveal` with a `userRandomness` that doesn't hash to the committed `userCommitment` → `InvalidReveal`.
- `getProof` / reveal for an unknown or unfulfilled `requestId` → `UnknownRequest`.

*For the exact error selectors and the full operator surface, consult the contract source (`contracts/core/*.sol`, `contracts/libraries/Errors.sol`) — the source of truth.*
