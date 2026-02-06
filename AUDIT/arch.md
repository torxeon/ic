# Internet Computer Protocol — Deep Architecture Analysis

> This document is a comprehensive architectural reference for the IC codebase at `github.com/dfinity/ic`.
> Every claim is traceable to source code paths or official documentation.
> Written for security researchers, auditors, and protocol engineers.

---

## 1. What the Internet Computer Actually Is

The Internet Computer (IC) is a distributed protocol that extends the public internet with compute capability. Unlike Ethereum's single global state machine, the IC is a **network of sovereign blockchains (subnets)** that communicate asynchronously. Each subnet is an independent replicated state machine running BFT consensus among 13–40 geographically distributed nodes.

The "replica" is the node software — implemented in Rust (~72 crates under `rs/`) — that every node on every subnet runs. It is the entire protocol in one binary.

**Source:** `rs/replica/bin/replica/main.rs` (entry point), `README.adoc` (overview)

---

## 2. The Four-Layer Protocol Stack

Every subnet runs the same four-layer stack. Messages flow top-to-bottom on ingress, bottom-to-top on egress:

```
┌──────────────────────────────────────────────┐
│  Layer 4: EXECUTION                          │
│  Canister WASM execution, System API,        │
│  cycles accounting                           │
│  rs/execution_environment/, rs/embedders/,   │
│  rs/canister_sandbox/                        │
├──────────────────────────────────────────────┤
│  Layer 3: MESSAGE ROUTING                    │
│  Induction, scheduling, cross-subnet (xnet)  │
│  rs/messaging/, rs/xnet/                     │
├──────────────────────────────────────────────┤
│  Layer 2: CONSENSUS                          │
│  Block making, notarization, finalization,   │
│  random beacon, DKG, threshold sigs          │
│  rs/consensus/, rs/crypto/                   │
├──────────────────────────────────────────────┤
│  Layer 1: PEER-TO-PEER                       │
│  QUIC transport, artifact gossip,            │
│  state sync                                  │
│  rs/p2p/                                     │
└──────────────────────────────────────────────┘
```

