# Gloas Devnet-0: How It Works

> This document describes the devnet-0 experience from the ground up: what
> actually happens at each step, visualized, then mapped back to the spec items
> that make each step work.
>
> **Devnet-0 simplification**: self-build only (no external builders), no sync,
> no full fork-choice enforcement, PTC attestations optional, no blobs/DataColumnSidecars.

---

## 1. The Big Picture

In Gloas, **the block and the payload are separate objects**. The proposer
broadcasts the beacon block first (with a *bid* promising a payload), and then
broadcasts the execution payload envelope as a second message. Everyone else
watches for both, attests to what they saw, and some validators (the PTC)
specifically attest to whether the payload arrived on time.

> **Devnet-0 note:** This diagram shows the full Gloas slot structure. For
> devnet-0, the PTC attestation step at T=9s is optional — teams may skip it
> entirely. The core devnet-0 flow is just: block+bid at T=0s → payload
> envelope shortly after → regular attestations at T=3s with `index` signaling
> payload status.

```
┌─────────────────────────────── Slot N (12 seconds) ───────────────────────────────┐
│                                           (not required for devnet-0)              │
│  T=0s         T=3s              T=6s              T=9s              T=12s          │
│  │            │                 │                 │                 │              │
│  ▼            ▼                 ▼                 ▼                 ▼              │
│  ┌──────┐    ┌──────────┐      ┌──────────┐      ┌──────────┐                     │
│  │BLOCK │    │ATTEST    │      │AGGREGATE │      │PTC ATTEST│     next slot        │
│  │ + BID│    │(25%)     │      │(50%)     │      │(75%)     │     begins           │
│  └──┬───┘    └──────────┘      └──────────┘      └──────────┘                     │
│     │                                                                             │
│     │  ┌─────────────────┐                                                        │
│     └─►│PAYLOAD ENVELOPE  │  (broadcast shortly after block)                      │
│        └─────────────────┘                                                        │
│                                                                                   │
└───────────────────────────────────────────────────────────────────────────────────┘
```