**Source:** [Network Architecture — internetcomputer.org](https://internetcomputer.org/docs/building-apps/essentials/network-overview), [IC Interface Specification](https://internetcomputer.org/docs/references/ic-interface-spec)

---

## 3. Consensus Layer — The Heart

### 3.1 Internet Computer Consensus Protocol (ICCP)

IC consensus is a **novel BFT protocol** (not Raft, not PBFT, not Tendermint). It uses a ranked committee leader model with cryptographic finality. Published formally in [ePrint 2021/632](https://eprint.iacr.org/2021/632.pdf).

**Core flow per round:**

1. **Random Beacon** — A threshold BLS12-381 signature on the previous beacon produces an unbiased random value. Each node broadcasts a signature share; `t+1` shares are aggregated into the beacon. This drives leader election.
   - Type: `RandomBeacon = Signed<RandomBeaconContent, ThresholdSignature<RandomBeaconContent>>`
   - Source: `rs/types/types/src/consensus.rs`

2. **Block Making** — Ranked block makers (determined by random beacon) propose blocks containing a `Payload` of ingress messages, xnet streams, and DKG dealings.
   - Type: `Block { version, parent, payload, height, rank, context: ValidationContext }`
   - Source: `rs/types/types/src/consensus.rs`

3. **Notarization** — Nodes validate a proposed block and broadcast BLS multi-signature shares. A block is notarized when ≥2/3 of subnet nodes sign it. Shares aggregate into a compact notarization.
   - Type: `Notarization = Signed<NotarizationContent, MultiSignature<NotarizationContent>>`
   - Source: `rs/types/types/src/consensus.rs`

4. **Finalization** — If a node only notarized ONE block at a given height, it also signs a finalization share. ≥2/3 support finalizes the block. This provides **cryptographic finality** (no forks after finalization).
   - Type: `Finalization = Signed<FinalizationContent, MultiSignature<FinalizationContent>>`

5. **Catch-Up Package (CUP)** — Periodic snapshots containing the finalized block, random beacon, and certified state hash. Allows nodes to sync without replaying from genesis.
   - Type: `CatchUpContent { version, block, random_beacon, state_hash, oldest_registry_version_in_use }`
   - Source: `rs/types/types/src/consensus/catchup.rs`

### 3.2 Distributed Key Generation (DKG)

The protocol uses **Non-Interactive Distributed Key Generation (NI-DKG)** to establish threshold BLS signing keys for each subnet without requiring synchronous rounds. DKG dealings are included as consensus payload.

**Source:** `rs/consensus/` (DKG submodule), DFINITY blog: [NI-DKG](https://medium.com/dfinity/applied-crypto-one-public-key-for-the-internet-computer-ni-dkg-4af800db869d)

### 3.3 Threshold Signatures — Chain-Key Cryptography

Each subnet has a single public key. The corresponding secret key is **never materialized** — it is secret-shared across all subnet nodes. Threshold signing (BLS for internal consensus, ECDSA/Schnorr for external chains) allows the subnet to sign as one entity.

- **Consensus signatures:** Threshold BLS12-381
- **Chain-key signatures (Bitcoin/Ethereum):** Threshold ECDSA (secp256k1), Threshold Schnorr
- **VetKD:** Verifiable Encrypted Threshold Key Derivation (newer addition)

**Source:** `rs/crypto/` (entire subtree), `rs/crypto/internal/` (implementations)

---

## 4. Execution Layer — Canister Runtime

### 4.1 Canister Model

Canisters are the IC's smart contracts. They are **actor-model processes** running WebAssembly:

- Each canister has its own WASM heap memory (max 4 GiB, default limit 3 GiB) and stable memory (max 500 GiB)
- Canisters process messages **sequentially** (one message at a time) — no reentrancy within a single canister
- Inter-canister calls are **asynchronous** — they create yield points where other messages can interleave
- Fuel: **Cycles** (analogous to gas), burned on compute, storage, and bandwidth

**Key constants** (from `rs/config/src/subnet_config.rs`):

| Constant | Value |
|---|---|
| `MAX_INSTRUCTIONS_PER_MESSAGE` | 40 billion |
| `MAX_INSTRUCTIONS_PER_ROUND` | 4 billion |
| `MAX_INSTRUCTIONS_PER_QUERY` | 5 billion |
| `MAX_INSTRUCTIONS_PER_INSTALL_CODE` | 300 billion |
| `MAX_INSTRUCTIONS_PER_SLICE` (DTS) | 2 billion |
| `MAX_WASM_MEMORY` | 4 GiB |
| `MAX_WASM64_MEMORY` | 6 GiB |
| `MAX_STABLE_MEMORY` | 500 GiB |
| `MAX_INTER_CANISTER_PAYLOAD` | 2 MiB |

**Source:** `rs/config/src/subnet_config.rs`, `rs/types/types/src/lib.rs`

### 4.2 WASM Instrumentation & Metering

Before execution, canister WASM binaries are **instrumented** by the embedder:

- A global instruction counter is injected
- At each basic block entry, the counter is decremented by the block's cost
- If the counter goes negative → trap (`InstructionLimitExceeded`)
- Supports **Deterministic Time Slicing (DTS)** — long computations are broken into 2B-instruction slices

**Instruction costs** (from `rs/embedders/src/wasm_utils/instrumentation.rs`):

| Operation | Cost |
|---|---|
| Arithmetic (i32/i64 add, mul, etc.) | 1 |
| Division / Remainder | 10 |
| Branch | 2 |
| Call | 20 |
| Load/Store | 1 |
| System API calls | 500–10,000 (varies) |

**Source:** `rs/embedders/src/wasm_utils/instrumentation.rs`, `rs/embedders/src/wasmtime_embedder/system_api_complexity.rs`

### 4.3 Canister Sandbox

Each canister executes in a **separate OS process** for crash isolation:

- Replica ↔ Sandbox communication via IPC
- Protocol defined in `rs/canister_sandbox/common/src/protocol/`
- Sandbox process managed by `rs/canister_sandbox/replica_controller/`

**Source:** `rs/canister_sandbox/README.md`

### 4.4 Cycles Account Manager

Cycles are the resource accounting unit. The lifecycle for every operation:

1. **Reserve** — Estimate max cycles needed, deduct from canister balance
2. **Execute** — Track actual consumption
3. **Refund** — Return `reserved - spent` to canister

Resource pricing scales with **subnet saturation** via `ResourceSaturation { usage, threshold, capacity }`:
- Below threshold: base price
- Above threshold: price scales linearly up to capacity
- At capacity: maximum price

**Source:** `rs/cycles_account_manager/src/lib.rs`

---

## 5. Message Routing & Cross-Subnet Communication

### 5.1 Intra-Subnet

Consensus produces ordered batches. Message routing places messages into target canister input queues (**induction**). Execution processes them. Output messages go to output queues, then back through routing.

### 5.2 Cross-Subnet (XNet)

Subnets communicate via **certified streams**:

```rust
pub struct XNetPayload {
    pub stream_slices: BTreeMap<SubnetId, CertifiedStreamSlice>,
}
```

Each stream slice is signed by the source subnet's threshold key → the receiving subnet can verify authenticity without trusting individual nodes.

**Source:** `rs/types/types/src/batch/xnet.rs`, `rs/xnet/`, `rs/messaging/`

---

## 6. State Management & Certification

### 6.1 Replicated State

The complete state of a subnet includes all canister states, system metadata, message queues, and ingress history. It is deterministic — all honest nodes in a subnet hold identical state.

### 6.2 State Certification via Merkle Trees

After each round, the state manager computes a **hash tree** (Merkle tree) over the canonical state using SHA-256 (`HASH_LENGTH = 32 bytes`). The root hash is threshold-signed by the subnet → anyone can verify state authenticity with just the subnet's public key.

**Source:** `rs/state_manager/src/tree_hash.rs`, `rs/canonical_state/`, `rs/crypto/tree_hash/`

### 6.3 State Sync

New or lagging nodes use the P2P state sync protocol to download state chunks (Merkle tree streaming) rather than replaying all blocks from genesis.

**Source:** `rs/p2p/state_sync_manager/`

---

## 7. Peer-to-Peer Layer

### 7.1 Transport

QUIC-based transport (`rs/p2p/quic_transport/`):
- Max message size: **128 MiB**
- Request/response multiplexing over substreams
- Automatic peer discovery and connection management

### 7.2 Artifact Propagation

Nodes gossip **artifacts** (consensus messages, DKG dealings, ingress messages, state sync chunks) using a priority-based delivery system with backpressure.

**Source:** `rs/p2p/artifact_downloader/`, `rs/p2p/artifact_manager/`, `rs/p2p/consensus_manager/`

---

## 8. Cryptography Component

### 8.1 Architecture

The crypto component (`rs/crypto/`) provides a unified interface for all cryptographic operations. Secret keys are managed by the **Crypto Service Provider (CSP)** vault — keys never leave the vault in plaintext.

### 8.2 Algorithms Used

| Purpose | Algorithm | Source |
|---|---|---|
| Node basic signatures | Ed25519 | `rs/crypto/internal/basic_sig/ed25519/` |
| Canister signatures | ECDSA (secp256k1, P-256) | `rs/crypto/internal/basic_sig/ecdsa_secp256k1/` |
| Consensus threshold sigs | BLS12-381 | `rs/crypto/internal/bls12_381/` |
| Chain-key (Bitcoin) | Threshold ECDSA (secp256k1) | `rs/crypto/internal/threshold_sig/` |
| TLS | Rustls + Ring | `rs/crypto/internal/tls/` |
| Hashing | SHA-256, SHA-224, SHA-3 | `rs/crypto/sha/` |
| State certification | Merkle hash trees (SHA-256) | `rs/crypto/tree_hash/` |
| DKG | NI-DKG (BLS-based) | `rs/consensus/` DKG module |

### 8.3 Node Key Types

Each node holds multiple key pairs:
1. **Node signing key** — Ed25519 for basic authentication
2. **Committee signing key** — BLS12-381 for consensus participation
3. **DKG dealing encryption key** — For encrypted DKG dealings
4. **TLS certificate** — For secure node-to-node communication
5. **I-DKG key** — For interactive DKG in threshold ECDSA

**Source:** `rs/crypto/node_key_generation/`, `rs/crypto/node_key_validation/`

---

## 9. Governance — Network Nervous System (NNS)

The NNS is a set of **system canisters** running on a dedicated subnet that govern the entire IC.

### 9.1 NNS Canisters

| Canister | Role | Source |
|---|---|---|
| **Governance** | Proposal submission, voting, neuron management | `rs/nns/governance/` |
| **Registry** | Global IC configuration (subnets, nodes, keys) | `rs/registry/` |
| **Ledger** | ICP token transfers | `rs/ledger_suite/icp/` |
| **Root** | Upgrade coordination for NNS canisters | `rs/nns/root/` |
| **CMC** | Cycles Minting Canister — ICP→Cycles conversion | `rs/nns/cmc/` |
| **Lifeline** | Emergency recovery control | `rs/nns/handlers/lifeline/` |
| **SNS-WASM** | SNS deployment factory | `rs/nns/sns-wasm/` |

### 9.2 Neuron Economics & Voting Power

**Neurons** are the governance primitive. Users lock ICP tokens into neurons to gain voting power and earn rewards.

**Neuron struct** (from `rs/nns/governance/src/neuron/types.rs`):
```
Neuron {
    id: NeuronId,
    subaccount: Subaccount (32 bytes),
    controller: PrincipalId,
    dissolve_state_and_age: DissolveStateAndAge,
    cached_neuron_stake_e8s: u64,
    neuron_fees_e8s: u64,
    maturity_e8s_equivalent: u64,
    staked_maturity_e8s_equivalent: Option<u64>,
    followees: HashMap<i32, Followees>,  // per-topic following
    kyc_verified: bool,
    neuron_type: Option<i32>,
}
```

**Voting Power Formula** (from `rs/nns/governance/src/neuron/types.rs:340`):

```
Voting Power = Stake × Dissolve Delay Bonus × Age Bonus

Where:
  Dissolve Delay Bonus = 1 + (dissolve_delay / MAX_DISSOLVE_DELAY)
    → ranges from 1.0x to 2.0x (at 8 years)

  Age Bonus = 1 + (age / (4 × MAX_AGE))
    → ranges from 1.0x to 1.25x (at 4 years)

  Maximum combined multiplier: 2.0 × 1.25 = 2.5x
```

**Key economic constants:**
- `MAX_DISSOLVE_DELAY`: 8 years
- `MAX_NEURON_AGE_FOR_AGE_BONUS`: 4 years
- `neuron_minimum_stake_e8s`: 1 ICP (100,000,000 e8s)
- `neuron_minimum_dissolve_delay_to_vote`: 6 months
- `reject_cost_e8s`: 25 ICP on mainnet (spam prevention)
- `transaction_fee_e8s`: 10,000 e8s (ICP ledger fee)
- `neuron_spawn_dissolve_delay`: 7 days

**Source:** `rs/nns/governance/src/neuron/types.rs`, `rs/nns/governance/src/network_economics.rs`, `rs/nns/governance/api/src/lib.rs`

### 9.3 Liquid Democracy

Neurons can **follow** other neurons on a per-topic basis. If a majority of a neuron's followees vote the same way, the follower's vote is automatically cast. This creates a delegation graph.

### 9.4 Voting Rewards

Rewards are distributed daily from a decaying reward pool:
- **Initial rate:** 10% per year
- **Final rate:** 5% per year (after 8 years from genesis)
- Quadratic decay between initial and final rates
- Distributed as **maturity** (not directly as ICP)
- Maturity can be converted to ICP via **spawning** (7-day delay) or **disbursement** (7-day delay, recommended)
- Maturity modulation adjusts the conversion rate

**Source:** `rs/nns/governance/src/reward/calculation.rs`, `rs/nns/governance/src/reward/distribution.rs`

### 9.5 Neurons' Fund

Neurons that opt in to the Neurons' Fund automatically co-invest in SNS token swaps:
- Max participation: 10% of maturity (1,000 basis points)
- Polynomial matching function scales with direct investor participation
- Contribution threshold: 75,000 XDR

**Source:** `rs/nns/governance/src/neurons_fund.rs`

---

## 10. Service Nervous System (SNS)

The SNS is a **DAO-in-a-box** framework — any dapp can decentralize its governance by launching an SNS.

### 10.1 SNS Canisters

Each SNS deployment includes: Governance, Root, Swap, Ledger (ICRC-1), and Index canisters.

### 10.2 Token Swap Mechanism

```
Lifecycle: Init → Open → Committed/Aborted → Settled
```

- Direct participants: send ICP, receive SNS tokens + neurons
- Neurons' Fund: matched participation (polynomial scaling)
- Max direct participants: 20,000
- Neuron basket creation in batches of 500

**Source:** `rs/sns/swap/src/swap.rs`

---

## 11. Cycles Minting Canister (CMC) — ICP→Cycles Economics

### 11.1 Conversion Mechanics

```
Cycles = ICP_amount × ICP_XDR_rate × CYCLES_PER_XDR

Where:
  CYCLES_PER_XDR = 1,000,000,000,000 (1 trillion)
  ICP_XDR_rate = 30-day moving average from exchange rate canister
```

### 11.2 Key Parameters

- Rate refresh interval: 5 minutes
- Rate cache: 60 days
- 30-day moving average for stability
- Minimum 4 XDR sources, minimum 4 CXDR sources
- Create canister refund fee: 4× ledger fee (40,000 e8s)
- Create canister minimum cycles: 100 billion
- Monthly minting limit: 150 × 10^15 cycles (default)

**Source:** `rs/nns/cmc/src/lib.rs`, `rs/nns/cmc/src/exchange_rate_canister.rs`

---

## 12. Chain Fusion — Cross-Chain Integration

### 12.1 Architecture

Instead of bridges (which require trusted intermediaries), IC uses **threshold signatures** to directly control accounts on external chains. A subnet collectively signs Bitcoin/Ethereum transactions using threshold ECDSA without any single node ever holding the full key.

### 12.2 ckBTC (Chain-Key Bitcoin)

- Minter canister at `rs/bitcoin/ckbtc/minter/`
- Monitors Bitcoin network for deposits to IC-controlled addresses
- Mints ckBTC (ICRC-1 token) 1:1 for Bitcoin deposits
- Burns ckBTC and signs Bitcoin withdrawal transactions via threshold ECDSA
- UTXO management with consolidation at >1,000 UTXOs
- Min resubmission delay: 24 hours
- Max requests per batch: 100

### 12.3 ckETH / ckERC20

- Minter canister at `rs/ethereum/cketh/minter/`
- Same pattern: deposits monitored, ckETH minted, withdrawals signed via threshold ECDSA
- Supports arbitrary ERC-20 tokens as ckERC20

**Source:** `rs/bitcoin/ckbtc/minter/src/lib.rs`, `rs/ethereum/cketh/minter/src/lib.rs`

---

## 13. Ledger Suite

### 13.1 ICP Ledger

The native ICP token ledger at `rs/ledger_suite/icp/`:
- AccountIdentifier = hash(PrincipalId + Subaccount)
- Fixed transaction fee: 10,000 e8s
- Supports ICRC-1/ICRC-2 standards (transfer, approve, transferFrom)
- Archive canisters for historical blocks (triggered at configurable thresholds)

### 13.2 ICRC-1 Ledger

Generic token ledger at `rs/ledger_suite/icrc1/`:
- Supports both U64 and U256 token precision
- Used by ckBTC, ckETH, ckERC20, and SNS tokens
- Multiple archive canisters with block-range tracking
- Default archive cycles: 10 trillion

### 13.3 Block & Archive Design

```
BlockType trait:
  from_transaction(parent_hash, tx, timestamp, effective_fee, fee_collector)
  encode() → EncodedBlock (binary)
  decode(EncodedBlock) → Self
  block_hash(encoded) → HashOf<EncodedBlock>
```

Archive trigger: when block count exceeds threshold, blocks are moved to archive canisters. An archiving guard prevents concurrent archiving operations.

**Source:** `rs/ledger_suite/common/ledger_canister_core/src/ledger.rs`, `rs/ledger_suite/common/ledger_canister_core/src/archive.rs`

---

## 14. Internet Identity

A decentralized authentication system using WebAuthn/FIDO2:
- Users create an "anchor" (identity)
- Authenticate via biometrics/hardware keys
- Generates unique pseudonymous principals per dapp (privacy-preserving)
- No passwords, no seed phrases for end users

**Source:** `https://github.com/dfinity/internet-identity` (separate repo, in scope)

---

## 15. Infrastructure: IC-OS

The IC runs on a custom three-tier operating system stack:

1. **SetupOS** — Initial node setup and provisioning
2. **HostOS** — Bare-metal host running on node hardware
3. **GuestOS** — Virtual machine running the replica binary

Each OS image is built reproducibly and upgraded via NNS proposals.

**Source:** `ic-os/` directory, `ic-os/README.adoc`

---

## 16. Key Identifiers

| Type | Definition | Source |
|---|---|---|
| `NodeId` | `Id<NodeTag, PrincipalId>` | `rs/types/base_types/src/lib.rs` |
| `SubnetId` | `Id<SubnetTag, PrincipalId>` | `rs/types/base_types/src/lib.rs` |
| `CanisterId` | Wrapper around `PrincipalId` | `rs/types/base_types/` |
| `RegistryVersion` | `AmountOf<RegistryVersionTag, u64>` | `rs/types/base_types/src/lib.rs` |
| `Height` | Block height (consensus round) | `rs/types/types/` |
| `Rank` | Block maker rank (from random beacon) | `rs/types/types/` |

---

## 17. Build System

- **Bazel** — Primary build system for reproducible builds (`MODULE.bazel`, `bazel/`)
- **Cargo** — Rust package manager (workspace with 532 members)
- Rust edition: 2024
- Build profiles: `release`, `release-stripped`, `release-lto`, `canister-release`

---

## 18. Testing Infrastructure

| Level | Location | Description |
|---|---|---|
| Unit tests | Within each crate | Standard `#[test]` |
| Integration tests | `rs/tests/` | System tests via `system_test` Bazel macro |
| State machine tests | `rs/state_machine_tests/` | Deterministic execution testing |
| Consensus simulation | `rs/consensus/` (test framework) | Multi-node simulation without real network |
| Fuzzers | `rs/fuzzers/` | Fuzz testing for parsers and protocols |
| Replica tests | `rs/replica_tests/` | Full replica integration |
| TLA+ | `rs/tla_instrumentation/` | Formal verification instrumentation |

---

## 19. Security Model Summary

**Trust assumptions:**
- Each subnet requires ≥2/3 honest nodes for safety
- The NNS subnet is the root of trust for the entire network
- Registry (on NNS) defines subnet membership and keys
- Nodes are operated by independent, vetted node providers
- Canister controllers can upgrade canister code (unless control is renounced or given to an SNS)

**Key security boundaries:**
- Consensus: Byzantine fault tolerance with 1/3 corruption threshold
- Cryptography: Threshold signatures prevent single-node key compromise
- Execution: Process-level sandbox isolation per canister
- State: Merkle-tree certification allows external verification
- Cross-chain: No bridges — threshold signatures control external accounts directly

---

*Document generated from source analysis of the `dfinity/ic` repository. All file paths are relative to the repository root. Cross-referenced with [IC documentation](https://internetcomputer.org/docs/), [IC Interface Specification](https://internetcomputer.org/docs/references/ic-interface-spec), and [IC Consensus paper (ePrint 2021/632)](https://eprint.iacr.org/2021/632.pdf).*