**Spec items enabling the timeline:**
- Timing: `ATTESTATION_DUE_BPS_GLOAS` (2500 = 25%), `AGGREGATE_DUE_BPS_GLOAS` (5000 = 50%) — [#16291](https://github.com/OffchainLabs/prysm/pull/16291) ✅
- PTC timing: `PAYLOAD_ATTESTATION_DUE_BPS` (7500 = 75%) — ❌

---

## 2. Flow A: Proposer Creates & Broadcasts Block + Payload

This is what the proposer does when it's their turn.

```
  PROPOSER (Slot N)
  ═════════════════

  ┌─────────────────────────────────────────────────────────┐
  │ 1. I'm the proposer for slot N                         │
  │                                                         │
  │ 2. Ask EL: "build me a payload"                        │
  │    ┌──────────────────────────────────────┐             │
  │    │ prepare_execution_payload()           │             │
  │    │   parent = state.latest_block_hash    │             │
  │    │   → EL returns PayloadId              │             │
  │    └──────────────────────────────────────┘             │
  │                                                         │
  │ 3. Create self-build bid                                │
  │    ┌──────────────────────────────────────┐             │
  │    │ bid.builder_index = SELF_BUILD (2^64) │             │
  │    │ bid.value = 0                         │             │
  │    │ bid.slot = N                          │             │
  │    │ bid.parent_block_hash = state hash    │             │
  │    │ bid.parent_block_root = parent root   │             │
  │    │ bid.block_hash = payload.block_hash   │             │
  │    │ signature = G2_POINT_AT_INFINITY      │             │
  │    └──────────────────────────────────────┘             │
  │                                                         │
  │ 4. Build BeaconBlock                                    │
  │    ┌──────────────────────────────────────┐             │
  │    │ body.signed_execution_payload_bid ←── self-build   │
  │    │ body.payload_attestations ←────────── [] (empty)   │
  │    │ body has NO execution_payload field   │             │
  │    │ (attestations, sync, exits... normal) │             │
  │    └──────────────────────────────────────┘             │
  │                                                         │
  │ 5. Sign & broadcast SignedBeaconBlock                   │
  │    ════════════════════════════════════                  │
  │    gossip topic: beacon_block                           │
  │                                                         │
  │ 6. Build ExecutionPayloadEnvelope                       │
  │    ┌──────────────────────────────────────┐             │
  │    │ envelope.payload = the EL payload     │             │
  │    │ envelope.execution_requests = reqs    │             │
  │    │ envelope.builder_index = SELF_BUILD   │             │
  │    │ envelope.beacon_block_root = block    │             │
  │    │ envelope.slot = N                     │             │
  │    │ Run process_execution_payload()       │             │
  │    │   → compute state_root                │             │
  │    │ envelope.state_root = result          │             │
  │    │ Sign with proposer key                │             │
  │    └──────────────────────────────────────┘             │
  │                                                         │
  │ 7. Broadcast SignedExecutionPayloadEnvelope             │
  │    ════════════════════════════════════════              │
  │    gossip topic: execution_payload                      │
  │                                                         │
  └─────────────────────────────────────────────────────────┘
```

### Spec items for this flow

| Step | What | Spec Item | Prysm Status |
|------|------|-----------|:------------:|
| 2 | Ask EL for payload | `prepare_execution_payload` (modified — uses `state.latest_block_hash`) | ❌ |
| 3 | Self-build bid | Constructing `signed_execution_payload_bid` with `BUILDER_INDEX_SELF_BUILD` | 🔄 [#16336](https://github.com/OffchainLabs/prysm/pull/16336) |
| 4 | Block body structure | Modified `BeaconBlockBody` — no exec payload; has bid + payload attestations | ✅ [#15618](https://github.com/OffchainLabs/prysm/pull/15618) |
| 4 | Payload attestations list | `MAX_PAYLOAD_ATTESTATIONS` preset (4) | ❌ |
| 5 | Block gossip | Gossip: `beacon_block` — modified validation (no payload checks, add bid checks) | ❌ |
| 6 | Envelope construction | Constructing `SignedExecutionPayloadEnvelope` | 🔄 [#16336](https://github.com/OffchainLabs/prysm/pull/16336) |
| 6 | Compute state root | `process_execution_payload` + `ApplyExecutionPayload` | ✅ [#15656](https://github.com/OffchainLabs/prysm/pull/15656) [#16356](https://github.com/OffchainLabs/prysm/pull/16356) |
| 6 | Verify envelope sig | `verify_execution_payload_envelope_signature` | ✅ [#15656](https://github.com/OffchainLabs/prysm/pull/15656) |
| 7 | Payload gossip | Gossip: `execution_payload` topic | ✅ [#16349](https://github.com/OffchainLabs/prysm/pull/16349) |
| — | Get block (gRPC + REST) | `GetBeaconBlock` adds Gloas case + `ProduceBlockV4` REST | 🔄 [#16336](https://github.com/OffchainLabs/prysm/pull/16336) |
| — | Publish block | `ProposeBlock` adds Gloas case | 🔄 [#16336](https://github.com/OffchainLabs/prysm/pull/16336) |
| — | Get envelope (gRPC + REST) | New `ExecutionPayloadEnvelope` RPC + REST handler | 🔄 [#16336](https://github.com/OffchainLabs/prysm/pull/16336) |
| — | Publish envelope (gRPC + REST) | New `PublishExecutionPayloadEnvelope` RPC + REST handler | 🔄 [#16336](https://github.com/OffchainLabs/prysm/pull/16336) |

---

## 3. Flow B: Node Receives Block

When a node receives a beacon block from gossip:

```
  NODE receives SignedBeaconBlock
  ═══════════════════════════════

  gossip topic: beacon_block
         │
         ▼
  ┌──────────────────────────────────────────────────────┐
  │                 GOSSIP VALIDATION                     │
  │                                                       │
  │  • Is this a valid block structure?                   │
  │  • Is the proposer correct for this slot?             │
  │  • Is the bid consistent?                             │
  │    - bid.slot matches block.slot                      │
  │    - bid.parent_block_hash matches state              │
  │    - bid.parent_block_root matches block.parent_root  │
  │    - For self-build: sig == G2∞, value == 0           │
  │                                                       │
  └───────────────┬──────────────────────────────────────┘
                  │ valid
                  ▼
  ┌──────────────────────────────────────────────────────┐
  │              STATE TRANSITION                         │
  │                                                       │
  │  process_slot()                                       │
  │    └─► Clear payload availability for this slot       │
  │                                                       │
  │  process_operations()                                 │
  │    ├─► process_execution_payload_bid()                │
  │    │     Store the bid commitment in state             │
  │    ├─► process_attestation() [for each attestation]   │
  │    │     index=0 → empty/same-slot                    │
  │    │     index=1 → full (payload was present)         │
  │    ├─► process_payload_attestation() [if any in block]│
  │    │     Validate PTC votes from previous slot        │
  │    └─► (other ops: slashings, exits, deposits...)     │
  │                                                       │
  │  process_withdrawals()                                │
  │    └─► If parent was EMPTY → return early (skip)      │
  │        If parent was FULL  → process normally          │
  │                                                       │
  │  process_epoch() [if epoch boundary]                  │
  │    └─► process_builder_pending_payments()             │
  │                                                       │
  └───────────────┬──────────────────────────────────────┘
                  │
                  ▼
        Block accepted. But it's "EMPTY" —
        no execution payload yet.
        Waiting for the envelope...
```

### Spec items for this flow

| Step | What | Spec Item | Prysm Status |
|------|------|-----------|:------------:|
| Gossip | Block validation | Modified `beacon_block` gossip validation | ❌ |
| Slot | Clear payload avail | Modified `process_slot` | ✅ [#15730](https://github.com/OffchainLabs/prysm/pull/15730) |
| Ops | Process bid | `process_execution_payload_bid` | ✅ [#15638](https://github.com/OffchainLabs/prysm/pull/15638) |
| Ops | Wire all ops | Modified `process_operations` — wire PTC + bid | ❌ |
| Ops | Process attestation | Modified `process_attestation` — `index` for payload | ✅ [#15736](https://github.com/OffchainLabs/prysm/pull/15736) |
| Ops | Flag indices | `get_attestation_participation_flag_indices` (modified) | ✅ [#15736](https://github.com/OffchainLabs/prysm/pull/15736) |
| Ops | PTC processing | `process_payload_attestation` | ✅ [#15650](https://github.com/OffchainLabs/prysm/pull/15650) |
| Ops | Proposer slashing | Modified `process_proposer_slashing` | ✅ [#16212](https://github.com/OffchainLabs/prysm/pull/16212) |
| Wdraw | Parent check | Modified `process_withdrawals` — early return if parent empty | 🔄 [#16310](https://github.com/OffchainLabs/prysm/pull/16310) |
| Wdraw | Expected wdraws | Modified `get_expected_withdrawals` | 🔄 [#16310](https://github.com/OffchainLabs/prysm/pull/16310) |
| Epoch | Pending payments | `process_builder_pending_payments` (wired in epoch) | ✅ impl, ❌ wiring [#15655](https://github.com/OffchainLabs/prysm/pull/15655) |

---

## 4. Flow C: Node Receives Payload Envelope

The payload arrives as a separate gossip message, typically shortly after the block:

```
  NODE receives SignedExecutionPayloadEnvelope
  ════════════════════════════════════════════

  gossip topic: execution_payload
         │
         ▼
  ┌────────────────────────────────────────────────────────┐
  │                 GOSSIP VALIDATION                       │
  │                                                         │
  │  • Have we seen the referenced block root?              │
  │  • Does the slot match the block's slot?                │
  │  • Does builder_index match the bid's builder_index?    │
  │  • Does payload hash match the bid's block_hash?        │
  │  • Is the envelope signature valid?                     │
  │    (self-build: verify with proposer pubkey)            │
  │  • Haven't we seen another envelope from this builder   │
  │    for this block?                                      │
  │                                                         │
  └──────────────────┬─────────────────────────────────────┘
                     │ valid
                     ▼
  ┌────────────────────────────────────────────────────────┐
  │         EXECUTION PAYLOAD PROCESSING                    │
  │                                                         │
  │  process_execution_payload()                            │
  │    ├─► Verify envelope signature                        │
  │    ├─► Check bid consistency (slot, parent, block_hash) │
  │    ├─► Send payload to EL for validation                │
  │    │     engine.notify_new_payload()                     │
  │    ├─► Process execution requests                       │
  │    │     (deposits, withdrawals, consolidations)         │
  │    ├─► Queue builder payment (0 for self-build)         │
  │    ├─► update_payload_expected_withdrawals()            │
  │    ├─► Set execution_payload_availability = true        │
  │    └─► Verify state_root matches                        │
  │                                                         │
  └──────────────────┬─────────────────────────────────────┘
                     │
                     ▼
        Block is now "FULL" — payload received!
        Parent status for next slot's attesters: FULL
```

### What if the payload never arrives?

```
  ┌──────────────────────────────────────────────────┐
  │  Payload NOT received via gossip                  │
  │                                                   │
  │  Option 1: Request it                             │
  │    ExecutionPayloadEnvelopesByRoot                │
  │    (req/resp protocol)                            │
  │                                                   │
  │  Option 2: It never arrives                       │
  │    Block stays EMPTY                              │
  │    Next slot's attesters set index=0              │
  │    Next slot's proposer:                          │
  │      - process_withdrawals → early return (skip)  │
  │      - Builds on top of empty slot normally       │
  │      - is_parent_block_full = false               │
  │                                                   │
  └──────────────────────────────────────────────────┘
```

### Spec items for this flow

| Step | What | Spec Item | Prysm Status |
|------|------|-----------|:------------:|
| Gossip | Envelope validation | `execution_payload` gossip validation | ✅ [#16349](https://github.com/OffchainLabs/prysm/pull/16349) |
| Gossip | Subscriber pipeline | Envelope subscriber → chain handler | 🔄 stub [#16349](https://github.com/OffchainLabs/prysm/pull/16349) |
| Chain | Chain handler | `ReceiveExecutionPayloadEnvelope` | ❌ (TODO stub) |
| Process | Payload processing | `process_execution_payload` | ✅ [#15656](https://github.com/OffchainLabs/prysm/pull/15656) |
| Process | Refactored apply | `ApplyExecutionPayload` | ✅ [#16356](https://github.com/OffchainLabs/prysm/pull/16356) |
| Process | Expected withdrawals | `update_payload_expected_withdrawals` | 🔄 [#16310](https://github.com/OffchainLabs/prysm/pull/16310) |
| Fallback | Request by root | `ExecutionPayloadEnvelopesByRoot v1` | ❌ |
| Fallback | Request config | `MAX_REQUEST_PAYLOADS` | ❌ |
| State | Parent full check | `is_parent_block_full` | ✅ |
| DB | Persist envelope | DB functions for saving payload | ✅ [#16301](https://github.com/OffchainLabs/prysm/pull/16301) |

---

## 5. Flow D: Attester Creates & Broadcasts Attestation

At T=3s (25% of slot), attesters must broadcast:

```
  ATTESTER (Slot N, attesting to head at Slot N-1)
  ════════════════════════════════════════════════

  ┌────────────────────────────────────────────────┐
  │                                                 │
  │  "What is the payload status of the block       │
  │   I'm attesting to?"                            │
  │                                                 │
  │  Case 1: Attesting to block in SAME slot (N)    │
  │    → data.index = 0 (always)                    │
  │    "Can't know payload status yet"              │
  │                                                 │
  │  Case 2: Attesting to block in slot N-1         │
  │    Did I see the payload for slot N-1?           │
  │                                                 │
  │    ┌───────────┐       ┌───────────┐            │
  │    │ YES (FULL)│       │ NO (EMPTY)│            │
  │    │ index = 1 │       │ index = 0 │            │
  │    └───────────┘       └───────────┘            │
  │                                                 │
  │  Set attestation.data.index accordingly         │
  │  Sign & broadcast on beacon_attestation topic   │
  │                                                 │
  └────────────────────────────────────────────────┘

  Gossip validation (receiver side):
  ┌────────────────────────────────────────────────┐
  │  • committee_index MUST be < 2 (not just 0)    │
  │  • Same-slot attestation: index MUST be 0      │
  │  • Cross-slot: index matches payload status    │
  └────────────────────────────────────────────────┘
```

### Spec items for this flow

| Step | What | Spec Item | Prysm Status |
|------|------|-----------|:------------:|
| Create | Set index field | Validator: modified attestation — `index` signals payload status | ❌ |
| Create | Matching logic | `MatchingPayload()` — same-slot vs cross-slot | ✅ [#15736](https://github.com/OffchainLabs/prysm/pull/15736) |
| Gossip | Index < 2 check | `beacon_attestation` / `beacon_aggregate_and_proof` validation | 🔄 [#16359](https://github.com/OffchainLabs/prysm/pull/16359) |
| Process | Participation flags | `get_attestation_participation_flag_indices` (modified) | ✅ [#15736](https://github.com/OffchainLabs/prysm/pull/15736) |
| Process | Same-slot check | `is_attestation_same_slot` | ✅ |

---

## 6. Flow E: PTC Member Attests to Payload (Optional for Devnet-0)

At T=9s (75% of slot), PTC members attest to whether they saw the payload:

```
  PTC MEMBER (Slot N, attesting to payload of Slot N)
  ═══════════════════════════════════════════════════

  ┌────────────────────────────────────────────────┐
  │                                                 │
  │  Am I in the PTC for slot N?                    │
  │    Check via get_ptc(state, slot)               │
  │    PTC_SIZE = 512 validators per slot           │
  │                                                 │
  │  Have I seen the block for slot N?              │
  │    NO  → don't submit (will be ignored)         │
  │    YES → continue                               │
  │                                                 │
  │  Have I seen the payload envelope?              │
  │                                                 │
  │  ┌──────────────────┐  ┌──────────────────┐    │
  │  │ YES              │  │ NO               │    │
  │  │ payload_present  │  │ payload_present  │    │
  │  │   = true         │  │   = false        │    │
  │  └──────────────────┘  └──────────────────┘    │
  │                                                 │
  │  Build PayloadAttestationMessage:               │
  │    data.beacon_block_root = block root          │
  │    data.slot = N                                │
  │    data.payload_present = true/false            │
  │    validator_index = my index                   │
  │    signature = sign(data)                       │
  │                                                 │
  │  Broadcast on payload_attestation_message topic │
  │                                                 │
  └────────────────────────────────────────────────┘

  Next slot's proposer aggregates these into
  payload_attestations in their block body
  (up to MAX_PAYLOAD_ATTESTATIONS = 4)
```

### Spec items for this flow

| Step | What | Spec Item | Prysm Status |
|------|------|-----------|:------------:|
| Assign | PTC selection | `get_ptc` + `compute_balance_weighted_selection` | ✅ [#16293](https://github.com/OffchainLabs/prysm/pull/16293) |
| Assign | PTC assignment | `get_ptc_assignment` | ❌ |
| Create | Build message | Constructing `PayloadAttestationMessage` | ❌ |
| Create | Sign message | `get_payload_attestation_message_signature` | ❌ |
| Gossip | PTC gossip | `payload_attestation_message` topic | ✅ [#16333](https://github.com/OffchainLabs/prysm/pull/16333) |
| Pool | Collect PTC atts | Payload attestation pool | 🔄 [#16308](https://github.com/OffchainLabs/prysm/pull/16308) |
| Include | Aggregate in block | Constructing `payload_attestations` for block body | ❌ |

---

## 7. Full Slot Lifecycle (Putting It All Together)

```
 Slot N-1 (previous)                     Slot N (current)
 ════════════════════                     ═══════════════════════════════════════

                                          T=0s
                                          ┌─────────────────────────────┐
                                          │ Proposer broadcasts:        │
                                          │   1. SignedBeaconBlock       │
                                          │      (with self-build bid)  │
                                          │   2. SignedExecPayloadEnv   │
                                          │      (actual payload)       │
                                          └─────────────┬───────────────┘
                                                        │
                                                        ▼
                                          ┌─────────────────────────────┐
                                          │ All nodes:                   │
                                          │  • Validate & accept block   │
                                          │    (EMPTY until payload)     │
                                          │  • process_slot: clear avail │
                                          │  • process_operations:       │
                                          │    - process bid             │
                                          │    - process attestations    │
                                          │  • process_withdrawals:      │
 Was slot N-1 FULL or EMPTY? ────────────►│    - skip if parent empty    │
                                          │  • Validate & accept payload │
                                          │    (now FULL)                │
                                          └─────────────┬───────────────┘
                                                        │
                                          T=3s          ▼
                                          ┌─────────────────────────────┐
                                          │ Attesters for slot N:        │
                                          │  • Check: is slot N-1 FULL?  │
                                          │    YES → index=1             │
                                          │    NO  → index=0             │
                                          │  • Broadcast attestation     │
                                          └─────────────┬───────────────┘
                                                        │
                                          T=6s          ▼
                                          ┌─────────────────────────────┐
                                          │ Aggregators:                 │
                                          │  • Aggregate attestations    │
                                          │  • Broadcast aggregates      │
                                          └─────────────┬───────────────┘
                                                        │
                                          T=9s          ▼
                                          ┌─────────────────────────────┐
                                          │ PTC members for slot N:      │
                                          │  (optional for devnet-0)     │
                                          │  • Did payload arrive?       │
                                          │  • Broadcast PTC attestation │
                                          └─────────────────────────────┘

                                              ──── Slot N+1 begins ────
                                          Proposer includes PTC votes
                                          from slot N in their block
```

---

## 8. The Empty Chain: What Happens When Payloads Are Missed

```
  Slot 100         Slot 101          Slot 102          Slot 103
  ═══════          ════════          ════════          ════════

  Block ✓          Block ✓           Block ✓           Block ✓
  Payload ✓        Payload ✗         Payload ✗         Payload ✓
  ─────────        ─────────         ─────────         ─────────
  FULL              EMPTY             EMPTY             FULL

  Attesters:       Attesters:        Attesters:        Attesters:
  index=?          index=1           index=0           index=0
  (prev slot)      (100 was FULL)    (101 was EMPTY)   (102 was EMPTY)

  Withdrawals:     Withdrawals:      Withdrawals:      Withdrawals:
  normal           normal            SKIP (parent      SKIP (parent
                   (parent FULL)     was EMPTY)        was EMPTY)
```

Key behavior:
- **Blocks always get proposed** — even without a payload from the previous slot
- **`is_parent_block_full`** determines withdrawal processing and attestation index
- **Payload requests** (`ExecutionPayloadEnvelopesByRoot`) help recover missed payloads
- **The chain never stalls** — it just has some empty slots mixed with full ones

### Spec items for empty chain handling

| What | Spec Item | Prysm Status |
|------|-----------|:------------:|
| Track full/empty | `is_parent_block_full` | ✅ |
| Skip withdrawals | Modified `process_withdrawals` — early return | 🔄 [#16310](https://github.com/OffchainLabs/prysm/pull/16310) |
| Recover payloads | `ExecutionPayloadEnvelopesByRoot v1` | ❌ |
| Payload availability | `execution_payload_availability` in state | ✅ [#15730](https://github.com/OffchainLabs/prysm/pull/15730) |

---

## 9. Fork Transition: Getting Into Gloas

Before any of the above can happen, we need to transition from Fulu to Gloas:

```
  ┌──────────────────────────────────────────────────────┐
  │              FULU STATE                               │
  │                                                       │
  │  At slot = first_slot_of(GLOAS_FORK_EPOCH):           │
  │                                                       │
  │  upgrade_to_gloas(state)                              │
  │    ├─► Copy all Fulu state                            │
  │    ├─► Initialize new Gloas fields:                   │
  │    │     latest_block_hash = last EL block hash       │
  │    │     latest_execution_payload_bid = empty bid     │
  │    │     execution_payload_availability = false        │
  │    │     builders = []  (empty registry)               │
  │    │     builder_pending_payments = []                 │
  │    │     builder_pending_withdrawals = []              │
  │    │     next_withdrawal_builder_index = 0             │
  │    │     payload_expected_withdrawals = empty          │
  │    └─► Return Gloas state                             │
  │                                                       │
  │  P2P fork digest changes:                             │
  │    GLOAS_FORK_VERSION (0x07000000)                    │
  │    compute_fork_version() adds Gloas case             │
  │                                                       │
  └──────────────────────────────────────────────────────┘
```

### Spec items for fork transition

| What | Spec Item | Prysm Status |
|------|-----------|:------------:|
| Fork version | `GLOAS_FORK_VERSION` (0x07000000) | ❌ |
| Fork epoch | `GLOAS_FORK_EPOCH` | ✅ |
| State upgrade | `upgrade_to_gloas` | ❌ |
| Version helper | Modified `compute_fork_version` | ❌ |
| State init | `InitializeFromProtoUnsafeGloas()` | ✅ [#15611](https://github.com/OffchainLabs/prysm/pull/15611) |
| Can upgrade | `CanUpgradeToGloas()` | ❌ |

---

## 10. Summary: All Flows → All Spec Items

### What's done (can rely on)

| Category | Items |
|----------|-------|
| **Protobuf types** | All containers defined — [#15601](https://github.com/OffchainLabs/prysm/pull/15601) |
| **Block/State types** | `BeaconBlockBody`, `BeaconState` — [#15618](https://github.com/OffchainLabs/prysm/pull/15618) [#15611](https://github.com/OffchainLabs/prysm/pull/15611) [#16164](https://github.com/OffchainLabs/prysm/pull/16164) |
| **Bid processing** | `process_execution_payload_bid` — [#15638](https://github.com/OffchainLabs/prysm/pull/15638) |
| **Payload processing** | `process_execution_payload` — [#15656](https://github.com/OffchainLabs/prysm/pull/15656) [#16356](https://github.com/OffchainLabs/prysm/pull/16356) |
| **Slot processing** | Modified `process_slot` — [#15730](https://github.com/OffchainLabs/prysm/pull/15730) |
| **Attestation processing** | Modified `process_attestation` + flag indices — [#15736](https://github.com/OffchainLabs/prysm/pull/15736) |
| **PTC selection** | `get_ptc` + balance-weighted selection — [#16293](https://github.com/OffchainLabs/prysm/pull/16293) |
| **Payload attestation** | `process_payload_attestation` — [#15650](https://github.com/OffchainLabs/prysm/pull/15650) |
| **Builder payments** | `process_builder_pending_payments` — [#15655](https://github.com/OffchainLabs/prysm/pull/15655) |
| **Proposer slashing** | Modified `process_proposer_slashing` — [#16212](https://github.com/OffchainLabs/prysm/pull/16212) |
| **Timing params** | All BPS constants — [#16291](https://github.com/OffchainLabs/prysm/pull/16291) |
| **Payload gossip** | Validation + subscriber — [#16349](https://github.com/OffchainLabs/prysm/pull/16349) [#16339](https://github.com/OffchainLabs/prysm/pull/16339) |
| **PTC gossip** | Validation + subscriber — [#16333](https://github.com/OffchainLabs/prysm/pull/16333) |
| **Predicates** | `is_parent_block_full`, `is_attestation_same_slot`, `is_valid_indexed_payload_attestation` |
| **DB layer** | Block + payload persistence — [#16301](https://github.com/OffchainLabs/prysm/pull/16301) |
| **Beacon API** | `GET /eth/v2/beacon/blocks/{block_id}` — [#16278](https://github.com/OffchainLabs/prysm/pull/16278) |

### What's in review (close to landing)

| Category | Items |
|----------|-------|
| **Withdrawals** | Full `process_withdrawals` rewrite — [#16310](https://github.com/OffchainLabs/prysm/pull/16310) |
| **Attestation gossip** | Index < 2 validation — [#16359](https://github.com/OffchainLabs/prysm/pull/16359) |
| **Proposer flow** | Block get/publish (Gloas case in existing RPCs) + new envelope RPCs + validator client flow + self-build bid — [#16336](https://github.com/OffchainLabs/prysm/pull/16336) |
| **Fork choice** | Full/empty model, weights — [#16338](https://github.com/OffchainLabs/prysm/pull/16338) [#16351](https://github.com/OffchainLabs/prysm/pull/16351) [#16357](https://github.com/OffchainLabs/prysm/pull/16357) |
| **Beacon APIs** | PTC duties, proposer duties, debug state, events, PTC pool — [#16326](https://github.com/OffchainLabs/prysm/pull/16326) [#16303](https://github.com/OffchainLabs/prysm/pull/16303) [#16296](https://github.com/OffchainLabs/prysm/pull/16296) [#16323](https://github.com/OffchainLabs/prysm/pull/16323) [#16306](https://github.com/OffchainLabs/prysm/pull/16306) |

### What's not started (blocks end-to-end)

| Priority | Item | Why it blocks |
|:--------:|------|---------------|
| 🔴 | `GLOAS_FORK_VERSION` in config | Everything P2P: fork digest, req/resp, version helpers |
| 🔴 | `upgrade_to_gloas` + `CanUpgradeToGloas` | No fork transition = nothing runs |
| 🔴 | Modified `process_operations` wiring | Bid + PTC processing exist but aren't called |
| 🔴 | `ReceiveExecutionPayloadEnvelope` chain handler | Gossip validation done but handler is a stub |
| 🔴 | `ExecutionPayloadEnvelopesByRoot v1` req/resp | Clients stall without payload recovery |
| 🟡 | `beacon_block` gossip validation for Gloas | Block gossip needs to handle new structure |
| 🟡 | Validator: attestation `index` construction | Validators need to set index based on payload status |
| 🟡 | `MAX_PAYLOAD_ATTESTATIONS` preset | Missing from config |
| 🟡 | `process_epoch` wiring for pending payments | Impl exists, not wired |
