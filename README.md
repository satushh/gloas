# GLOAS: ePBS and Related Consensus-Layer Changes

A Complete Visual Guide

Scope note: the latest Gloas spec is more than EIP-7732/ePBS. It includes
EIP-7732, EIP-7843, EIP-8045, and EIP-8061; the Gloas `ExecutionPayload` also
carries the EIP-7928 `block_access_list`.

---

## Table of Contents

1. [The Big Picture: What Problem Does GLOAS Solve?](#1-the-big-picture-what-problem-does-gloas-solve)
2. [Before vs After: Structural Changes](#2-before-vs-after-structural-changes)
3. [Slot Timeline: What Happens When?](#3-slot-timeline-what-happens-when)
4. [New Data Structures](#4-new-data-structures)
5. [Fork Choice Changes: Empty vs Full Blocks](#5-fork-choice-changes-empty-vs-full-blocks)
6. [Networking & P2P Changes](#6-networking--p2p-changes)
7. [Actor Perspectives](#7-actor-perspectives)
8. [Putting It All Together](#8-putting-it-all-together)
9. [Spec Reference Index](#9-spec-reference-index)
10. [FAQ](#10-faq)

---

## 1. THE BIG PICTURE: What Problem Does GLOAS Solve?

```text
  The Current Problem (Pre-GLOAS)

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                        CURRENT SYSTEM (Pre-GLOAS)                           │
  │                                                                             │
  │   ┌──────────────┐         ┌─────────────────┐         ┌──────────────┐     │
  │   │   BUILDER    │ ──────► │     RELAY       │ ──────► │  PROPOSER    │     │
  │   │  (MEV-Boost) │  block  │  (Trusted 3rd   │  block  │ (Validator)  │     │
  │   └──────────────┘         │     Party)      │         └──────────────┘     │
  │                            └─────────────────┘                              │
  │                                    │                                        │
  │                                    │ PROBLEMS:                              │
  │                                    │ • Relays are centralized               │
  │                                    │ • Relays can censor transactions       │
  │                                    │ • Builders must trust relays           │
  │                                    │ • No protocol-level guarantees         │
  │                                    │ • Relay can steal MEV                  │
  │                                    ▼                                        │
  │                          ┌─────────────────┐                                │
  │                          │  TRUST ISSUES   │                                │
  │                          │  & CENTRALIZED  │                                │
  │                          │  FAILURE POINTS │                                │
  │                          └─────────────────┘                                │
  └─────────────────────────────────────────────────────────────────────────────┘

  Why MEV-Boost exists today:
  - Proposers want MEV (Maximal Extractable Value) profits
  - Builders specialize in constructing profitable blocks
  - But: proposers can't trust builders to reveal blocks, builders can't trust proposers to not steal

  MEV-Boost "solution": Trusted relays act as escrow, but this introduces centralization.

  ---
  The GLOAS Solution: Protocol-Level PBS

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                 GLOAS SYSTEM (EIP-7732 + companion EIPs)                    │
  │                                                                             │
  │   ┌──────────────┐                                   ┌──────────────┐       │
  │   │   BUILDER    │ ─────── BID ────────────────────► │  PROPOSER    │       │
  │   │ (Staked actor│    SignedExecutionPayloadBid      │ (Validator)  │       │
  │   │  with 0x03)  │                                   └──────┬───────┘       │
  │   └──────┬───────┘                                          │               │
  │          │                                                  │               │
  │          │ PAYLOAD                              BEACON BLOCK│               │
  │          │ SignedExecutionPayloadEnvelope       (with bid)  │               │
  │          │                                                  │               │
  │          ▼                                                  ▼               │
  │   ┌──────────────────────────────────────────────────────────────────┐      │
  │   │                     ETHEREUM PROTOCOL                            │      │
  │   │                                                                  │      │
  │   │  • Bids are commitments enforced by protocol                     │      │
  │   │  • Builder pays when FULL parent built OR quorum reached          │      │
  │   │  • PTC votes on payload timeliness and blob data availability    │      │
  │   │  • No trusted third party needed!                                │      │
  │   └──────────────────────────────────────────────────────────────────┘      │
  │                                    │                                        │
  │   SPEC ENTITIES:                   ▼                                        │
  │   • BUILDER_WITHDRAWAL_PREFIX = 0x03           ┌─────────────────┐          │
  │   • is_builder_withdrawal_credential()        │   TRUSTLESS!    │          │
  │   • PTC_SIZE = 512 positions                   │  DECENTRALIZED  │          │
  │   • get_ptc(state, slot)                       │  CENSORSHIP     │          │
  │                                                │  RESISTANT      │          │
  │                                                └─────────────────┘          │
  └─────────────────────────────────────────────────────────────────────────────┘

  Key insight: The protocol itself becomes the escrow through cryptographic commitments and economic penalties.

  ---
```

## 2. BEFORE vs AFTER: Structural Changes

```text
  2.1 BeaconBlockBody Changes

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                    BEFORE (Fulu)                │    AFTER (GLOAS)          │
  ├─────────────────────────────────────────────────┼───────────────────────────┤
  │                                                 │                           │
  │  BeaconBlockBody {                              │  BeaconBlockBody {        │
  │    randao_reveal                                │    randao_reveal          │
  │    eth1_data                                    │    eth1_data              │
  │    graffiti                                     │    graffiti               │
  │    proposer_slashings                           │    proposer_slashings     │
  │    attester_slashings                           │    attester_slashings     │
  │    attestations                                 │    attestations           │
  │    deposits                                     │    deposits               │
  │    voluntary_exits                              │    voluntary_exits        │
  │    sync_aggregate                               │    sync_aggregate         │
  │    bls_to_execution_changes                     │    bls_to_execution_changes│
  │                                                 │                           │
  │    ╔══════════════════════════════╗             │    ╔═════════════════════╗│
  │    ║  execution_payload ❌ REMOVED ║            │    ║ signed_execution_   ║│
  │    ║  blob_kzg_commitments ❌      ║            │    ║ payload_bid ✅ NEW  ║│
  │    ║  execution_requests ❌        ║            │    ║                     ║│
  │    ╚══════════════════════════════╝             │    ║ payload_attestations║│
  │                                                 │    ║ ✅ NEW              ║│
  │                                                 │    ║ parent_execution_  ║│
  │                                                 │    ║ requests ✅ NEW    ║│
  │  }                                              │    ╚═════════════════════╝│
  │                                                 │  }                        │
  └─────────────────────────────────────────────────┴───────────────────────────┘

  WHY THIS CHANGE?
  ════════════════
  BEFORE: Proposer includes the actual execution_payload in their block
          → Proposer must build the block OR trust a relay

  AFTER:  Proposer includes only a BID (commitment) from a builder
          → Execution payload comes SEPARATELY from the builder
          → Separation of concerns: Proposer selects bid, Builder delivers payload

  SPEC: beacon-chain.md → BeaconBlockBody container
        New fields: signed_execution_payload_bid: SignedExecutionPayloadBid
                    payload_attestations: List[PayloadAttestation, MAX_PAYLOAD_ATTESTATIONS]
                    parent_execution_requests: ExecutionRequests

  2.2 Block Structure: One Block Becomes Two Objects

                              BEFORE (Fulu)
      ┌─────────────────────────────────────────────────────────┐
      │                    SignedBeaconBlock                    │
      │  ┌───────────────────────────────────────────────────┐  │
      │  │                 BeaconBlock                       │  │
      │  │  ┌─────────────────────────────────────────────┐  │  │
      │  │  │              BeaconBlockBody                │  │  │
      │  │  │  ┌───────────────────────────────────────┐  │  │  │
      │  │  │  │         ExecutionPayload              │  │  │  │
      │  │  │  │  • transactions                       │  │  │  │
      │  │  │  │  • withdrawals                        │  │  │  │
      │  │  │  └───────────────────────────────────────┘  │  │  │
      │  │  │  • blob_kzg_commitments                     │  │  │
      │  │  │  • execution_requests                       │  │  │
      │  │  └─────────────────────────────────────────────┘  │  │
      │  └───────────────────────────────────────────────────┘  │
      └─────────────────────────────────────────────────────────┘
                           ALL IN ONE BLOCK
                                │
                                ▼

                              AFTER (GLOAS)
      ┌─────────────────────────────────┐     ┌─────────────────────────────────┐
      │      SignedBeaconBlock          │     │  SignedExecutionPayloadEnvelope │
      │  ┌───────────────────────────┐  │     │  ┌───────────────────────────┐  │
      │  │       BeaconBlock         │  │     │  │ ExecutionPayloadEnvelope  │  │
      │  │  ┌─────────────────────┐  │  │     │  │  ┌─────────────────────┐  │  │
      │  │  │   BeaconBlockBody   │  │  │     │  │  │  ExecutionPayload   │  │  │
      │  │  │                     │  │  │     │  │  │  • transactions     │  │  │
      │  │  │ ╔═════════════════╗ │  │  │     │  │  │  • withdrawals      │  │  │
      │  │  │ ║ SignedExecution ║ │  │  │ ──► │  │  └─────────────────────┘  │  │
      │  │  │ ║ PayloadBid      ║ │  │  │ref  │  │  execution_requests       │  │
      │  │  │ ║ (commitment)    ║ │  │  │     │  │                            │  │
      │  │  │ ╚═════════════════╝ │  │  │     │  │  beacon_block_root ◄──────┼──┤
      │  │  │ payload_attestations│  │  │     │  │  parent_beacon_block_root │  │
      │  │  │ parent_execution_req│  │  │     │  │                            │  │
      │  │  └─────────────────────┘  │  │     │  └───────────────────────────┘  │
      │  └───────────────────────────┘  │     └─────────────────────────────────┘
      └─────────────────────────────────┘
             PROPOSER creates                      BUILDER creates
             (at slot start)                       (after block seen)

  Reasoning: This separation allows the builder to see the beacon block before revealing their payload, creating a commit-reveal scheme that's enforced by the protocol.
  Execution requests from the revealed payload are processed by the child block through parent_execution_requests.

  ---
  2.3 BeaconState Changes

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                         NEW FIELDS IN BeaconState                           │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  REMOVED:                                                                   │
  │  ────────                                                                   │
  │  ❌ latest_execution_payload_header                                         │
  │     └─► WHY? No longer storing full header, only the bid commitment         │
  │                                                                             │
  │  ADDED:                                                                     │
  │  ──────                                                                     │
  │  ✅ builders: List[Builder, BUILDER_REGISTRY_LIMIT]                         │
  │     └─► Separate registry for builders (not validators!)                    │
  │                                                                             │
  │  ✅ next_withdrawal_builder_index: BuilderIndex                             │
  │     └─► Tracks sweep position for builder withdrawals                      │
  │                                                                             │
  │  ✅ latest_execution_payload_bid: ExecutionPayloadBid                       │
  │     └─► Stores the committed bid (block_hash, value, builder_index, etc)    │
  │                                                                             │
  │  ✅ execution_payload_availability: Bitvector[SLOTS_PER_HISTORICAL_ROOT]    │
  │     └─► Tracks which slots had payloads delivered (for attestation rewards) │
  │                                                                             │
  │  ✅ builder_pending_payments: Vector[BuilderPendingPayment, 2*SLOTS_PER_EPOCH]│
  │     └─► Payments waiting for quorum confirmation (2 epoch window)           │
  │                                                                             │
  │  ✅ builder_pending_withdrawals: List[BuilderPendingWithdrawal, 1M limit]   │
  │     └─► Confirmed payments queued for withdrawal to proposer                │
  │                                                                             │
  │  ✅ latest_block_hash: Hash32                                               │
  │     └─► Tracks the most recent execution block hash for continuity          │
  │                                                                             │
  │  ✅ payload_expected_withdrawals: List[Withdrawal, MAX_WITHDRAWALS_PER_PAYLOAD]│
  │     └─► Pre-computed withdrawals the payload must honor                     │
  │                                                                             │
  │  ✅ ptc_window: Vector[Vector[ValidatorIndex, PTC_SIZE], ...]               │
  │     └─► Cached PTC assignments for previous/current/lookahead epochs        │
  │                                                                             │
  │  SPEC: beacon-chain.md → BeaconState container                              │
  │  LIMIT: BUILDER_PENDING_WITHDRAWALS_LIMIT = 1,048,576                       │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  ---
```

## 3. SLOT TIMELINE: What Happens When?

```text
  3.1 The New Slot Structure

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                          SLOT N TIMELINE (12 seconds)                       │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  0s            3s              6s              9s              12s          │
  │  │             │               │               │               │            │
  │  ▼             ▼               ▼               ▼               ▼            │
  │  ┌─────────────┬───────────────┬───────────────┬───────────────┐            │
  │  │   0-25%     │    25-50%     │    50-75%     │   75-100%     │            │
  │  │             │               │               │               │            │
  │  │  PROPOSER   │  ATTESTERS    │  AGGREGATORS  │     PTC       │            │
  │  │  broadcasts │  vote on      │  aggregate    │  vote on      │            │
  │  │  block      │  block        │  attestations │  PAYLOAD      │            │
  │  │             │               │               │               │            │
  │  └─────────────┴───────────────┴───────────────┴───────────────┘            │
  │        │              │               │               │                     │
  │        │              │               │               │                     │
  │  ┌─────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐                 │
  │  │  BUILDER  │  │           │  │           │  │           │                 │
  │  │  sees     │  │           │  │           │  │           │                 │
  │  │  block,   │  │           │  │           │  │           │                 │
  │  │  reveals  │  │           │  │           │  │           │                 │
  │  │  payload  │  │           │  │           │  │           │                 │
  │  └───────────┘  └───────────┘  └───────────┘  └───────────┘                 │
  │                                                                             │
  │  TIMING CONSTANTS (SPEC: validator.md):                                     │
  │  ════════════════════════════════════                                       │
  │  ATTESTATION_DUE_BPS_GLOAS   = 2500 (25% = 3s)  ← Earlier than before!      │
  │  AGGREGATE_DUE_BPS_GLOAS     = 5000 (50% = 6s)                              │
  │  SYNC_MESSAGE_DUE_BPS_GLOAS  = 2500 (25% = 3s)                              │
  │  CONTRIBUTION_DUE_BPS_GLOAS  = 5000 (50% = 6s)                              │
  │  PAYLOAD_DUE_BPS             = 7500 (75% = 9s)  ← NEW! Payload due          │
  │  PAYLOAD_ATTESTATION_DUE_BPS = 7500 (75% = 9s)  ← NEW! For PTC              │
  │                                                                             │
  │  FUNCTIONS (fork-choice.md): get_attestation_due_ms(epoch)                  │
  │                              get_payload_attestation_due_ms(epoch)          │
  │                              get_payload_due_ms(epoch)                      │
  │                                                                             │
  │  WHY EARLIER ATTESTATION DEADLINE?                                          │
  │  ─────────────────────────────────                                          │
  │  Attesters vote at 25% (was 33%) to give the builder more time              │
  │  to construct and broadcast the payload before PTC deadline at 75%          │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  3.2 Detailed Actor Timeline

  TIME    PROPOSER              BUILDER               ATTESTERS           PTC
  ═════   ════════              ═══════               ═════════           ═══

  SLOT N-1 (previous slot)
  ─────────────────────────────────────────────────────────────────────────────
          │                     │                     │                   │
          │                     │ Constructs payload  │                   │
          │                     │ for slot N          │                   │
          │                     │                     │                   │
          │                     ▼                     │                   │
          │              ┌─────────────┐              │                   │
          │              │ Creates BID │              │                   │
          │              │ with:       │              │                   │
          │              │ • block_hash│              │                   │
          │              │ • value     │              │                   │
          │              │ • requests_ │              │                   │
          │              │   root      │              │                   │
          │              │ • signature │              │                   │
          │              └──────┬──────┘              │                   │
          │                     │                     │                   │
          │ ◄───────────────────┘                     │                   │
          │   Receives bid                            │                   │
          │   (via P2P or direct)                     │                   │

  SLOT N: 0% (0 seconds)
  ─────────────────────────────────────────────────────────────────────────────
          │                     │                     │                   │
          ▼                     │                     │                   │
   ┌──────────────┐             │                     │                   │
   │ Creates      │             │                     │                   │
   │ BeaconBlock  │             │                     │                   │
   │ with:        │             │                     │                   │
   │ • bid inside │             │                     │                   │
   │ • payload_   │             │                     │                   │
   │   attestations│            │                     │                   │
   │   (from N-1) │             │                     │                   │
   │ • parent_exec│             │                     │                   │
   │   requests   │             │                     │                   │
   └──────┬───────┘             │                     │                   │
          │                     │                     │                   │
          │ BROADCASTS ─────────┼─────────────────────┼───────────────────►
          │ SignedBeaconBlock   │                     │                   │
          │                     │                     │                   │
          │                     ▼                     │                   │
          │              ┌─────────────┐              │                   │
          │              │ Sees block  │              │                   │
          │              │ Verifies    │              │                   │
          │              │ their bid   │              │                   │
          │              │ was included│              │                   │
          │              └──────┬──────┘              │                   │
          │                     │                     │                   │
          │                     ▼                     │                   │
          │              ┌─────────────┐              │                   │
          │              │ BROADCASTS  │              │                   │
          │              │ Execution   │──────────────┼───────────────────►
          │              │ Payload     │              │                   │
          │              │ Envelope    │              │                   │
          │              └─────────────┘              │                   │

  SLOT N: 25% (3 seconds) - ATTESTATION DEADLINE
  ─────────────────────────────────────────────────────────────────────────────
          │                     │                     │                   │
          │                     │                     ▼                   │
          │                     │              ┌─────────────┐            │
          │                     │              │ ATTEST to   │            │
          │                     │              │ block with  │            │
          │                     │              │ index field:│            │
          │                     │              │ • same-slot:│            │
          │                     │              │   index = 0 │            │
          │                     │              │ • prior-slot│            │
          │                     │              │   0=empty   │            │
          │                     │              │   1=full    │            │
          │                     │              └──────┬──────┘            │
          │                     │                     │                   │
          │                     │                     ▼                   │
          │                     │              BROADCAST attestation      │

  SLOT N: 75% (9 seconds) - PTC DEADLINE
  ─────────────────────────────────────────────────────────────────────────────
          │                     │                     │                   │
          │                     │                     │                   ▼
          │                     │                     │            ┌─────────────┐
          │                     │                     │            │ PTC votes   │
          │                     │                     │            │ on payload: │
          │                     │                     │            │ • present?  │
          │                     │                     │            │ • available?│
          │                     │                     │            └──────┬──────┘
          │                     │                     │                   │
          │                     │                     │                   ▼
          │                     │                     │            BROADCAST
          │                     │                     │            PayloadAttestation
          │                     │                     │            Message

  SLOT N+1: 0% - NEXT SLOT
  ─────────────────────────────────────────────────────────────────────────────
          │                     │                     │                   │
          ▼                     │                     │                   │
   Next proposer                │                     │                   │
   includes PTC                 │                     │                   │
   attestations from            │                     │                   │
   slot N in block              │                     │                   │

  ---
  3.3 The Payment Flow

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                    BUILDER PAYMENT LIFECYCLE                                │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  SLOT N: Bid Committed                                                      │
  │  ════════════════════                                                       │
  │                                                                             │
  │  ┌─────────────┐      ┌─────────────┐      ┌─────────────────────────┐      │
  │  │ Builder     │      │ Proposer    │      │ BeaconState             │      │
  │  │ bid.value   │ ───► │ includes    │ ───► │ builder_pending_payments│      │
  │  │ = 1 ETH     │      │ bid in      │      │ [slot N] = {            │      │
  │  │             │      │ block       │      │   weight: 0,            │      │
  │  └─────────────┘      └─────────────┘      │   withdrawal: {         │      │
  │                                            │     amount: 1 ETH,      │      │
  │                                            │     builder_index: X    │      │
  │                                            │   }                     │      │
  │                                            │ }                       │      │
  │                                            └─────────────────────────┘      │
  │                                                                             │
  │  SLOT N: Attestations Accumulate Weight (for quorum backup)                 │
  │  ════════════════════════════════════════════════════════                  │
  │                                                                             │
  │  Same-slot attesters voting for the block add their effective_balance       │
  │  to the payment's "weight" field (used if payload NOT delivered):           │
  │                                                                             │
  │  ┌─────────────────┐                                                        │
  │  │ Attester A      │                                                        │
  │  │ eff_bal: 32 ETH │ ──┐                                                    │
  │  └─────────────────┘   │                                                    │
  │  ┌─────────────────┐   │    ┌─────────────────────────────┐                 │
  │  │ Attester B      │   ├──► │ payment.weight += eff_bal   │                 │
  │  │ eff_bal: 32 ETH │ ──┤    │ (accumulates with each      │                 │
  │  └─────────────────┘   │    │  same-slot attestation)     │                 │
  │  ┌─────────────────┐   │    └─────────────────────────────┘                 │
  │  │ Attester C      │ ──┘                                                    │
  │  │ eff_bal: 64 ETH │                                                        │
  │  └─────────────────┘                                                        │
  │                                                                             │
  │  PARENT PAYLOAD PROCESSED: Immediate Queue                                  │
  │  ════════════════════════════════                                           │
  │                                                                             │
  │  When a child block runs process_parent_execution_payload on FULL parent:   │
  │  → Payment is IMMEDIATELY moved to builder_pending_withdrawals              │
  │  → No quorum check needed - the child built on the revealed parent payload  │
  │                                                                             │
  │  EPOCH BOUNDARY: Quorum Check (backup for withheld payloads)                │
  │  ═════════════════════════════════════════════════════════                  │
  │                                                                             │
  │  At epoch processing (process_builder_pending_payments), for payments       │
  │  where the payload was NOT delivered:                                       │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐    │
  │  │                                                                     │    │
  │  │  quorum = (total_active_balance / SLOTS_PER_EPOCH) * 60%            │    │
  │  │                                                                     │    │
  │  │  if payment.weight >= quorum:                                       │    │
  │  │      → Move to builder_pending_withdrawals (CONFIRMED!)             │    │
  │  │      → Builder pays even though they withheld payload!              │    │
  │  │  else:                                                              │    │
  │  │      → Payment DISCARDED (builder keeps their stake!)               │    │
  │  │                                                                     │    │
  │  └─────────────────────────────────────────────────────────────────────┘    │
  │                                                                             │
  │  WHY 60% QUORUM?                                                            │
  │  ═══════════════                                                            │
  │  BUILDER_PAYMENT_THRESHOLD_NUMERATOR   = 6                                  │
  │  BUILDER_PAYMENT_THRESHOLD_DENOMINATOR = 10                                 │
  │                                                                             │
  │  This ensures payments only go through when there's strong consensus        │
  │  that the block was actually received and valid.                            │
  │                                                                             │
  │  WITHDRAWAL SWEEP: Actual ETH Transfer                                      │
  │  ═══════════════════════════════════                                       │
  │                                                                             │
  │  During withdrawal processing, builder_pending_withdrawals are converted   │
  │  to actual Withdrawal objects that send ETH to the proposer's fee_recipient │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  ---
```

## 4. NEW DATA STRUCTURES

```text
  4.1 The Execution Payload Bid

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                         ExecutionPayloadBid                                 │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │  ExecutionPayloadBid {                                              │   │
  │  │                                                                     │   │
  │  │    ┌─────────────────────┬──────────────────────────────────────┐   │   │
  │  │    │ parent_block_hash   │ Hash32 - EL parent (for continuity)  │   │   │
  │  │    ├─────────────────────┼──────────────────────────────────────┤   │   │
  │  │    │ parent_block_root   │ Root - CL parent beacon block        │   │   │
  │  │    ├─────────────────────┼──────────────────────────────────────┤   │   │
  │  │    │ block_hash          │ Hash32 - COMMITTED payload hash      │◄─┼───┤
  │  │    ├─────────────────────┼──────────────────────────────────────┤  │   │
  │  │    │ prev_randao         │ Bytes32 - For EL randomness          │  │   │ This
  │  │    ├─────────────────────┼──────────────────────────────────────┤  │   │ is the
  │  │    │ fee_recipient       │ Address - Where payment goes         │  │   │ COMMITMENT
  │  │    ├─────────────────────┼──────────────────────────────────────┤  │   │
  │  │    │ gas_limit           │ uint64 - Block gas limit             │  │   │
  │  │    ├─────────────────────┼──────────────────────────────────────┤  │   │
  │  │    │ builder_index       │ BuilderIndex - Who made bid          │  │   │
  │  │    ├─────────────────────┼──────────────────────────────────────┤  │   │
  │  │    │ slot                │ Slot - Which slot this is for        │  │   │
  │  │    ├─────────────────────┼──────────────────────────────────────┤  │   │
  │  │    │ value               │ Gwei - PAYMENT to proposer           │◄─┼───┤
  │  │    ├─────────────────────┼──────────────────────────────────────┤  │   │
  │  │    │ execution_payment   │ Gwei - For trusted out-of-protocol   │  │   │
  │  │    │                     │ auctions (must be 0 for gossip)      │  │   │
  │  │    ├─────────────────────┼──────────────────────────────────────┤  │   │
  │  │    │ blob_kzg_commitments│ List[KZGCommitment] - Blob KZG       │  │   │
  │  │    │                     │ commitments (moved here from envelope)│  │   │
  │  │    ├─────────────────────┼──────────────────────────────────────┤  │   │
  │  │    │ execution_requests_ │ Root - commitment to execution        │  │   │
  │  │    │ root                │ requests in the envelope              │  │   │
  │  │    └─────────────────────┴──────────────────────────────────────┘  │   │
  │  │  }                                                                  │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  KEY INSIGHT: The bid commits to:                                           │
  │  ═══════════════════════════════                                            │
  │  1. A specific block_hash (builder can't change payload after bid)          │
  │  2. A specific value (payment amount is locked in)                          │
  │  3. A specific parent (prevents bid reuse on different forks)               │
  │  4. Blob KZG commitments (ensures DA is also committed)                     │
  │  5. Execution requests root (processed later by the child block)            │
  │                                                                             │
  │  Pending payment is queued with the bid; FULL parent or quorum settles it   │
  │                                                                             │
  │  SPEC: beacon-chain.md → ExecutionPayloadBid container                      │
  │        SignedExecutionPayloadBid wraps with BLS signature                   │
  │  FUNCTIONS: verify_execution_payload_bid_signature()                        │
  │             process_execution_payload_bid()                                 │
  │  GOSSIP: execution_payload_bid topic (p2p-interface.md)                     │
  │  BEACON API:                                                                │
  │    GET  /eth/v1/validator/execution_payload_bid/{slot}/{builder_index}      │
  │         → Builder retrieves their execution payload bid to sign             │
  │    POST /eth/v1/beacon/execution_payload_bid                                │
  │         → Publishes a signed execution payload bid to the network           │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  4.2 The Execution Payload Envelope

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                      ExecutionPayloadEnvelope                               │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  This is what the BUILDER broadcasts after seeing the beacon block:         │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │  ExecutionPayloadEnvelope {                                         │   │
  │  │                                                                     │   │
  │  │    ┌─────────────────────────────────────────────────────────────┐ │   │
  │  │    │ payload: ExecutionPayload                                   │ │   │
  │  │    │   • parent_hash                                             │ │   │
  │  │    │   • fee_recipient                                           │ │   │
  │  │    │   • state_root                                              │ │   │
  │  │    │   • receipts_root                                           │ │   │
  │  │    │   • logs_bloom                                              │ │   │
  │  │    │   • prev_randao         ◄── Must match bid!                 │ │   │
  │  │    │   • block_number                                            │ │   │
  │  │    │   • gas_limit           ◄── Must match bid!                 │ │   │
  │  │    │   • gas_used                                                │ │   │
  │  │    │   • timestamp                                               │ │   │
  │  │    │   • extra_data                                              │ │   │
  │  │    │   • base_fee_per_gas                                        │ │   │
  │  │    │   • block_hash          ◄── Must match bid.block_hash!      │ │   │
  │  │    │   • transactions                                            │ │   │
  │  │    │   • withdrawals         ◄── Must match state expected!      │ │   │
  │  │    │   • blob_gas_used                                           │ │   │
  │  │    │   • excess_blob_gas                                         │ │   │
  │  │    │   • block_access_list   ◄── EIP-7928                       │ │   │
  │  │    │   • slot_number         ◄── Must match block slot!          │ │   │
  │  │    └─────────────────────────────────────────────────────────────┘ │   │
  │  │                                                                     │   │
  │  │    execution_requests: ExecutionRequests                            │   │
  │  │      • deposits, withdrawals, consolidations (from EL)              │   │
  │  │                                                                     │   │
  │  │    builder_index: BuilderIndex     ◄── Must match bid!              │   │
  │  │                                                                     │   │
  │  │    beacon_block_root: Root         ◄── Links to the beacon block    │   │
  │  │                                                                     │   │
  │  │    parent_beacon_block_root: Root  ◄── Must match block.parent_root │   │
  │  │                                                                     │   │
  │  │  }                                                                  │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  VERIFICATION CHAIN:                                                        │
  │  ══════════════════                                                         │
  │                                                                             │
  │  BeaconBlock                ExecutionPayloadBid           Envelope          │
  │  ┌──────────┐               ┌──────────────┐          ┌──────────────┐      │
  │  │ Contains │──────────────►│ block_hash   │◄─────────│ payload.     │      │
  │  │ bid      │               │              │  MUST    │ block_hash   │      │
  │  └──────────┘               │ builder_idx  │◄─MATCH──►│ builder_idx  │      │
  │                             │ requests_root│◄─MATCH──►│ exec_requests│      │
  │                             │ blob_comms   │ (commitments now in bid) │      │
  │                             └──────────────┘          └──────────────┘      │
  │                                                                             │
  │  SPEC: beacon-chain.md → ExecutionPayloadEnvelope container                 │
  │        SignedExecutionPayloadEnvelope wraps with BLS signature              │
  │  FUNCTIONS: verify_execution_payload_envelope_signature()                   │
  │             verify_execution_payload_envelope()                             │
  │  GOSSIP: execution_payload topic (p2p-interface.md)                         │
  │  HANDLER: on_execution_payload_envelope() (fork-choice.md)                  │
  │  BEACON API:                                                                │
  │    GET  /eth/v1/validator/execution_payload_envelope/{slot}/{builder_index} │
  │         → Builder retrieves their execution payload envelope to sign        │
  │    POST /eth/v1/beacon/execution_payload_envelope                           │
  │         → Publishes a signed execution payload envelope to the network      │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  4.3 Payload Attestation Structures (The PTC)

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                    PAYLOAD TIMELINESS COMMITTEE (PTC)                       │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  WHAT IS THE PTC?                                                           │
  │  ════════════════                                                           │
  │  A vector of 512 PTC positions (PTC_SIZE = 2^9) selected per slot           │
  │  to attest payload timeliness and blob data availability.                   │
  │  Selection can contain duplicate validators; each position counts.          │
  │                                                                             │
  │  ┌──────────────────────────────────────────────────────────────────────┐  │
  │  │                                                                      │  │
  │  │   Slot N Committees          PTC Selection                           │  │
  │  │   ══════════════════         ═════════════                           │  │
  │  │                                                                      │  │
  │  │   ┌─────────────┐                                                    │  │
  │  │   │ Committee 0 │ ──┐                                                │  │
  │  │   │ (64 vals)   │   │                                                │  │
  │  │   ├─────────────┤   │         ┌──────────────────────────┐          │  │
  │  │   │ Committee 1 │   │         │                          │          │  │
  │  │   │ (64 vals)   │   ├────────►│  compute_balance_        │          │  │
  │  │   ├─────────────┤   │         │  weighted_selection()    │          │  │
  │  │   │ Committee 2 │   │         │                          │          │  │
  │  │   │ (64 vals)   │   │         │  Selects 512 positions   │          │  │
  │  │   ├─────────────┤   │         │  weighted by stake       │          │  │
  │  │   │    ...      │ ──┘         │  (higher stake = more    │          │  │
  │  │   │ Committee N │              │   likely to be picked)  │          │  │
  │  │   └─────────────┘              └───────────┬──────────────┘          │  │
  │  │                                            │                         │  │
  │  │                                            ▼                         │  │
  │  │                                 ┌──────────────────────┐             │  │
  │  │                                 │ PTC (512 positions) │              │  │
  │  │                                 │   for Slot N         │             │  │
  │  │                                 └──────────────────────┘             │  │
  │  │                                                                      │  │
  │  └──────────────────────────────────────────────────────────────────────┘  │
  │                                                                             │
  │  WHY BALANCE-WEIGHTED SELECTION?                                            │
  │  ═══════════════════════════════                                            │
  │  - Validators with more stake have more to lose from lying                  │
  │  - Aligns PTC voting power with economic security                           │
  │  - Prevents Sybil attacks (can't get more PTC slots by splitting stake)     │
  │                                                                             │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  PayloadAttestationData                        PayloadAttestationMessage    │
  │  ═════════════════════                         ════════════════════════     │
  │                                                                             │
  │  ┌─────────────────────────┐                  ┌─────────────────────────┐  │
  │  │ beacon_block_root: Root │                  │ validator_index         │  │
  │  │   └─► Which block?      │                  │   └─► Who is voting?    │  │
  │  │                         │                  │                         │  │
  │  │ slot: Slot              │                  │ data: PayloadAttest-    │  │
  │  │   └─► Which slot?       │                  │       ationData         │  │
  │  │                         │                  │                         │  │
  │  │ payload_present: bool   │◄────────────────►│ signature: BLSSignature │  │
  │  │   └─► Was payload       │   included in    │   └─► Signed by voter   │  │
  │  │       seen?             │                  │                         │  │
  │  │                         │                  └─────────────────────────┘  │
  │  │ blob_data_available:    │                          │                    │
  │  │   bool                  │                          │ Individual votes   │
  │  │   └─► Are blobs         │                          │ get AGGREGATED     │
  │  │       available?        │                          ▼                    │
  │  └─────────────────────────┘                  ┌─────────────────────────┐  │
  │                                               │ PayloadAttestation      │  │
  │                                               │ (aggregated)            │  │
  │                                               │                         │  │
  │                                               │ aggregation_bits:       │  │
  │                                               │   Bitvector[512]        │  │
  │                                               │   └─► Which PTC members │  │
  │                                               │       are included?     │  │
  │                                               │                         │  │
  │                                               │ data: PayloadAttest-    │  │
  │                                               │       ationData         │  │
  │                                               │                         │  │
  │                                               │ signature: BLSSignature │  │
  │                                               │   └─► Aggregated sig    │  │
  │                                               └─────────────────────────┘  │
  │                                                                             │
  │  Up to 4 aggregated PayloadAttestations can be included per block           │
  │  (MAX_PAYLOAD_ATTESTATIONS = 4)                                             │
  │                                                                             │
  │  SPEC: beacon-chain.md                                                      │
  │  CONSTANTS: PTC_SIZE = 512, MAX_PAYLOAD_ATTESTATIONS = 4                    │
  │             DOMAIN_PTC_ATTESTER = DomainType('0x0C000000')                  │
  │  CONTAINERS: PayloadAttestationData, PayloadAttestation,                    │
  │              PayloadAttestationMessage, IndexedPayloadAttestation           │
  │  FUNCTIONS: get_ptc(), get_indexed_payload_attestation()                    │
  │             compute_balance_weighted_selection()                            │
  │             process_payload_attestation()                                   │
  │  GOSSIP: payload_attestation_message topic (p2p-interface.md)               │
  │  HANDLER: on_payload_attestation_message() (fork-choice.md)                 │
  │  VALIDATOR: get_ptc_assignment(), get_payload_attestation_message_signature()│
  │  BEACON API:                                                                │
  │    POST /eth/v1/validator/duties/ptc/{epoch}                                │
  │         → Retrieves PTC duties for a given epoch and validator indices      │
  │    GET  /eth/v1/validator/payload_attestation_data/{slot}                   │
  │         → Produces the payload attestation data to be signed by PTC member  │
  │    POST /eth/v1/beacon/pool/payload_attestations                            │
  │         → Submits a signed payload attestation message to the pool          │
  │    GET  /eth/v1/beacon/pool/payload_attestations                            │
  │         → Retrieves payload attestations from the beacon node's pool        │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  4.4 Builder Pending Payment Structures

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                     BUILDER PAYMENT DATA STRUCTURES                         │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  BuilderPendingPayment                    BuilderPendingWithdrawal          │
  │  ════════════════════                     ════════════════════════          │
  │                                                                             │
  │  ┌─────────────────────────┐              ┌─────────────────────────┐      │
  │  │ weight: Gwei            │              │ fee_recipient: Address  │      │
  │  │   └─► Accumulated stake │              │   └─► Where to send ETH │      │
  │  │       from same-slot    │              │                         │      │
  │  │       attesters         │              │ amount: Gwei            │      │
  │  │                         │              │   └─► How much to pay   │      │
  │  │ withdrawal:             │              │                         │      │
  │  │   BuilderPending-       │──────────────│ builder_index:          │      │
  │  │   Withdrawal            │   contains   │   BuilderIndex          │      │
  │  │                         │              │   └─► Who pays          │      │
  │  │                         │              │                         │      │
  │  └─────────────────────────┘              └─────────────────────────┘      │
  │                                                                             │
  │  LIFECYCLE:                                                                 │
  │  ══════════                                                                 │
  │                                                                             │
  │  ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐   │
  │  │ Bid included in  │     │ Payload revealed │     │ Withdrawal       │   │
  │  │ block            │ ──► │ (child processes │ ──► │ sweep processes  │   │
  │  │                  │     │  FULL parent) OR │     │                  │   │
  │  │ builder_pending_ │     │ Quorum reached   │     │ Converted to     │   │
  │  │ payments[slot]   │     │ at epoch boundary│     │ actual Withdrawal│   │
  │  │ created          │     │ (if no payload)  │     │ in execution     │   │
  │  └──────────────────┘     └──────────────────┘     └──────────────────┘   │
  │                                                                             │
  │  2-EPOCH WINDOW (from beacon-chain.md):                                     │
  │  ═══════════════════════════════════════                                    │
  │                                                                             │
  │  builder_pending_payments: Vector[BuilderPendingPayment, 2 * SLOTS_PER_EPOCH]│
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │ Epoch N-1 slots          │            Epoch N slots                 │   │
  │  │ [0..31]                  │            [32..63]                      │   │
  │  │ (previous epoch)         │            (current epoch)               │   │
  │  │                          │                                          │   │
  │  │ Processed at end of      │            Being filled by               │   │
  │  │ epoch N                  │            current blocks                │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  WHY 2 EPOCHS?                                                              │
  │  Gives attesters from the previous epoch time to be included in blocks,     │
  │  accumulating weight for the payment quorum check.                          │
  │                                                                             │
  │  SPEC: beacon-chain.md                                                      │
  │  CONTAINERS: BuilderPendingPayment, BuilderPendingWithdrawal                │
  │  CONSTANTS: BUILDER_PAYMENT_THRESHOLD_NUMERATOR = 6                         │
  │             BUILDER_PAYMENT_THRESHOLD_DENOMINATOR = 10 (60% quorum)         │
  │  FUNCTIONS: process_builder_pending_payments()                              │
  │             get_builder_payment_quorum_threshold()                          │
  │             get_builder_withdrawals()                                       │
  │             get_builders_sweep_withdrawals()                                │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  ---
```

## 5. FORK CHOICE CHANGES: Empty vs Full Blocks

```text
  5.1 The New Concept: Payload Status

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                    BEFORE vs AFTER: Block Completeness                      │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  BEFORE (Fulu): A block is either valid or invalid                          │
  │  ═══════════════════════════════════════════════════                        │
  │                                                                             │
  │    Block arrives ──► Validate ──► Valid? ──► Add to fork choice             │
  │                                                                             │
  │    Simple binary: block exists or doesn't                                   │
  │                                                                             │
  │  ───────────────────────────────────────────────────────────────────────── │
  │                                                                             │
  │  AFTER (GLOAS): A block can be "empty" or "full"                            │
  │  ════════════════════════════════════════════════                           │
  │                                                                             │
  │    BeaconBlock arrives ──► Validate ──► Add with PAYLOAD_STATUS_PENDING     │
  │                                                │                            │
  │                                                ▼                            │
  │                              ┌─────────────────────────────────┐            │
  │                              │  Wait for ExecutionPayload      │            │
  │                              │  Envelope from builder...       │            │
  │                              └─────────────────────────────────┘            │
  │                                        │                                    │
  │                        ┌───────────────┴───────────────┐                   │
  │                        │                               │                    │
  │                        ▼                               ▼                    │
  │              ┌─────────────────┐             ┌─────────────────┐           │
  │              │ Payload arrives │             │ Payload doesn't │           │
  │              │ and validates   │             │ arrive/unknown │           │
  │              │                 │             │                 │           │
  │              │ PAYLOAD_STATUS_ │             │ PAYLOAD_STATUS_ │           │
  │              │ FULL            │             │ EMPTY           │           │
  │              └─────────────────┘             └─────────────────┘           │
  │                                                                             │
  │  THREE PAYLOAD STATES:                                                      │
  │  ════════════════════                                                       │
  │                                                                             │
  │  ┌───────────────────┬────────────────────────────────────────────────┐    │
  │  │ PAYLOAD_STATUS_   │ Block just arrived, waiting for payload        │    │
  │  │ PENDING (2)       │ Fork choice doesn't know yet if it's full/empty│    │
  │  ├───────────────────┼────────────────────────────────────────────────┤    │
  │  │ PAYLOAD_STATUS_   │ Payload was NOT delivered (or not valid)       │    │
  │  │ EMPTY (0)         │ Block is "empty" - no execution happened       │    │
  │  ├───────────────────┼────────────────────────────────────────────────┤    │
  │  │ PAYLOAD_STATUS_   │ Payload WAS delivered and validated            │    │
  │  │ FULL (1)          │ Block is "full" - execution completed          │    │
  │  └───────────────────┴────────────────────────────────────────────────┘    │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  5.2 Fork Choice Tree with Payload Status

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                     FORK CHOICE TREE VISUALIZATION                          │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  BEFORE: Simple tree of blocks                                              │
  │  ═════════════════════════════                                              │
  │                                                                             │
  │                    ┌─────┐                                                  │
  │                    │ B1  │                                                  │
  │                    └──┬──┘                                                  │
  │                       │                                                     │
  │              ┌────────┴────────┐                                            │
  │              │                 │                                            │
  │           ┌──┴──┐           ┌──┴──┐                                         │
  │           │ B2  │           │ B3  │                                         │
  │           └──┬──┘           └─────┘                                         │
  │              │                                                              │
  │           ┌──┴──┐                                                           │
  │           │ B4  │ ◄── HEAD                                                  │
  │           └─────┘                                                           │
  │                                                                             │
  │  ─────────────────────────────────────────────────────────────────────────  │
  │                                                                             │
  │  AFTER: Tree with BOTH block AND payload status                             │
  │  ═════════════════════════════════════════════════                          │
  │                                                                             │
  │  Each block can branch into EMPTY or FULL versions:                         │
  │                                                                             │
  │                         ┌─────────────┐                                     │
  │                         │ B1 (PENDING)│                                     │
  │                         └──────┬──────┘                                     │
  │                                │                                            │
  │                 ┌──────────────┴──────────────┐                             │
  │                 │                             │                             │
  │          ┌──────┴──────┐              ┌───────┴──────┐                      │
  │          │ B1 (EMPTY)  │              │ B1 (FULL)    │                      │
  │          │ No payload  │              │ Has payload  │                      │
  │          └──────┬──────┘              └───────┬──────┘                      │
  │                 │                             │                             │
  │          ┌──────┴──────┐              ┌───────┴──────┐                      │
  │          │ B2 builds   │              │ B2 builds    │                      │
  │          │ on EMPTY B1 │              │ on FULL B1   │                      │
  │          └─────────────┘              └──────────────┘                      │
  │                                                                             │
  │  THE ForkChoiceNode STRUCTURE:                                              │
  │  ════════════════════════════                                               │
  │                                                                             │
  │  class ForkChoiceNode(Container):                                           │
  │      root: Root              # The beacon block root                        │
  │      payload_status: uint8   # EMPTY=0, FULL=1, PENDING=2                   │
  │                                                                             │
  │  A single beacon block root can have MULTIPLE nodes in fork choice!         │
  │  One for each payload status (but PENDING is transitional).                 │
  │                                                                             │
  │  SPEC: fork-choice.md                                                       │
  │  CONSTANTS: PAYLOAD_STATUS_EMPTY=0, PAYLOAD_STATUS_FULL=1,                  │
  │             PAYLOAD_STATUS_PENDING=2, PAYLOAD_TIMELY_THRESHOLD=256          │
  │  CONTAINERS: ForkChoiceNode, LatestMessage (modified), Store (modified)     │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  5.3 How get_head() Works Now

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                     MODIFIED HEAD SELECTION ALGORITHM                       │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  BEFORE: Walk tree, pick child with highest weight                          │
  │  ════════════════════════════════════════════════                           │
  │                                                                             │
  │    head = justified_root                                                    │
  │    while head has children:                                                 │
  │        head = child with max(weight)                                        │
  │    return head                                                              │
  │                                                                             │
  │  ─────────────────────────────────────────────────────────────────────────  │
  │                                                                             │
  │  AFTER: Navigate BOTH block tree AND payload status                         │
  │  ═══════════════════════════════════════════════════                        │
  │                                                                             │
  │    head = ForkChoiceNode(root=justified_root, payload_status=PENDING)       │
  │                                                                             │
  │    while head has children:                                                 │
  │        children = get_node_children(store, blocks, head)                    │
  │        │                                                                    │
  │        │  ┌─────────────────────────────────────────────────────────┐      │
  │        │  │ get_node_children() logic:                              │      │
  │        │  │                                                         │      │
  │        │  │ if head.payload_status == PENDING:                      │      │
  │        │  │     # First decide: EMPTY or FULL?                      │      │
  │        │  │     children = [                                        │      │
  │        │  │         Node(root, EMPTY),                              │      │
  │        │  │         Node(root, FULL)  # only if payload available   │      │
  │        │  │     ]                                                   │      │
  │        │  │                                                         │      │
  │        │  │ else:  # EMPTY or FULL                                  │      │
  │        │  │     # Now find actual child blocks                      │      │
  │        │  │     children = blocks that build on this (root, status) │      │
  │        │  │                                                         │      │
  │        │  └─────────────────────────────────────────────────────────┘      │
  │        │                                                                    │
  │        ▼                                                                    │
  │        head = max(children, key=lambda c: (                                │
  │            get_weight(c),                  # Primary: attestation weight   │
  │            c.root,                         # Tiebreaker 1: lexicographic   │
  │            get_payload_status_tiebreaker(c)# Tiebreaker 2: payload status  │
  │        ))                                                                   │
  │                                                                             │
  │    return head   # Returns ForkChoiceNode, not just Root!                   │
  │                                                                             │
  │  PAYLOAD STATUS TIEBREAKER LOGIC:                                           │
  │  ════════════════════════════════                                           │
  │                                                                             │
  │  When deciding between EMPTY and FULL for previous slot's block:            │
  │                                                                             │
  │  should_extend_payload(root) returns True if:                               │
  │    - payload locally verified AND PTC supports timeliness + blob DA, OR     │
  │    - No proposer boost yet, OR                                              │
  │    - Proposer boost is for different parent, OR                             │
  │    - Proposer boost block builds on FULL version                            │
  │                                                                             │
  │  This creates a "sticky" preference for FULL when evidence supports it.     │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  5.4 Attestation Voting: Now Signals Payload Status

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │              ATTESTATIONS NOW CARRY PAYLOAD INFORMATION                     │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  BEFORE: attestation.data.index = committee index (0 to N-1)                │
  │  ═══════════════════════════════════════════════════════════                │
  │                                                                             │
  │  AFTER: attestation.data.index = payload status signal (0 or 1)             │
  │  ════════════════════════════════════════════════════════════               │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  AttestationData {                                                  │   │
  │  │      slot: Slot                                                     │   │
  │  │      index: uint64        ◄── REPURPOSED!                           │   │
  │  │      beacon_block_root: Root                                        │   │
  │  │      source: Checkpoint                                             │   │
  │  │      target: Checkpoint                                             │   │
  │  │  }                                                                  │   │
  │  │                                                                     │   │
  │  │  index = 0: "I'm attesting the payload is NOT present (EMPTY)"      │   │
  │  │  index = 1: "I'm attesting the payload IS present (FULL)"           │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  SPECIAL CASE - Same-slot attestations:                                     │
  │  ═════════════════════════════════════                                      │
  │                                                                             │
  │  If attesting to a block from the CURRENT slot (same slot as attestation):  │
  │  • MUST set index = 0                                                       │
  │  • WHY? Attesters at 25% likely haven't seen the payload yet                │
  │  • They're just voting on the beacon block, not its payload                 │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │   Slot N                                                            │   │
  │  │   │                                                                 │   │
  │  │   │ 0%: Block proposed                                              │   │
  │  │   │ ~5-10%: Payload might arrive                                    │   │
  │  │   │ 25%: Attestation deadline ◄── Most attesters haven't seen       │   │
  │  │   │                               payload yet, so index MUST be 0   │   │
  │  │   │ 75%: PTC deadline                                               │   │
  │  │   │                                                                 │   │
  │  │   Slot N+1                                                          │   │
  │  │   │ 25%: Attestation deadline ◄── NOW attesters can signal          │   │
  │  │   │                               index=0 (EMPTY) or index=1 (FULL) │   │
  │  │                                   based on what they saw            │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  LatestMessage ALSO UPDATED:                                                │
  │  ══════════════════════════                                                 │
  │                                                                             │
  │  class LatestMessage:                                                       │
  │      slot: Slot              # Changed from epoch!                          │
  │      root: Root                                                             │
  │      payload_present: bool   # NEW! Derived from attestation index          │
  │                                                                             │
  │  WHY slot instead of epoch?                                                 │
  │  More granular tracking needed because payload status can vary per slot.    │
  │                                                                             │
  │  SPEC: fork-choice.md, validator.md                                         │
  │  CONTAINERS: LatestMessage (modified to use slot instead of epoch)          │
  │  FUNCTIONS: update_latest_messages(), validate_on_attestation()             │
  │             is_attestation_same_slot()                                      │
  │             get_attestation_participation_flag_indices() (beacon-chain.md)  │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  5.5 New Fork Choice Handlers

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                      NEW FORK CHOICE EVENT HANDLERS                         │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  on_block (MODIFIED)                                                        │
  │  ═══════════════════                                                        │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  def on_block(store, signed_block):                                 │   │
  │  │      block = signed_block.message                                   │   │
  │  │                                                                     │   │
  │  │      # If block builds on FULL parent, parent payload must be known │   │
  │  │      if is_parent_node_full(store, block):                          │   │
  │  │          assert is_payload_verified(store, block.parent_root)       │   │
  │  │                                                                     │   │
  │  │      state = copy(store.block_states[block.parent_root])            │   │
  │  │      state_transition(state, signed_block, validate_result=True)    │   │
  │  │                                                                     │   │
  │  │      store.blocks[root] = block                                     │   │
  │  │      store.block_states[root] = state   # Post-block state          │   │
  │  │                                                                     │   │
  │  │      # NEW: Initialize PTC vote tracking                            │   │
  │  │      store.payload_timeliness_vote[root] = [None] * 512             │   │
  │  │      store.payload_data_availability_vote[root] = [None] * 512      │   │
  │  │                                                                     │   │
  │  │      # NEW: Process payload attestations from previous slot         │   │
  │  │      notify_ptc_messages(store, state, block.body.payload_attestations)│
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  on_execution_payload_envelope (NEW)                                        │
  │  ═══════════════════════════════════                                        │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  def on_execution_payload_envelope(store, signed_envelope):         │   │
  │  │      envelope = signed_envelope.message                             │   │
  │  │                                                                     │   │
  │  │      # Beacon block must be known                                   │   │
  │  │      assert envelope.beacon_block_root in store.block_states        │   │
  │  │                                                                     │   │
  │  │      # Check blob data availability                                 │   │
  │  │      assert is_data_available(envelope.beacon_block_root)           │   │
  │  │                                                                     │   │
  │  │      state = store.block_states[envelope.beacon_block_root]         │   │
  │  │                                                                     │   │
  │  │      # Pure verification; processing is deferred to child block     │   │
  │  │      verify_execution_payload_envelope(                             │   │
  │  │          state, signed_envelope, ENGINE)                            │   │
  │  │                                                                     │   │
  │  │      # Store verified envelope                                      │   │
  │  │      store.payloads[envelope.beacon_block_root] = envelope          │   │
  │  │      #                                                              │   │
  │  │      # This makes the FULL version of this block available         │   │
  │  │      # in fork choice!                                              │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  on_payload_attestation_message (NEW)                                       │
  │  ════════════════════════════════════                                       │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  def on_payload_attestation_message(store, msg, is_from_block):     │   │
  │  │      data = msg.data                                                │   │
  │  │                                                                     │   │
  │  │      # Get the PTC for this slot                                    │   │
  │  │      state = store.block_states[data.beacon_block_root]             │   │
  │  │      ptc = get_ptc(state, data.slot)                                │   │
  │  │                                                                     │   │
  │  │      # Ignore if not for the block's slot                           │   │
  │  │      if data.slot != state.slot: return                             │   │
  │  │                                                                     │   │
  │  │      # Validator can appear multiple times in the PTC               │   │
  │  │      ptc_indices = [i for i, v in enumerate(ptc)                    │   │
  │  │                     if v == msg.validator_index]                    │   │
  │  │      assert len(ptc_indices) > 0                                    │   │
  │  │                                                                     │   │
  │  │      # If from wire, ensure current slot and verify signature       │   │
  │  │      if not is_from_block:                                          │   │
  │  │          assert data.slot == get_current_slot(store)                │   │
  │  │          assert is_valid_indexed_payload_attestation(...)           │   │
  │  │                                                                     │   │
  │  │      # Update every PTC position held by this validator             │   │
  │  │      root = data.beacon_block_root                                  │   │
  │  │      timely_votes = store.payload_timeliness_vote[root]             │   │
  │  │      da_votes = store.payload_data_availability_vote[root]          │   │
  │  │      for ptc_index in ptc_indices:                                  │   │
  │  │          timely_votes[ptc_index] = data.payload_present             │   │
  │  │          da_votes[ptc_index] = data.blob_data_available             │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  payload_timeliness - How PTC votes are evaluated:                          │
  │  ════════════════════════════════════════════════                           │
  │                                                                             │
  │  def payload_timeliness(store, root, timely):                               │
  │      assert root in store.payload_timeliness_vote                           │
  │      # Local payload availability gates the PTC vote                        │
  │      if root not in store.payloads:                                         │
  │          return not timely                                                  │
  │      votes = store.payload_timeliness_vote[root]                            │
  │      return sum(vote == timely for vote in votes) > (                       │
  │          PAYLOAD_TIMELY_THRESHOLD)                                          │
  │                                                                             │
  │  def payload_data_availability(store, root, available):                     │
  │      assert root in store.payload_data_availability_vote                    │
  │      # Blob DA also depends on local verified payload availability          │
  │      if root not in store.payloads:                                         │
  │          return not available                                               │
  │      votes = store.payload_data_availability_vote[root]                     │
  │      return sum(vote == available for vote in votes) > (                    │
  │          DATA_AVAILABILITY_TIMELY_THRESHOLD)                                │
  │                                                                             │
  │  SPEC: fork-choice.md                                                       │
  │  HANDLERS: on_block() (modified), is_data_available() (modified),           │
  │            on_execution_payload_envelope() (new)                            │
  │            on_payload_attestation_message() (new)                           │
  │  FUNCTIONS: payload_timeliness(), payload_data_availability(),              │
  │             is_payload_verified(), notify_ptc_messages()                    │
  │             get_parent_payload_status(), is_parent_node_full()              │
  │             should_extend_payload(), get_payload_status_tiebreaker()        │
  │             get_attestation_score() (modified)                              │
  │  STORE FIELDS: payloads, payload_timeliness_vote,                           │
  │                payload_data_availability_vote (new)                         │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  5.6 Fast Confirmation Changes

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                         FAST CONFIRMATION IN GLOAS                          │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  get_node_for_root(block_root):                                             │
  │      return ForkChoiceNode(root=block_root, payload_status=PENDING)         │
  │                                                                             │
  │  Safe execution block hash:                                                 │
  │      confirmed block's parent payload hash is safe                          │
  │                                                                             │
  │  WHY? A confirmed beacon block contains a bid for its own payload, but      │
  │  that payload is only processed by a child block if the parent is FULL.     │
  │  Therefore, the safe execution hash is:                                     │
  │      confirmed.body.signed_execution_payload_bid.message.parent_block_hash  │
  │                                                                             │
  │  SPEC: fast-confirmation.md                                                 │
  └─────────────────────────────────────────────────────────────────────────────┘

  ---
```

## 6. NETWORKING & P2P CHANGES

```text
  6.1 New Gossip Topics

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                         NEW GOSSIP TOPICS IN GLOAS                          │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  BEFORE (Fulu): Main gossip topics                                          │
  │  ════════════════════════════════                                           │
  │                                                                             │
  │  • beacon_block           ─── SignedBeaconBlock (contains execution_payload)│
  │  • beacon_aggregate_and_proof                                               │
  │  • beacon_attestation_{subnet_id}                                           │
  │  • sync_committee messages                                                  │
  │  • data_column_sidecar_{subnet_id}                                          │
  │                                                                             │
  │  ───────────────────────────────────────────────────────────────────────── │
  │                                                                             │
  │  AFTER (GLOAS): Expanded and modified                                       │
  │  ════════════════════════════════════                                       │
  │                                                                             │
  │  ┌───────────────────────────────────┬─────────────────────────────────┐   │
  │  │ Topic                             │ Message Type                    │   │
  │  ├───────────────────────────────────┼─────────────────────────────────┤   │
  │  │ beacon_block                      │ SignedBeaconBlock               │   │
  │  │ (MODIFIED - no longer has payload)│ (smaller! no execution_payload) │   │
  │  ├───────────────────────────────────┼─────────────────────────────────┤   │
  │  │ execution_payload ✨ NEW          │ SignedExecutionPayloadEnvelope  │   │
  │  │                                   │ (the actual payload)            │   │
  │  ├───────────────────────────────────┼─────────────────────────────────┤   │
  │  │ execution_payload_bid ✨ NEW      │ SignedExecutionPayloadBid       │   │
  │  │                                   │ (builder bids)                  │   │
  │  ├───────────────────────────────────┼─────────────────────────────────┤   │
  │  │ proposer_preferences ✨ NEW       │ SignedProposerPreferences       │   │
  │  │                                   │ (proposer fee/gas preferences)  │   │
  │  ├───────────────────────────────────┼─────────────────────────────────┤   │
  │  │ payload_attestation_message ✨ NEW│ PayloadAttestationMessage       │   │
  │  │                                   │ (PTC votes)                     │   │
  │  ├───────────────────────────────────┼─────────────────────────────────┤   │
  │  │ data_column_sidecar_{subnet_id}   │ DataColumnSidecar               │   │
  │  │ (MODIFIED structure)              │ (simplified, no block header)   │   │
  │  └───────────────────────────────────┴─────────────────────────────────┘   │
  │                                                                             │
  │  MESSAGE FLOW VISUALIZATION:                                                │
  │  ═══════════════════════════                                                │
  │                                                                             │
  │                          BEFORE                                             │
  │                                                                             │
  │    Proposer ───beacon_block───► Network                                     │
  │              (big, contains                                                 │
  │               execution_payload)                                            │
  │                                                                             │
  │                          AFTER                                              │
  │                                                                             │
  │    Proposer ───proposer_preferences──► Network ───► Builders                │
  │                                                                             │
  │    Builder ───execution_payload_bid───► Network ───► Proposer               │
  │                (commitment)                              │                  │
  │                                                          │                  │
  │    Proposer ───beacon_block────────────► Network         │                  │
  │              (small, contains bid)                       │                  │
  │                                                          │                  │
  │    Builder ◄─────────────────────────────────────────────┘                  │
  │         │    (sees block)                                                   │
  │         │                                                                   │
  │         └──execution_payload─────────► Network ───► All nodes               │
  │            (actual payload)                                                 │
  │                                                                             │
  │    PTC ────payload_attestation_message──► Network ───► Next proposer        │
  │         (votes on timeliness + blob DA)                                     │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  6.2 Validation Rules for New Topics

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                    GOSSIP VALIDATION RULES                                  │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  execution_payload_bid                                                      │
  │  ═════════════════════                                                      │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │ REJECT if:                                                          │   │
  │  │   • builder_index is not valid/active (is_active_builder fails)     │   │
  │  │   • execution_payment is non-zero (reserved for trusted auctions)  │   │
  │  │   • fee_recipient doesn't match proposer's preferences              │   │
  │  │   • too many blob_kzg_commitments for the blob schedule             │   │
  │  │   • bid.slot is not greater than parent beacon block slot           │   │
  │  │   • signature is invalid                                           │   │
  │  │                                                                     │   │
  │  │ IGNORE if:                                                          │   │
  │  │   • slot is not current or next                                    │   │
  │  │   • SignedProposerPreferences for this slot not yet seen           │   │
  │  │   • Already seen valid bid from this builder for this slot         │   │
  │  │   • Not the highest value bid for this slot+parent                 │   │
  │  │   • Builder doesn't have sufficient balance                        │   │
  │  │   • parent_block_hash unknown in fork choice                       │   │
  │  │   • gas_limit not target-compatible with parent payload gas limit   │   │
  │  │   • parent_block_root unknown in fork choice                       │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  execution_payload                                                          │
  │  ═════════════════                                                          │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │ REJECT if:                                                          │   │
  │  │   • Referenced block doesn't pass validation                        │   │
  │  │   • payload.slot_number doesn't match block.slot                    │   │
  │  │   • builder_index doesn't match bid.builder_index                   │   │
  │  │   • payload.block_hash doesn't match bid.block_hash                 │   │
  │  │   • hash_tree_root(execution_requests) != bid.execution_requests_root│  │
  │  │   • signature is invalid                                            │   │
  │  │                                                                     │   │
  │  │ IGNORE if:                                                          │   │
  │  │   • beacon_block_root not yet seen                                  │   │
  │  │   • Already seen valid envelope for this block from this builder    │   │
  │  │   • slot < finalized slot                                           │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  payload_attestation_message                                                │
  │  ═══════════════════════════                                                │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │ REJECT if:                                                          │   │
  │  │   • Referenced block doesn't pass validation                       │   │
  │  │   • validator_index not in get_ptc() for that slot                 │   │
  │  │   • signature is invalid                                           │   │
  │  │                                                                     │   │
  │  │ IGNORE if:                                                          │   │
  │  │   • slot is not current slot (with clock disparity allowance)      │   │
  │  │   • Already seen valid message from this validator                 │   │
  │  │   • beacon_block_root not yet seen                                 │   │
  │  │   • referenced block is not at data.slot                           │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  proposer_preferences                                                       │
  │  ════════════════════                                                       │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │ REJECT if:                                                          │   │
  │  │   • validator is not proposer for preferences.proposal_slot         │   │
  │  │   • signature is invalid                                            │   │
  │  │                                                                     │   │
  │  │ IGNORE if:                                                          │   │
  │  │   • proposal_slot outside proposer lookahead or already passed      │   │
  │  │   • dependent_root not yet seen                                     │   │
  │  │   • already seen valid preferences for same proposer slot/root      │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  6.3 DataColumnSidecar Changes

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                    DataColumnSidecar: BEFORE vs AFTER                       │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  BEFORE (Fulu)                          AFTER (GLOAS)                       │
  │  ══════════════                         ═════════════                       │
  │                                                                             │
  │  DataColumnSidecar {                    DataColumnSidecar {                 │
  │    index                                  index                             │
  │    column                                 column                            │
  │    kzg_commitments                        kzg_proofs                        │
  │    kzg_proofs                                                               │
  │                                                                             │
  │    signed_block_header ❌ REMOVED                                           │
  │    kzg_commitments ❌ REMOVED (now in bid)                                  │
  │    kzg_commitments_inclusion_proof ❌                                       │
  │                                                                             │
  │                                           slot ✅ NEW                       │
  │                                           beacon_block_root ✅ NEW          │
  │  }                                      }                                   │
  │                                                                             │
  │  WHY THIS CHANGE?                                                           │
  │  ════════════════                                                           │
  │                                                                             │
  │  BEFORE: Sidecars needed to prove they came from a specific block           │
  │          via signed_block_header and merkle inclusion proof                 │
  │          → Required proposer signature                                      │
  │          → Proposer was responsible for blob distribution                   │
  │                                                                             │
  │  AFTER:  BUILDER distributes sidecars, not proposer!                        │
  │          Verification is done differently:                                  │
  │          1. Get bid from beacon block                                       │
  │          2. KZG commitments are in bid.blob_kzg_commitments directly       │
  │          3. Verify slot and beacon_block_root match                         │
  │                                                                             │
  │          → No proposer signature needed on sidecars                         │
  │          → Simpler structure                                                │
  │          → Builder takes responsibility for DA                              │
  │                                                                             │
  │  VALIDATION NOW:                                                            │
  │  ═══════════════                                                            │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  def verify_data_column_sidecar(sidecar, kzg_commitments):           │   │
  │  │      # Index must be valid                                          │   │
  │  │      if sidecar.index >= NUMBER_OF_COLUMNS: return False            │   │
  │  │                                                                     │   │
  │  │      # Must have blobs                                              │   │
  │  │      if len(sidecar.column) == 0: return False                      │   │
  │  │                                                                     │   │
  │  │      # Consistent lengths (kzg_commitments passed from bid)         │   │
  │  │      if len(sidecar.column) != len(kzg_commitments): return False   │   │
  │  │      if len(sidecar.column) != len(sidecar.kzg_proofs): return False│   │
  │  │                                                                     │   │
  │  │      return True                                                    │   │
  │  │                                                                     │   │
  │  │  # On gossip, ALSO check:                                           │   │
  │  │  # - A valid block for the sidecar's slot has been seen             │   │
  │  │  #   (SHOULD queue for deferred validation if not yet seen)         │   │
  │  │  # - slot matches block slot                                        │   │
  │  │  # - verify using bid.blob_kzg_commitments from the block           │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  6.4 New Req/Resp Methods

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                    NEW REQ/RESP PROTOCOL METHODS                            │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  ExecutionPayloadEnvelopesByRange v1 ✨ NEW                                 │
  │  ══════════════════════════════════════════                                 │
  │                                                                             │
  │  Protocol: /eth2/beacon_chain/req/execution_payload_envelopes_by_range/1/   │
  │                                                                             │
  │  Request:                          Response:                                │
  │  ┌────────────────────┐            ┌────────────────────────────────────┐   │
  │  │ start_slot: Slot   │ ──────────►│ List[SignedExecutionPayloadEnvelope│   │
  │  │ count: uint64      │            │      MAX_REQUEST_BLOCKS_DENEB]     │   │
  │  └────────────────────┘            └────────────────────────────────────┘   │
  │                                                                             │
  │  Use case: Syncing execution payloads for a range of slots                  │
  │            (like BeaconBlocksByRange, but for payloads)                     │
  │                                                                             │
  │  ─────────────────────────────────────────────────────────────────────────  │
  │                                                                             │
  │  ExecutionPayloadEnvelopesByRoot v1 ✨ NEW                                  │
  │  ═════════════════════════════════════════                                  │
  │                                                                             │
  │  Protocol: /eth2/beacon_chain/req/execution_payload_envelopes_by_root/1/    │
  │                                                                             │
  │  Request:                          Response:                                │
  │  ┌────────────────────┐            ┌────────────────────────────────────┐  │
  │  │ List[Root,         │ ──────────►│ List[SignedExecutionPayloadEnvelope│  │
  │  │   MAX_REQUEST_     │            │      MAX_REQUEST_PAYLOADS]         │  │
  │  │   PAYLOADS]        │            │                                    │  │
  │  │                    │            │ (MAX_REQUEST_PAYLOADS = 128)       │  │
  │  └────────────────────┘            └────────────────────────────────────┘  │
  │                                                                             │
  │  Use case: Requesting specific payloads by beacon block root                │
  │            e.g., when you see a PTC vote saying "payload present"           │
  │            but you never received the payload                               │
  │                                                                             │
  │  WHY NEW METHODS?                                                           │
  │  ════════════════                                                           │
  │                                                                             │
  │  BEFORE: Beacon blocks contained execution payloads                         │
  │          → BeaconBlocksByRange/Root got you everything                      │
  │                                                                             │
  │  AFTER:  Beacon blocks and execution payloads are SEPARATE                  │
  │          → Need separate methods to request payloads                        │
  │          → Client might have block but missing payload                      │
  │          → Client might need to catch up on missed payloads                 │
  │                                                                             │
  │  SPEC: p2p-interface.md                                                     │
  │  CONSTANTS: MAX_REQUEST_PAYLOADS = 128                                      │
  │  GOSSIP: beacon_block, proposer_preferences, execution_payload_bid,         │
  │          execution_payload, payload_attestation_message,                    │
  │          data_column_sidecar_{subnet_id}                                    │
  │  REQ/RESP: ExecutionPayloadEnvelopesByRange v1 (new)                        │
  │            ExecutionPayloadEnvelopesByRoot v1 (new)                         │
  │            BeaconBlocksByRange v2, BeaconBlocksByRoot v2 (updated)          │
  │  CONTAINERS: DataColumnSidecar (modified - removed signed_block_header)     │
  │  FUNCTIONS: compute_fork_version(), verify_data_column_sidecar()            │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  ---
```

## 7. ACTOR PERSPECTIVES

```text
  7.1 How to Become a Builder

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                    BECOMING A BUILDER: Step by Step                         │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  IMPORTANT: Builders are NOT validators!                                    │
  │  ═══════════════════════════════════════                                    │
  │                                                                             │
  │  In GLOAS, builders are separate staked actors with their own registry.     │
  │  They deposit stake but do NOT perform validation duties.                   │
  │                                                                             │
  │  ┌───────────────────────────────────────────────────────────────────────┐ │
  │  │                                                                       │ │
  │  │  VALIDATORS                          BUILDERS                         │ │
  │  │  ══════════                          ════════                         │ │
  │  │  • Stored in state.validators        • Stored in state.builders       │ │
  │  │  • Use ValidatorIndex                • Use BuilderIndex               │ │
  │  │  • Attest, propose, sync committee   • Only build & reveal payloads   │ │
  │  │  • Earn attestation rewards          • Earn MEV profits               │ │
  │  │  • Can be slashed for misdeeds       • Stake covers bid obligations   │ │
  │  │  • MIN_ACTIVATION_BALANCE = 32 ETH   • MIN_DEPOSIT_AMOUNT (1 ETH) min  │ │
  │  │                                                                       │ │
  │  └───────────────────────────────────────────────────────────────────────┘ │
  │                                                                             │
  │  STEP 1: Submit Builder Deposit                                             │
  │  ══════════════════════════════                                             │
  │                                                                             │
  │  Builders deposit via the same deposit contract. New builders use a         │
  │  builder withdrawal credential format, subject to routing checks:           │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │ deposit_data = DepositData(                                         │   │
  │  │     pubkey = your_bls_pubkey,                                       │   │
  │  │     withdrawal_credentials = 0x03 + 0x00*11 + your_execution_addr,  │   │
  │  │     amount = MIN_DEPOSIT_AMOUNT or more,  # At least 1 ETH           │   │
  │  │     signature = sign(deposit_data)                                  │   │
  │  │ )                                                                   │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  The 0x03 prefix is the builder-withdrawal credential prefix, but routing  │
  │  still depends on the pubkey's existing state (see below).                 │
  │                                                                             │
  │  ROUTING RULES:                                                            │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │ • Existing builder pubkey: top up builder balance, any prefix       │   │
  │  │ • Existing validator pubkey: route to validator side                │   │
  │  │ • New pubkey + 0x03 + no pending validator deposit: new builder     │   │
  │  │ • New pubkey with pending validator deposit: stay validator-side    │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  STEP 2: Builder Entry Created                                              │
  │  ═════════════════════════════                                              │
  │                                                                             │
  │  When routed as a new builder deposit, a Builder entry is created:          │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │ Builder(                                                            │   │
  │  │     pubkey: BLSPubkey,              # Your signing key              │   │
  │  │     version: uint8,                 # For future upgrades           │   │
  │  │     execution_address: Address,     # Where you receive payments    │   │
  │  │     balance: Gwei,                  # Your staked balance           │   │
  │  │     deposit_epoch: Epoch,           # When you deposited            │   │
  │  │     withdrawable_epoch: Epoch,      # When you can withdraw (exit)  │   │
  │  │ )                                                                   │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  STEP 3: Become Active After Deposit Epoch Finalizes                        │
  │  ════════════════════════════════════════════════════                       │
  │                                                                             │
  │  Unlike validators, builders don't need MIN_ACTIVATION_BALANCE.             │
  │  A builder is active once their deposit_epoch is finalized. Then they can:  │
  │  • Submit bids to proposers                                                 │
  │  • Have bids included in blocks (if sufficient balance)                     │
  │  • Reveal payloads and earn MEV                                             │
  │  • Pay proposers from your staked balance                                   │
  │                                                                             │
  │  KEY DIFFERENCES:                                                           │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  Builders:                        Validators:                       │   │
  │  │  • Separate registry              • Validator registry              │   │
  │  │  • BuilderIndex type              • ValidatorIndex type             │   │
  │  │  • No validation duties           • Must attest, propose, etc.      │   │
  │  │  • MIN_DEPOSIT_AMOUNT (1 ETH)     • Need 32 ETH to activate         │   │
  │  │  • 8192 epoch exit delay          • Validator withdrawability delay │   │
  │  │  • CAN submit execution bids      • CAN self-build (special index)  │   │
  │  │  • CAN pay proposers              • N/A                             │   │
  │  │                                                                     │   │
  │  │  Self-Build: Proposers can build their own payloads using the       │   │
  │  │  special BUILDER_INDEX_SELF_BUILD value (no builder needed).        │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  EXITING AS A BUILDER:                                                      │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  1. Initiate exit via voluntary exit message                        │   │
  │  │  2. Only valid if pending builder balance to withdraw is zero       │   │
  │  │  3. Wait MIN_BUILDER_WITHDRAWABILITY_DELAY (8192 epochs)            │   │
  │  │  4. Balance withdrawn to execution_address                          │   │
  │  │                                                                     │   │
  │  │  Delay prevents sweep-stalling from repeated builder deposit/exit   │   │
  │  │  cycles and leaves margin for payment settlement.                   │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  7.2 Builder Activity: Submitting Bids and Payloads

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                    BUILDER WORKFLOW: Bids and Payloads                      │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  PHASE 0: Get Proposer Preferences (NEW!)                                   │
  │  ═══════════════════════════════════════                                    │
  │                                                                             │
  │  For trustless gossip bids, know the proposer's preferences first!         │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  Listen to "proposer_preferences" gossip topic...                   │   │
  │  │                                                                     │   │
  │  │  Proposers may broadcast preferences for lookahead slots:          │   │
  │  │                                                                     │   │
  │  │  ProposerPreferences {                                              │   │
  │  │      dependent_root: Root,          # Lookahead-dependent root       │   │
  │  │      proposal_slot: Slot,           # Which slot they're proposing  │   │
  │  │      validator_index: ValidatorIndex,                               │   │
  │  │      fee_recipient: ExecutionAddress, # Where they want payment     │   │
  │  │      target_gas_limit: uint64,        # Their preferred gas limit   │   │
  │  │  }                                                                  │   │
  │  │                                                                     │   │
  │  │  IMPORTANT: fee_recipient must match, and gas_limit must be         │   │
  │  │  target-compatible with the proposer's target_gas_limit.            │   │
  │  │                                                                     │   │
  │  │  If no preferences received → no trustless bid for that slot        │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  PHASE 1: Construct the Payload (before slot)                               │
  │  ════════════════════════════════════════════                               │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  1. Call execution engine: engine_getPayloadV6                      │   │
  │  │     Use proposer's target_gas_limit from preferences!               │   │
  │  │                                                                     │   │
  │  │  2. Receive:                                                        │   │
  │  │     • execution_payload (transactions, withdrawals, etc.)           │   │
  │  │     • blobs_bundle (blobs, commitments, proofs)                     │   │
  │  │     • execution_requests                                            │   │
  │  │     • block_value (MEV extracted)                                   │   │
  │  │                                                                     │   │
  │  │  3. Store this payload - you'll need it later!                      │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  PHASE 2: Create and Broadcast Bid                                          │
  │  ═════════════════════════════════                                          │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  bid = ExecutionPayloadBid(                                         │   │
  │  │      parent_block_hash = payload.parent_hash,                       │   │
  │  │      parent_block_root = hash_tree_root(state.latest_block_header), │   │
  │  │      block_hash = payload.block_hash,          # COMMITMENT!        │   │
  │  │      prev_randao = payload.prev_randao,                             │   │
  │  │      fee_recipient = preferences.fee_recipient, # From preferences! │   │
  │  │      gas_limit = payload.gas_limit,             # Target-compatible │   │
  │  │      builder_index = my_builder_index,          # BuilderIndex type │   │
  │  │      slot = target_slot,                       # Current or next    │   │
  │  │      value = payment_amount,                   # What I'll pay      │   │
  │  │      execution_payment = 0,                    # Must be 0 for gossip│   │
  │  │      blob_kzg_commitments = blobs.commitments, │   │
  │  │      execution_requests_root = hash_tree_root(execution_requests),  │   │
  │  │  )                                                                  │   │
  │  │                                                                     │   │
  │  │  signed_bid = SignedExecutionPayloadBid(                            │   │
  │  │      message = bid,                                                 │   │
  │  │      signature = sign(bid, DOMAIN_BEACON_BUILDER)                   │   │
  │  │  )                                                                  │   │
  │  │                                                                     │   │
  │  │  broadcast(signed_bid) on "execution_payload_bid" topic             │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  PHASE 3: Watch for Beacon Block                                            │
  │  ═══════════════════════════════                                            │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  Listen to "beacon_block" topic...                                  │   │
  │  │                                                                     │   │
  │  │  When block arrives:                                                │   │
  │  │    included_bid = block.body.signed_execution_payload_bid.message   │   │
  │  │                                                                     │   │
  │  │    if included_bid.builder_index == my_builder_index:               │   │
  │  │        # MY BID WAS SELECTED!                                       │   │
  │  │        # Reveal payload; child block settles payment if FULL parent  │   │
  │  │        proceed to Phase 4                                           │   │
  │  │    else:                                                            │   │
  │  │        # Different builder won, I keep my payload                   │   │
  │  │        # No payment, no reveal needed                               │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  PHASE 4: Reveal Payload (if your bid was selected)                         │
  │  ══════════════════════════════════════════════════                         │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  envelope = ExecutionPayloadEnvelope(                               │   │
  │  │      payload = stored_payload,           # From Phase 1             │   │
  │  │      execution_requests = stored_requests,                          │   │
  │  │      builder_index = my_builder_index,   # BuilderIndex type        │   │
  │  │      beacon_block_root = hash_tree_root(received_block),            │   │
  │  │      parent_beacon_block_root = received_block.parent_root,         │   │
  │  │  )                                                                  │   │
  │  │                                                                     │   │
  │  │  signed_envelope = SignedExecutionPayloadEnvelope(                  │   │
  │  │      message = envelope,                                            │   │
  │  │      signature = sign(envelope, DOMAIN_BEACON_BUILDER)              │   │
  │  │  )                                                                  │   │
  │  │                                                                     │   │
  │  │  broadcast(signed_envelope) on "execution_payload" topic            │   │
  │  │                                                                     │   │
  │  │  ALSO broadcast DataColumnSidecars (blobs) if any!                  │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  WHAT IF YOU DON'T REVEAL?                                                  │
  │  ════════════════════════                                                   │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  If builder doesn't reveal payload on a timely head:                │   │
  │  │                                                                     │   │
  │  │  • PTC votes "payload_present = false"                              │   │
  │  │  • Payment still goes through if same-slot quorum reached!          │   │
  │  │  • Builder loses MEV opportunity                                    │   │
  │  │  • But proposer still gets paid (builder's stake)                   │   │
  │  │                                                                     │   │
  │  │  Honest withholding is allowed if the referenced block was not      │   │
  │  │  timely and is not the builder's head.                              │   │
  │  │                                                                     │   │
  │  │  Payment finalization depends on same-slot attestation quorum       │   │
  │  │  Builders can't grief proposers if quorum is reached                │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  SPEC: builder.md                                                           │
  │  FUNCTIONS: get_execution_payload_bid_signature()                           │
  │             get_execution_payload_envelope_signature()                      │
  │             get_data_column_sidecars(), get_data_column_sidecars_from_block()│
  │  CONSTANTS: BUILDER_WITHDRAWAL_PREFIX = 0x03                                │
  │             DOMAIN_BEACON_BUILDER = DomainType('0x0B000000')                │
  │  BEACON API:                                                                │
  │    GET  /eth/v1/validator/execution_payload_bid/{slot}/{builder_index}      │
  │         → Builder retrieves their bid to sign                               │
  │    POST /eth/v1/beacon/execution_payload_bid                                │
  │         → Publishes signed bid to network                                   │
  │    GET  /eth/v1/validator/execution_payload_envelope/{slot}/{builder_index} │
  │         → Builder retrieves their envelope to sign after seeing block       │
  │    POST /eth/v1/beacon/execution_payload_envelope                           │
  │         → Publishes signed envelope to network                              │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  7.3 Validator Perspective: New Duties

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                    VALIDATOR CHANGES IN GLOAS                               │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  NEW DUTY: Payload Timeliness Committee (PTC)                               │
  │  ════════════════════════════════════════════                               │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  At start of each epoch, check:                                     │   │
  │  │                                                                     │   │
  │  │  assignment = get_ptc_assignment(state, next_epoch, my_index)       │   │
  │  │                                                                     │   │
  │  │  if assignment is not None:                                         │   │
  │  │      # I'm on PTC duty for slot `assignment`!                       │   │
  │  │      # Must vote on timeliness + blob DA at 75% of that slot        │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  PTC VOTING WORKFLOW:                                                       │
  │  ════════════════════                                                       │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  At 75% into my assigned slot:                                      │   │
  │  │                                                                     │   │
  │  │  1. Check: Did I see a beacon block for this slot?                  │   │
  │  │     └─ NO: Don't submit attestation (ignored anyway)                │   │
  │  │     └─ YES: Continue...                                             │   │
  │  │                                                                     │   │
  │  │  2. Check: Did I see its envelope before get_payload_due_ms()?      │   │
  │  │     └─ NO: Set payload_present = false                              │   │
  │  │     └─ YES: Set payload_present = true                              │   │
  │  │                                                                     │   │
  │  │  3. Create attestation:                                             │   │
  │  │                                                                     │   │
  │  │     msg = PayloadAttestationMessage(                                │   │
  │  │         validator_index = my_index,                                 │   │
  │  │         data = PayloadAttestationData(                              │   │
  │  │             beacon_block_root = seen_block_root,                    │   │
  │  │             slot = current_slot,                                    │   │
  │  │             payload_present = true/false,                           │   │
  │  │             blob_data_available = ...,                              │   │
  │  │         ),                                                          │   │
  │  │         signature = sign(data, DOMAIN_PTC_ATTESTER)                 │   │
  │  │     )                                                               │   │
  │  │                                                                     │   │
  │  │  4. Broadcast on "payload_attestation_message" topic                │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  ───────────────────────────────────────────────────────────────────────── │
  │                                                                             │
  │  MODIFIED DUTY: Attestations Now Signal Payload Status                      │
  │  ═════════════════════════════════════════════════════                      │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  When creating an attestation:                                      │   │
  │  │                                                                     │   │
  │  │  if attesting to block from CURRENT slot:                           │   │
  │  │      data.index = 0   # Always 0 for same-slot                      │   │
  │  │                                                                     │   │
  │  │  else:  # Attesting to block from PREVIOUS slot                     │   │
  │  │      check fork choice for that block's payload status:             │   │
  │  │                                                                     │   │
  │  │      if payload_status == EMPTY:                                    │   │
  │  │          data.index = 0   # Signal "I saw EMPTY block"              │   │
  │  │      elif payload_status == FULL:                                   │   │
  │  │          data.index = 1   # Signal "I saw FULL block"               │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  ───────────────────────────────────────────────────────────────────────── │
  │                                                                             │
  │  MODIFIED DUTY: Block Proposal                                              │
  │  ═════════════════════════════                                              │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  BEFORE: Proposer builds or gets full block from relay              │   │
  │  │                                                                     │   │
  │  │  AFTER:                                                             │   │
  │  │                                                                     │   │
  │  │  1. Listen to "execution_payload_bid" topic (or out-of-band)        │   │
  │  │                                                                     │   │
  │  │  2. Select a head and call should_build_on_full(store, head)        │   │
  │  │                                                                     │   │
  │  │  3. Select a bid (highest value? most trusted builder?)             │   │
  │  │                                                                     │   │
  │  │  4. Verify bid is valid:                                            │   │
  │  │     • Builder index is valid and active                             │   │
  │  │     • Builder has sufficient balance                                │   │
  │  │     • Bid slot matches current slot                                 │   │
  │  │     • Bid parent root matches my parent_root                        │   │
  │  │     • Bid parent hash matches FULL/EMPTY parent choice              │   │
  │  │     • Signature valid (unless self-build with infinity sig)         │   │
  │  │                                                                     │   │
  │  │  5. Include bid in block:                                           │   │
  │  │     block.body.signed_execution_payload_bid = selected_bid          │   │
  │  │                                                                     │   │
  │  │  6. Include payload attestations from previous slot:                │   │
  │  │     block.body.payload_attestations = aggregate(ptc_messages)       │   │
  │  │                                                                     │   │
  │  │  7. Set parent_execution_requests from store.payloads if FULL,      │   │
  │  │     otherwise use empty ExecutionRequests()                         │   │
  │  │                                                                     │   │
  │  │  8. NO LONGER include execution_payload in block!                   │   │
  │  │                                                                     │   │
  │  │  9. NO LONGER responsible for DataColumnSidecar distribution        │   │
  │  │     (builder does this now)                                         │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  SELF-BUILDING (Proposer == Builder):                                       │
  │  ════════════════════════════════════                                       │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  If proposer wants to build their own block:                        │   │
  │  │                                                                     │   │
  │  │  bid = ExecutionPayloadBid(                                         │   │
  │  │      builder_index = BUILDER_INDEX_SELF_BUILD,  # UINT64_MAX        │   │
  │  │      value = 0,                        # No payment to self         │   │
  │  │      ...other fields...                                             │   │
  │  │  )                                                                  │   │
  │  │                                                                     │   │
  │  │  signed_bid = SignedExecutionPayloadBid(                            │   │
  │  │      message = bid,                                                 │   │
  │  │      signature = BLS.G2_POINT_AT_INFINITY  # Special: no sig needed │   │
  │  │  )                                                                  │   │
  │  │                                                                     │   │
  │  │  Self-builds don't need 0x03 credentials - any proposer can do it!  │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  SPEC: validator.md                                                         │
  │  FUNCTIONS: get_ptc_assignment(), get_payload_attestation_message_signature()│
  │             prepare_execution_payload() (modified)                          │
  │  CONSTANTS: DOMAIN_PTC_ATTESTER = DomainType('0x0C000000')                  │
  │             PAYLOAD_ATTESTATION_DUE_BPS = 7500 (75% into slot)              │
  │  CONTAINERS: PayloadAttestationMessage, PayloadAttestationData              │
  │  BEACON API:                                                                │
  │    GET  /eth/v4/validator/blocks/{slot}                                     │
  │         → Proposer retrieves unsigned gloas BeaconBlock to sign             │
  │    POST /eth/v2/beacon/blocks                                               │
  │         → Proposer publishes signed BeaconBlock to network                  │
  │    POST /eth/v1/validator/duties/ptc/{epoch}                                │
  │         → Validator checks PTC duty assignments for the epoch               │
  │    GET  /eth/v1/validator/payload_attestation_data/{slot}                   │
  │         → PTC member retrieves payload attestation data to sign             │
  │    POST /eth/v1/beacon/pool/payload_attestations                            │
  │         → PTC member submits signed payload attestation                     │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  ---
```

## 8. PUTTING IT ALL TOGETHER

```text
  8.1 Complete Message Flow Diagram

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                    COMPLETE GLOAS SLOT FLOW                                 │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  SLOT N-1 (Preparation)                                                     │
  │  ══════════════════════                                                     │
  │                                                                             │
  │      ┌──────────┐                                                           │
  │      │ PROPOSER │ Broadcasts SignedProposerPreferences                      │
  │      └────┬─────┘                                                           │
  │           │                                                                 │
  │           ▼                                                                 │
  │      ┌──────────┐                                                           │
  │      │ BUILDER  │ Constructs payload, creates bid                           │
  │      └────┬─────┘                                                           │
  │           │                                                                 │
  │           │ SignedExecutionPayloadBid                                       │
  │           │ (via P2P or direct to proposer)                                 │
  │           ▼                                                                 │
  │      ┌──────────┐                                                           │
  │      │ PROPOSER │ Receives and stores best bids                             │
  │      └──────────┘                                                           │
  │                                                                             │
  │  SLOT N: 0% (0 seconds) - BLOCK PROPOSAL                                    │
  │  ═══════════════════════════════════════                                    │
  │                                                                             │
  │      ┌──────────┐                                                           │
  │      │ PROPOSER │                                                           │
  │      └────┬─────┘                                                           │
  │           │ Creates BeaconBlock containing:                                 │
  │           │ • Selected bid (signed_execution_payload_bid)                   │
  │           │ • Payload attestations from slot N-1                            │
  │           │ • Parent execution requests if building on FULL parent          │
  │           │                                                                 │
  │           │ SignedBeaconBlock                                               │
  │           ▼                                                                 │
  │     ┌─────────────────────────────────────────────────────────────────┐     │
  │     │                         P2P NETWORK                             │     │
  │     │                     "beacon_block" topic                        │     │
  │     └─────────────────────────────────────────────────────────────────┘     │
  │           │                   │                   │                         │
  │           ▼                   ▼                   ▼                         │
  │      ┌─────────┐        ┌──────────┐        ┌───────────┐                   │
  │      │ BUILDER │        │VALIDATORS│        │   NODES   │                   │
  │      │ (sees   │        │(store    │        │ (add to   │                   │
  │      │ their   │        │ block)   │        │ fork      │                   │
  │      │ bid!)   │        │          │        │ choice)   │                   │
  │      └────┬────┘        └──────────┘        └───────────┘                   │
  │           │                                                                 │
  │  SLOT N: ~5-15% - PAYLOAD REVEAL                                            │
  │  ═══════════════════════════════                                            │
  │                                                                             │
  │           │ SignedExecutionPayloadEnvelope                                  │
  │           │ + DataColumnSidecars (if blobs)                                 │
  │           ▼                                                                 │
  │     ┌─────────────────────────────────────────────────────────────────┐     │
  │     │                         P2P NETWORK                             │     │
  │     │              "execution_payload" + "data_column_sidecar_*"      │     │
  │     └─────────────────────────────────────────────────────────────────┘     │
  │           │                   │                   │                         │
  │           ▼                   ▼                   ▼                         │
  │      ┌─────────┐        ┌──────────┐        ┌───────────┐                   │
  │      │   PTC   │        │VALIDATORS│        │   NODES   │                   │
  │      │(waiting │        │(waiting  │        │ (process  │                   │
  │      │ to vote)│        │ to vote) │        │ verify +  │                   │
  │      │         │        │          │        │ store env)│                   │
  │      └─────────┘        └──────────┘        └───────────┘                   │
  │                                                                             │
  │  SLOT N: 25% (3 seconds) - ATTESTATION DEADLINE                             │
  │  ═══════════════════════════════════════════════                            │
  │                                                                             │
  │      ┌──────────┐                                                           │
  │      │VALIDATORS│                                                           │
  │      └────┬─────┘                                                           │
  │           │ Attestations with index=0 (same slot)                           │
  │           │ or index=0/1 (if attesting to prev slot based on payload)       │
  │           ▼                                                                 │
  │     ┌─────────────────────────────────────────────────────────────────┐     │
  │     │                         P2P NETWORK                             │     │
  │     │                  "beacon_attestation_*" topics                  │     │
  │     └─────────────────────────────────────────────────────────────────┘     │
  │                                                                             │
  │  SLOT N: 75% (9 seconds) - PTC DEADLINE                                     │
  │  ══════════════════════════════════════                                     │
  │                                                                             │
  │      ┌──────────┐                                                           │
  │      │   PTC    │ 512 positions vote on timeliness + blob DA                │
  │      │ (512     │                                                           │
  │      │positions)│                                                           │
  │      └────┬─────┘                                                           │
  │           │ PayloadAttestationMessage                                       │
  │           │ (payload_present = true/false)                                  │
  │           ▼                                                                 │
  │     ┌─────────────────────────────────────────────────────────────────┐     │
  │     │                         P2P NETWORK                             │     │
  │     │               "payload_attestation_message" topic               │     │
  │     └─────────────────────────────────────────────────────────────────┘     │
  │           │                                                                 │
  │           ▼                                                                 │
  │      ┌──────────────┐                                                       │
  │      │ NEXT PROPOSER│ Collects and aggregates PTC messages                  │
  │      │ (slot N+1)   │ for inclusion in their block                          │
  │      └──────────────┘                                                       │
  │                                                                             │
  │  SLOT N+1: 0% - NEXT BLOCK                                                  │
  │  ═════════════════════════                                                  │
  │                                                                             │
  │      Next block contains:                                                   │
  │      • payload_attestations (aggregated from slot N)                        │
  │      • parent_execution_requests from slot N payload if building on FULL    │
  │      • New signed_execution_payload_bid for slot N+1                        │
  │      • State processes slot N payload only when parent chosen as FULL       │
  │                                                                             │
  │  EPOCH BOUNDARY - PAYMENT PROCESSING                                        │
  │  ═══════════════════════════════════                                        │
  │                                                                             │
  │      ┌──────────────────────────────────────────────────────────────────┐   │
  │      │                                                                  │   │
  │      │  For each pending_payment from previous epoch:                   │   │
  │      │                                                                  │   │
  │      │    if payment.weight >= quorum (60% of per-slot stake):          │   │
  │      │        → Move to builder_pending_withdrawals                     │   │
  │      │        → Builder will pay proposer!                              │   │
  │      │    else:                                                         │   │
  │      │        → Discard payment                                         │   │
  │      │        → Builder keeps stake                                     │   │
  │      │        → (Maybe attack detected, or network issues)              │   │
  │      │                                                                  │   │
  │      └──────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  BEACON API SUMMARY (EXTERNAL / PENDING, NOT IN LOCAL CONSENSUS SPECS):    │
  │  ═══════════════════════════════════════════════════════════════════════    │
  │  Local Gloas specs define gossip and req/resp; HTTP APIs may differ.       │
  │                                                                             │
  │  BeaconBlock:                                                               │
  │    GET  /eth/v4/validator/blocks/{slot}         → Get block to sign         │
  │    POST /eth/v2/beacon/blocks                   → Publish signed block      │
  │                                                                             │
  │  ExecutionPayloadBid:                                                       │
  │    GET  /eth/v1/validator/execution_payload_bid/{slot}/{builder_index}      │
  │    POST /eth/v1/beacon/execution_payload_bid                                │
  │                                                                             │
  │  ExecutionPayloadEnvelope:                                                  │
  │    GET  /eth/v1/validator/execution_payload_envelope/{slot}/{builder_index} │
  │    POST /eth/v1/beacon/execution_payload_envelope                           │
  │                                                                             │
  │  PTC (Payload Timeliness Committee):                                        │
  │    POST /eth/v1/validator/duties/ptc/{epoch}    → Get PTC duties            │
  │    GET  /eth/v1/validator/payload_attestation_data/{slot}                   │
  │    POST /eth/v1/beacon/pool/payload_attestations                            │
  │    GET  /eth/v1/beacon/pool/payload_attestations                            │
  │                                                                             │
  │  Reference: https://github.com/ethereum/beacon-APIs/pull/552                │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  8.2 Key Takeaways

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                          GLOAS KEY TAKEAWAYS                                │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  1. SEPARATION OF CONCERNS                                                  │
  │  ════════════════════════                                                   │
  │  • Proposers: Select best bid, create beacon block                          │
  │  • Builders: Construct payloads, reveal after block                         │
  │  • PTC: Vote on payload timeliness and blob data availability               │
  │  • All enforced at protocol level!                                          │
  │                                                                             │
  │  2. BID PAYMENT SETTLEMENT                                                  │
  │  ═════════════════════════                                                  │
  │  • Builder commits to payment via bid                                       │
  │  • If child builds on FULL parent: payment settles immediately              │
  │  • If payload withheld/ignored: payment still happens if quorum reached     │
  │  • Prevents griefing attacks on proposers                                   │
  │                                                                             │
  │  3. TWO-PHASE STATE TRANSITION                                              │
  │  ═════════════════════════════                                              │
  │  • Phase 1: Process beacon block (bid committed)                            │
  │  • Envelope arrival: verify + store payload for fork choice                 │
  │  • Child block: process parent payload via parent_execution_requests        │
  │  • Fork choice tracks EMPTY vs FULL versions                                │
  │                                                                             │
  │  4. NEW STAKED ACTOR TYPE: BUILDER                                          │
  │  ════════════════════════════════                                           │
  │  • 0x03 withdrawal credential prefix                                        │
  │  • Separate registry (state.builders), NOT validators                       │
  │  • Can submit bids and pay proposers                                        │
  │  • No validation duties (no attesting, no proposing)                        │
  │                                                                             │
  │  5. PAYLOAD TIMELINESS COMMITTEE (PTC)                                      │
  │  ═════════════════════════════════════                                      │
  │  • 512 PTC positions per slot                                               │
  │  • Balance-weighted selection (sybil resistant)                             │
  │  • Vote at 75% of slot                                                      │
  │  • Votes on payload timeliness and blob data availability                   │
  │  • Separate from same-slot attestation payment quorum                       │
  │                                                                             │
  │  6. MODIFIED ATTESTATIONS                                                   │
  │  ═════════════════════════                                                  │
  │  • data.index now signals payload status (0=empty, 1=full)                  │
  │  • Same-slot attestations always use index=0                                │
  │  • Contributes to payment weight accumulation                               │
  │                                                                             │
  │  7. SIMPLER BLOB DISTRIBUTION                                               │
  │  ═════════════════════════════                                              │
  │  • Builder distributes DataColumnSidecars (not proposer)                    │
  │  • Sidecar structure simplified (no header/proof/commitments)               │
  │  • KZG commitments now in bid, verified via bid.blob_kzg_commitments        │
  │                                                                             │
  │  8. NO MORE TRUSTED RELAYS                                                  │
  │  ═════════════════════════                                                  │
  │  • Protocol itself is the escrow                                            │
  │  • Cryptographic commitments replace trust                                  │
  │  • Decentralized and censorship resistant                                   │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  8.3 File-to-Concept Mapping

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                    WHICH FILE COVERS WHAT?                                  │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  ┌───────────────────┬──────────────────────────────────────────────────┐   │
  │  │ File              │ Key Concepts                                     │   │
  │  ├───────────────────┼──────────────────────────────────────────────────┤   │
  │  │ beacon-chain.md   │ • All new containers (Bid, Envelope, PTC types)  │   │
  │  │                   │ • Modified BeaconState and BeaconBlockBody       │   │
  │  │                   │ • State transition (process_block, etc.)         │   │
  │  │                   │ • Payment quorum and withdrawal logic            │   │
  │  │                   │ • Builder credentials (0x03 prefix)              │   │
  │  ├───────────────────┼──────────────────────────────────────────────────┤   │
  │  │ fork-choice.md    │ • PayloadStatus (PENDING/EMPTY/FULL)             │   │
  │  │                   │ • ForkChoiceNode with payload tracking           │   │
  │  │                   │ • Modified get_head() algorithm                  │   │
  │  │                   │ • on_execution_payload_envelope handler          │   │
  │  │                   │ • PTC timeliness + data availability votes       │   │
  │  │                   │ • Attestation index interpretation               │   │
  │  ├───────────────────┼──────────────────────────────────────────────────┤   │
  │  │ p2p-interface.md  │ • New gossip topics (bid, payload, PTC)          │   │
  │  │                   │ • Validation rules for each message type         │   │
  │  │                   │ • Modified DataColumnSidecar structure           │   │
  │  │                   │ • New req/resp methods for payloads              │   │
  │  ├───────────────────┼──────────────────────────────────────────────────┤   │
  │  │ validator.md      │ • New timing parameters (earlier deadlines)      │   │
  │  │                   │ • PTC assignment and voting workflow             │   │
  │  │                   │ • How to construct payload_attestations          │   │
  │  │                   │ • Modified block proposal (select bid)           │   │
  │  │                   │ • Attestation index signaling                    │   │
  │  ├───────────────────┼──────────────────────────────────────────────────┤   │
  │  │ builder.md        │ • Builder deposits and 0x03 routing rules        │   │
  │  │                   │ • Bid construction workflow                      │   │
  │  │                   │ • Payload envelope construction                  │   │
  │  │                   │ • DataColumnSidecar creation                     │   │
  │  │                   │ • Honest withholding (when to not reveal)        │   │
  │  ├───────────────────┼──────────────────────────────────────────────────┤   │
  │  │ fork.md           │ • GLOAS_FORK_VERSION and GLOAS_FORK_EPOCH        │   │
  │  │                   │ • upgrade_to_gloas() function                    │   │
  │  │                   │ • State migration from Fulu                      │   │
  │  │                   │ • onboard_builders_from_pending_deposits()       │   │
  │  ├───────────────────┼──────────────────────────────────────────────────┤   │
  │  │ fast-confirmation │ • get_node_for_root returns ForkChoiceNode       │   │
  │  │ .md               │ • Safe execution hash is confirmed parent hash   │   │
  │  ├───────────────────┼──────────────────────────────────────────────────┤   │
  │  │ weak-subjectivity │ • Weak subjectivity period formula updated for   │   │
  │  │ .md               │   EIP-8061 churn split                           │   │
  │  └───────────────────┴──────────────────────────────────────────────────┘   │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  ---
  Summary

  GLOAS is a consensus-layer upgrade centered on EIP-7732/ePBS, plus companion changes such as SLOTNUM, churn updates, and payload shape updates. The key innovation is replacing trusted relays with protocol-enforced commitments:

  1. Builders stake ETH and submit cryptographic commitments (bids)
  2. Proposers select bids and include them in beacon blocks
  3. Builders reveal payloads after seeing their bid was selected
  4. PTC (512 positions, duplicates possible) votes on payload timeliness and blob DA
  5. Protocol finalizes payment via same-slot attestation quorum

  This creates a trustless, decentralized block building market where:
  - Builders can't grief proposers if quorum is reached
  - Proposers can't steal MEV (builder reveals after block)
  - No centralized relay needed (protocol is the escrow)
```

---

## 9. SPEC REFERENCE INDEX

Detailed reference for constants, containers, and functions from the GLOAS specs. Entries marked explainer-only are local explanatory concepts, not normative spec helpers. Each entry explains not just what it is, but **why** it exists and how it fits into the design.

---

### 9.0 Custom Types

#### `BuilderIndex` (uint64)

**What:** An index into the builder registry (`state.builders`).

**Why it exists:** GLOAS introduces a completely separate registry for builders, distinct from the validator registry. Builders are specialized actors who construct execution payloads but do NOT perform validation duties (no attesting, no sync committee, no proposing). Having a separate index type prevents confusion and accidental mixing of builder and validator indices in the codebase. When you see `BuilderIndex`, you know it refers to `state.builders[i]`, not `state.validators[i]`.

---

#### `BlockAccessList`

**What:** `ByteList[MAX_BYTES_PER_TRANSACTION]` carrying an RLP-encoded block access list in the Gloas `ExecutionPayload`.

**Why it exists:** Gloas inherits the EIP-7928 payload shape. The bid does not carry the access list directly; it commits to it through `block_hash`, because `block_access_list` lives inside the execution payload.

---

### 9.1 Constants

#### Index Flags

**`BUILDER_INDEX_FLAG`** = `uint64(2**40)`

**What:** A bitwise flag used to mark an index as a `BuilderIndex`.

**Why it exists:** In some contexts (like withdrawal processing), both validators and builders can appear in the same data structures. This flag allows the protocol to distinguish between them. If `index & BUILDER_INDEX_FLAG != 0`, it's a builder; otherwise it's a validator. The value `2**40` is chosen to be larger than any realistic validator count while still fitting in uint64.

---

#### Domain Types

**`DOMAIN_BEACON_BUILDER`** = `DomainType('0x0B000000')`

**What:** Signing domain for builder messages (bids and payload envelopes).

**Why it exists:** BLS signatures in Ethereum use "domain separation" to prevent cross-protocol replay attacks. A signature valid for one purpose (e.g., an attestation) must not be valid for another (e.g., a bid). By giving builders their own domain, a builder's bid signature cannot be replayed as an attestation or vice versa. The `0x0B` prefix is unique to builder operations.

**`DOMAIN_PTC_ATTESTER`** = `DomainType('0x0C000000')`

**What:** Signing domain for Payload Timeliness Committee attestations.

**Why it exists:** PTC members vote on whether they saw the execution payload before the due time and whether blob data was available. These votes need their own domain because they serve a different purpose than regular attestations. Separating the domain prevents a regular attestation from being misinterpreted as a PTC vote.

**`DOMAIN_PROPOSER_PREFERENCES`** = `DomainType('0x0D000000')`

**What:** Signing domain for proposer preference messages.

**Why it exists:** Before builders can construct valid bids, they need to know the proposer's `fee_recipient` (where to send payment) and `target_gas_limit` preferences. Proposers may broadcast signed preferences for lookahead proposal slots so builders know what values to use. The separate domain ensures these preference messages can't be confused with other signed messages.

---

#### Misc Constants

**`BUILDER_INDEX_SELF_BUILD`** = `BuilderIndex(UINT64_MAX)`

**What:** A sentinel value indicating the proposer built the payload themselves.

**Why it exists:** Proposers don't have to use external builders—they can construct their own payloads (called "self-building"). When self-building, there's no actual builder in the registry, so we need a special marker. `UINT64_MAX` is chosen because it's an impossible index (no registry will ever have that many entries), making it unambiguous. Self-built blocks use a special signature (`G2_POINT_AT_INFINITY`) and don't require 0x03 credentials.

**`BUILDER_PAYMENT_THRESHOLD_NUMERATOR`** = `6` and **`BUILDER_PAYMENT_THRESHOLD_DENOMINATOR`** = `10`

**What:** Together these define the 60% quorum threshold for builder payments.

**Why it exists:** Builder payments shouldn't execute blindly—what if the proposer equivocated or the network partitioned? The protocol requires 60% of same-slot attestation weight before confirming a payment. This threshold balances:
- **Too low (e.g., 33%):** Payments could execute even during attacks or partitions
- **Too high (e.g., 90%):** Normal network latency might prevent legitimate payments

60% ensures honest supermajority agreement before money moves. The numerator/denominator split allows integer arithmetic without floating point.

---

#### Withdrawal Prefixes

**`BUILDER_WITHDRAWAL_PREFIX`** = `Bytes1('0x03')`

**What:** The first byte of withdrawal credentials used for new builder deposits.

**Why it exists:** Ethereum already has withdrawal credential prefixes: `0x00` for BLS credentials, `0x01` for execution layer credentials, `0x02` for compounding validators. GLOAS adds `0x03` for builder withdrawal credentials, but the prefix is not the only routing rule:
- Existing builder pubkey → top up `state.builders`, regardless of prefix
- Existing validator pubkey → validator-side deposit path
- New pubkey + `0x03` + no pending validator deposit → create a builder
- New pubkey blocked by a pending validator deposit → validator-side pending queue

This reuses existing deposit infrastructure while keeping builder and validator pubkeys from crossing registries accidentally.

---

#### Preset Constants

**`PTC_SIZE`** = `512`

**What:** Number of PTC positions in the Payload Timeliness Committee per slot.

**Why it exists:** The PTC votes on payload timeliness and blob data availability. 512 provides:
- **Security:** Hard to bribe/corrupt a significant fraction of randomly-selected positions
- **Efficiency:** Small enough that votes can be aggregated into blocks without excessive overhead
- **Decentralization:** Large enough sample from the full validator set

The value is chosen to balance security margins with practical constraints on block size and gossip overhead.

**`MAX_PAYLOAD_ATTESTATIONS`** = `4`

**What:** Maximum aggregated payload attestations per block.

**Why it exists:** PTC members might vote with different `PayloadAttestationData` (e.g., some saw payload before the due time, some didn't; some had blob data available, some didn't). Each unique data combination needs a separate aggregate. With 2 boolean fields (`payload_present`, `blob_data_available`), there are 4 possible combinations, and `MAX_PAYLOAD_ATTESTATIONS = 4` covers them.

**`BUILDER_REGISTRY_LIMIT`** = `2**40` (≈ 1 trillion)

**What:** Maximum builders allowed in `state.builders`.

**Why it exists:** State fields need explicit limits for SSZ serialization. The limit is set astronomically high because:
- There's no practical reason to cap builder count tightly
- Builders are market actors; we shouldn't artificially restrict competition
- The limit only affects worst-case state size calculations

**`BUILDER_PENDING_WITHDRAWALS_LIMIT`** = `2**20` (≈ 1 million)

**What:** Maximum pending builder payment withdrawals in state.

**Why it exists:** After a builder payment reaches quorum, it's queued for withdrawal (actual ETH transfer). This queue needs a limit. 1 million is generous—even with a block every 12 seconds, that's ~140 days of backlog at 1 payment per slot before hitting the limit.

**`MAX_BUILDERS_PER_WITHDRAWALS_SWEEP`** = `16,384`

**What:** Maximum builders processed per slot during withdrawal sweep.

**Why it exists:** Like validators, builders may have pending withdrawals (exiting the protocol). Processing happens via a "sweep" that iterates through builders. To bound per-slot computation, we limit how many builders are checked per sweep. 16,384 ensures the entire builder registry can be swept in reasonable time even if it grows large.

---

#### Configuration Constants

**`MIN_BUILDER_WITHDRAWABILITY_DELAY`** = `8192` epochs (≈ 36.4 days with 12s slots)

**What:** Minimum wait time before a builder can withdraw after initiating exit.

**Why it exists:** This delay prevents a sweep-stalling attack where an attacker repeatedly deposits and exits builders to block validator withdrawals. The current spec value gives a long exit buffer:

1. **Attack cost:** Repeated builder deposit/exit cycles keep capital locked for many epochs.
2. **Withdrawal safety:** Exiting builders remain around long enough for pending obligations to settle.
3. **Payment settlement:** Builder payments settle through the pending-payment and withdrawal queues before full builder withdrawal.

---

#### Fork Choice Constants

**`PAYLOAD_TIMELY_THRESHOLD`** = `256` (PTC_SIZE // 2)
**`DATA_AVAILABILITY_TIMELY_THRESHOLD`** = `256` (PTC_SIZE // 2)

**What:** Minimum PTC votes needed to consider a payload timely, and minimum PTC votes needed to consider blob data available.

**Why it exists:** Fork choice separately evaluates payload timeliness and blob data availability. If >256 of 512 PTC positions vote for the queried value, the corresponding view wins. Simple majority (50%+1) prevents:
- A minority of slow/offline PTC members from incorrectly marking a timely/available payload as late or unavailable
- An attacker from claiming timeliness or data availability by bribing only a minority

**`PAYLOAD_STATUS_EMPTY`** = `0`, **`PAYLOAD_STATUS_FULL`** = `1`, **`PAYLOAD_STATUS_PENDING`** = `2`

**What:** The three possible states of a block's payload in fork choice.

**Why they exist:** In GLOAS, a beacon block arrives first, then its payload arrives separately. Fork choice must track this two-phase process:
- **PENDING:** Block seen, waiting to see if payload arrives
- **EMPTY:** Block seen, payload did NOT arrive (builder withheld or too slow)
- **FULL:** Block seen AND payload received and validated

This three-state model lets fork choice reason about partial blocks and choose the best chain considering payload availability.

**`ATTESTATION_TIMELINESS_INDEX`** = `0`, **`PTC_TIMELINESS_INDEX`** = `1`, **`NUM_BLOCK_TIMELINESS_DEADLINES`** = `2`

**What:** Indexes for the two block-timeliness deadlines tracked in fork choice.

**Why they exist:** Gloas has separate timeliness signals for regular attestations and PTC payload attestations. `NUM_BLOCK_TIMELINESS_DEADLINES = 2` sizes the per-block timeliness vector that tracks those separate deadlines.

---

#### Time Parameters

**`ATTESTATION_DUE_BPS_GLOAS`** = `2500` (25% = 3 seconds into slot)
**`SYNC_MESSAGE_DUE_BPS_GLOAS`** = `2500` (25% = 3 seconds into slot)

**What:** When attesters and sync committee members should submit their messages.

**Why it changed:** Pre-GLOAS, attestations were due at 33% (4 seconds). GLOAS moves attestations and sync messages earlier to 25% (3 seconds). Why?

The new slot timeline needs more time for:
1. Builder to see the block and release the payload
2. PTC to see the payload and vote
3. Next proposer to collect PTC votes

Moving attestations earlier frees up time later in the slot for these new activities.

**`AGGREGATE_DUE_BPS_GLOAS`** = `5000` (50% = 6 seconds into slot)
**`CONTRIBUTION_DUE_BPS_GLOAS`** = `5000` (50% = 6 seconds into slot)

**What:** When aggregate attestations and sync committee contributions are due.

**Why it changed:** Aggregates and contributions move to the middle of the slot so normal consensus aggregation finishes before the PTC payload-attestation deadline.

**`PAYLOAD_DUE_BPS`** = `7500` (75% = 9 seconds into slot)
**`PAYLOAD_ATTESTATION_DUE_BPS`** = `7500` (75% = 9 seconds into slot)

**What:** When the builder payload is considered due, and when PTC members must submit their payload attestations.

**Why this timing:** PTC members vote on whether they've seen the payload before `get_payload_due_ms()` and whether blob data is available. Waiting until 75% of the slot (9 seconds) gives builders maximum time to reveal their payload while still leaving 3 seconds for the next proposer to collect and aggregate PTC votes.

---

#### P2P Configuration

**`MAX_REQUEST_PAYLOADS`** = `128`

**What:** Maximum payloads requestable in one req/resp call.

**Why it exists:** GLOAS adds new req/resp methods for fetching execution payloads (since they're no longer embedded in blocks). This limit prevents a single request from overwhelming bandwidth while still allowing efficient batch syncing.

---

#### Fork Configuration

**`GLOAS_FORK_VERSION`** = `Version('0x07000000')`

**What:** The 4-byte fork identifier for GLOAS.

**Why it exists:** Each Ethereum fork has a unique version for domain separation. Signatures made under one fork version are invalid under another, preventing cross-fork replay attacks. `0x07` follows the sequence: Phase0 (`0x00`), Altair (`0x01`), Bellatrix (`0x02`), Capella (`0x03`), Deneb (`0x04`), Electra (`0x05`), Fulu (`0x06`), GLOAS (`0x07`).

**`GLOAS_FORK_EPOCH`** = `Epoch(18446744073709551615)` placeholder / TBD activation

**What:** The epoch when GLOAS activates.

**Why TBD:** Fork epochs are determined through community coordination after extensive testing. The current spec keeps the max-uint64 placeholder until an activation epoch is chosen.

---

### 9.2 Containers

#### New Containers

**`Builder`**
- **Fields:** `pubkey`, `version`, `execution_address`, `balance`, `deposit_epoch`, `withdrawable_epoch`

**Why it exists:** GLOAS creates a first-class role for block builders separate from validators. The `Builder` container stores everything needed to track a builder's identity and economic state:
- `pubkey`: Their BLS public key for signing bids and envelopes
- `execution_address`: Where they receive any refunds (from 0x03 credentials)
- `balance`: Their staked ETH that backs their bid commitments
- `deposit_epoch` / `withdrawable_epoch`: Lifecycle tracking for exit delays

Unlike validators, builders don't attest or propose—they only build payloads and get paid for them.

---

**`BuilderPendingPayment`**
- **Fields:** `weight`, `withdrawal`

**Why it exists:** Builder payments don't execute immediately—they wait for quorum confirmation. This container tracks a payment in limbo:
- `weight`: Accumulates as same-slot attestations arrive (Gwei of attesting stake)
- `withdrawal`: The actual payment details (amount, recipient) to execute if quorum is reached

At each epoch boundary, the protocol checks: did `weight` reach 60% threshold? If yes, move to `builder_pending_withdrawals`. If no (or expired), discard.

---

**`BuilderPendingWithdrawal`**
- **Fields:** `fee_recipient`, `amount`, `builder_index`

**Why it exists:** Once a payment reaches quorum, it's "confirmed" but not yet executed. This container queues the confirmed payment for actual ETH transfer:
- `fee_recipient`: The proposer's address that receives the payment
- `amount`: How much ETH to transfer
- `builder_index`: Which builder is paying (to deduct from their balance)

Withdrawals are processed during the withdrawal sweep, interleaved with regular validator withdrawals.

---

**`PayloadAttestationData`**
- **Fields:** `beacon_block_root`, `slot`, `payload_present`, `blob_data_available`

**Why it exists:** PTC members need to communicate what they observed. This is the data they sign:
- `beacon_block_root`: Which block they're attesting about
- `slot`: When (prevents replay across slots)
- `payload_present`: "Did I see the execution payload before `get_payload_due_ms()`?" (true/false)
- `blob_data_available`: "Is the blob data available for this block?" (true/false)

Two boolean fields allow PTC members to report partial delivery (payload but no blobs, or neither).

---

**`PayloadAttestation`** and **`PayloadAttestationMessage`**

**Why they exist:** Like regular attestations, payload attestations have two forms:
- `PayloadAttestationMessage`: Individual vote from one PTC member (gossiped on p2p)
- `PayloadAttestation`: Aggregated form with `aggregation_bits` bitfield (included in blocks)

Aggregation reduces block size—instead of 512 individual signatures, we get a few aggregates with combined BLS signatures.

---

**`IndexedPayloadAttestation`**
- **Fields:** `attesting_indices`, `data`, `signature`

**Why it exists:** For signature verification, clients need explicit validator indices rather than only aggregation bits. This container expands a `PayloadAttestation` into concrete attesting indices, making aggregate signature checks straightforward.

---

**`ExecutionPayloadBid`**
- **Fields:** `parent_block_hash`, `parent_block_root`, `block_hash`, `prev_randao`, `fee_recipient`, `gas_limit`, `builder_index`, `slot`, `value`, `execution_payment`, `blob_kzg_commitments`, `execution_requests_root`

**Why it exists:** This is the core of ePBS—the builder's cryptographic commitment. Key fields:
- `block_hash`: Commits to the EXACT payload the builder will reveal (can't change later)
- `value`: How much the builder will pay the proposer (from their staked balance)
- `blob_kzg_commitments`: The actual blob KZG commitments (moved here from the envelope, so they're committed at bid time)
- `execution_requests_root`: Commitment to the `ExecutionRequests` carried in the envelope and processed by the child block
- `fee_recipient` / `gas_limit`: Must match or be compatible with proposer's preferences (ensures builder respects proposer's wishes)
- `execution_payment`: Reserved for trusted out-of-protocol auctions (must be 0 for public gossip)

Once this bid is included in a timely head block, the builder is expected to reveal a payload matching `block_hash`. An honest builder may withhold if the referenced block was not timely and is not the builder's head; payment only settles through FULL-parent processing or the quorum path.

---

**`SignedExecutionPayloadBid`** and **`SignedExecutionPayloadEnvelope`**

**Why they exist:** Both bids and envelopes need signatures to prove authenticity:
- Bid signature proves the builder authorized this commitment
- Envelope signature proves the builder authored the revealed payload

Without signatures, anyone could forge bids or claim to be a builder.

---

**`ExecutionPayloadEnvelope`**
- **Fields:** `payload`, `execution_requests`, `builder_index`, `beacon_block_root`, `parent_beacon_block_root`

**Why it exists:** After the beacon block is published, the builder reveals their payload in this envelope:
- `payload`: The actual execution payload (transactions, state changes)
- `beacon_block_root`: Links this envelope to the specific block that included the bid
- `parent_beacon_block_root`: Links the envelope back to the beacon parent root

Note: `blob_kzg_commitments` was moved from the envelope to the bid, so commitments are locked in at bid time rather than reveal time.

The envelope lets nodes verify the payload matches the bid commitment. State processing is deferred: the child block carries `parent_execution_requests` and calls `process_parent_execution_payload`.

---

**`ProposerPreferences`** and **`SignedProposerPreferences`**

**Why they exist:** Builders need to know proposer preferences BEFORE constructing bids:
- `fee_recipient`: Where the proposer wants to receive payment
- `target_gas_limit`: The proposer's preferred gas target for the block

Without this, builders would have to guess, and bids with wrong values would be rejected. Proposers may broadcast signed preferences for upcoming proposal slots in the proposer lookahead.

---

#### Modified Containers

**`BeaconBlockBody`**
- **Added:** `signed_execution_payload_bid`, `payload_attestations`, `parent_execution_requests`
- **Removed:** `execution_payload`, `blob_kzg_commitments`, `execution_requests`

**Why it changed:** This is THE fundamental change of ePBS. Pre-GLOAS, proposers included the full execution payload in their block. Post-GLOAS:
- Proposers include a BID (commitment) instead of the payload itself
- The actual payload comes later from the builder
- Proposers also include PTC votes from the previous slot
- Child blocks include `parent_execution_requests` when building on the parent's FULL payload

This separation is what enables trustless proposer-builder separation—the proposer commits to a bid without seeing the payload contents.

---

**`ExecutionPayload`**
- **Added/retained post-Deneb fields:** `blob_gas_used`, `excess_blob_gas`
- **Added in Gloas payload shape:** `block_access_list` (EIP-7928), `slot_number` (EIP-7843)

**Why it changed:** The payload is no longer in the beacon block body, but the envelope still carries the full execution payload. `slot_number` lets the payload bind to the beacon slot, and `block_access_list` carries the EIP-7928 payload data.

---

**`BeaconState`**
- **Added:** `builders`, `next_withdrawal_builder_index`, `latest_execution_payload_bid`, `execution_payload_availability`, `builder_pending_payments`, `builder_pending_withdrawals`, `latest_block_hash`, `payload_expected_withdrawals`, `ptc_window`
- **Removed:** `latest_execution_payload_header`

**Why it changed:** The state needs new fields for builder economics:
- `builders`: The separate builder registry (like `validators` but for builders)
- `builder_pending_payments`: Payments waiting for quorum
- `builder_pending_withdrawals`: Confirmed payments waiting to execute
- `execution_payload_availability`: Bitvector tracking which recent slots had payloads delivered
- `latest_block_hash`: Tracks the chain of execution blocks for parent hash validation
- `ptc_window`: Cached PTC committees for previous/current/lookahead epochs

The removed `latest_execution_payload_header` is replaced by the bid/envelope system.

---

**`ForkChoiceNode`**
- **Fields:** `root`, `payload_status`

**Why it exists/changed:** Pre-GLOAS, each block root mapped to one fork choice node. Post-GLOAS, a single block root can have MULTIPLE nodes with different payload statuses:
- (root, PENDING): Block seen, payload not yet resolved
- (root, EMPTY): Block seen, payload didn't arrive
- (root, FULL): Block seen, payload arrived and validated

Fork choice now picks the best `(root, status)` pair, not just the best root. This lets the chain continue even if some builders withhold payloads.

---

**`LatestMessage`**
- **Fields:** `slot`, `root`, `payload_present`
- **Changed:** Now tracks `slot` (was `epoch`) and adds `payload_present`

**Why it changed:** For LMD-GHOST fork choice, we track each validator's latest vote. GLOAS needs finer granularity:
- `slot` instead of `epoch`: Same-slot vs previous-slot attestations behave differently
- `payload_present`: The attester's view of payload availability affects which node their vote supports

---

**`Store`**
- **Added:** `payloads`, `payload_timeliness_vote`, `payload_data_availability_vote`

**Why it changed:** Fork choice needs new tracking:
- `payloads`: Maps beacon block roots to verified `ExecutionPayloadEnvelope` objects
- `payload_timeliness_vote`: Tracks PTC votes for payload presence/timeliness
- `payload_data_availability_vote`: Tracks PTC votes for blob data availability

---

**`DataColumnSidecar`**
- **Added:** `slot`, `beacon_block_root`
- **Removed:** `signed_block_header`, `kzg_commitments`, `kzg_commitments_inclusion_proof`

**Why it changed:** Pre-GLOAS, sidecars needed a signed block header and merkle proof because PROPOSERS distributed them—you needed to verify the proposer authorized this data.

Post-GLOAS, BUILDERS distribute sidecars. The KZG commitments are now stored in the bid (`bid.blob_kzg_commitments`), not in the sidecar. Verification is simpler:
1. Look up the bid in the beacon block
2. Use `bid.blob_kzg_commitments` directly for verification
3. Verify `slot` and `beacon_block_root` match

No proposer signature needed—the bid commitment handles authenticity. A new `verify_data_column_sidecar_kzg_proofs` function takes `kzg_commitments` as a parameter alongside the sidecar.

---

### 9.3 Functions

#### Predicates (New)

**`is_builder_index(index)`**

**Why it exists:** When processing withdrawals or other operations, the system encounters indices that could be either validators or builders. This function checks the `BUILDER_INDEX_FLAG` bit to determine which registry the index refers to. Without this, code would incorrectly look up builder indices in the validator registry (or vice versa).

---

**`is_active_builder(state, builder_index)`**

**Why it exists:** Before accepting a bid, the protocol must verify the builder is active (deposited, not exited). An exited builder shouldn't be able to submit new bids. This function checks the builder's lifecycle state.

---

**`is_builder_withdrawal_credential(withdrawal_credentials)`**

**Why it exists:** This helper only checks whether withdrawal credentials start with `0x03`. It does not route the deposit by itself. `process_deposit_request` applies the full routing rules: existing builder pubkeys top up builders, existing validator pubkeys stay validator-side, and new `0x03` deposits create builders only when no pending validator deposit blocks that pubkey.

---

**`is_attestation_same_slot(state, data)`**

**Why it exists:** Builder payments only count attestations from the SAME slot as the block. Why? Same-slot attestations prove the attester saw the block quickly, indicating network health. If the network is partitioned or the block was delayed, same-slot attestations won't happen, and the payment shouldn't execute. This function identifies which attestations count toward quorum.

---

**`is_valid_indexed_payload_attestation(state, indexed_payload_attestation)`**

**Why it exists:** Before accepting a payload attestation, we must verify its basic validity:
- Are the indices non-empty?
- Are the indices sorted? (prevents double-counting in aggregation)
- Is the aggregate BLS signature valid?

Note: This function does NOT verify PTC membership—it only checks structural validity and signature. PTC membership verification happens separately (e.g., in `on_payload_attestation_message` in fork-choice).

---

**Parent-full check in `process_withdrawals(state)`**

**Why it exists:** Some protocol rules depend on whether the parent block's payload was delivered. The latest spec checks this by comparing:
```
state.latest_block_hash == state.latest_execution_payload_bid.block_hash
```
If the bid's committed `block_hash` equals the chain's `latest_block_hash`, the parent payload was processed as FULL. This check must happen before the current block's bid overwrites `latest_execution_payload_bid`.

---

#### Helper Functions (New)

**`can_builder_cover_bid(state, builder_index, bid_value)`**

**Why it exists:** Before accepting a bid into a block, we must verify the builder can actually pay. This function checks:
```
pending_balance = get_pending_balance_to_withdraw_for_builder(...)
min_balance = MIN_DEPOSIT_AMOUNT + pending_balance
available = builder_balance - min_balance
return available >= bid_amount
```
A builder can't bid more than they can afford, preventing default scenarios.

---

**`compute_balance_weighted_selection(state, indices, seed, size, shuffle)`**

**Why it exists:** PTC selection must be sybil-resistant—validators with more stake should be proportionally more likely to be selected. Simple random selection would let someone with 1000 validators dominate the PTC. Balance-weighted selection ensures a validator with 64 ETH is twice as likely to be selected as one with 32 ETH. This is the core algorithm powering fair PTC composition.

---

**`get_ptc(state, slot)`**

**Why it exists:** Given a slot, which 512 PTC positions form the Payload Timeliness Committee? This function computes the PTC by applying balance-weighted selection to that slot's attestation committees. The result is deterministic, can contain duplicate validators, and lets all nodes agree on committee positions.

---

**`get_builder_payment_quorum_threshold(state)`**

**Why it exists:** How much attestation weight is needed to confirm a payment? This function calculates the 60% threshold in Gwei:
```
(total_active_balance / SLOTS_PER_EPOCH) * 6 / 10
```
The result is compared against accumulated `weight` in pending payments to determine if quorum is reached.

---

#### State Transition Functions (New)

**`process_slot(state)`** — MODIFIED

**Why it changed:** Each slot, the protocol must clear the execution payload availability flag for the NEXT slot. This is done by setting `execution_payload_availability[(slot + 1) % SLOTS_PER_HISTORICAL_ROOT] = 0`.

**The reason:** The availability bitvector tracks which recent slots had their payloads delivered. Before a new slot begins, we reset its flag to 0 (not available). If a later child block applies that slot's parent payload as FULL, the flag is set to 1. This allows the protocol to track payload delivery history for recent slots.

---

**`process_epoch(state)`** — MODIFIED

**Why it changed:** Added calls to `process_builder_pending_payments()` and `process_ptc_window()`.

**The reason:** Pending payments have a lifecycle:
1. Created when a bid is included in a block
2. Accumulate weight as same-slot attestations arrive
3. At epoch boundary: check if quorum reached → move to withdrawal queue OR expired → discard

Epoch processing is the natural place for this "end of period" accounting and for sliding the cached PTC lookahead window.

---

**`process_operations(state, body)`** — MODIFIED

**Why it changed:** Execution-layer requests are no longer processed directly from the beacon block body. Gloas moves those requests into the payload envelope and applies them from the child block through `parent_execution_requests`.

Key changes:
1. Asserts `len(body.deposits) == 0`
2. Removes direct `process_deposit_request`, `process_withdrawal_request`, and `process_consolidation_request` calls from block operations
3. Processes `body.payload_attestations` with `process_payload_attestation`
4. Leaves deposits/withdrawals/consolidations to `apply_parent_execution_payload` when the parent is FULL

---

**`process_execution_payload_bid(state, block)`**

**Why it exists:** When a beacon block arrives containing a builder's bid, this function:
1. Validates the bid (correct slot, correct parent, builder is active)
2. Verifies the builder's signature
3. Checks the builder can cover the bid value
4. Creates a `BuilderPendingPayment` entry
5. Updates `latest_execution_payload_bid` in state

This is the "commit" phase—the builder is now on the hook for this bid.

---

**`verify_execution_payload_envelope(state, signed_envelope, execution_engine)`**

**Why it exists:** When the payload envelope arrives (after the beacon block), this function:
1. Verifies the envelope matches the committed bid (`block_hash` must match)
2. Verifies the builder's signature
3. Sends the payload to the execution engine for validation
4. Verifies `execution_requests_root`, `payload.slot_number`, timestamp, parent hash, and expected withdrawals

This is the "reveal" phase—the builder proves they have a valid payload matching their commitment. It is pure verification; state updates happen later through `process_parent_execution_payload` in the child block.

---

**`process_parent_execution_payload(state, block)`**

**Why it exists:** When a child beacon block is processed, this function decides whether the parent was FULL or EMPTY:
1. If the current bid's `parent_block_hash` does not equal the previous bid's `block_hash`, the parent is EMPTY and `parent_execution_requests` must be empty.
2. If it matches, the parent is FULL, so the child block must carry execution requests whose hash equals the previous bid's `execution_requests_root`.
3. It applies the parent execution requests, settles the parent builder payment, marks payload availability, and updates `latest_block_hash`.

---

**`process_payload_attestation(state, payload_attestation)`**

**Why it exists:** When processing a block, we encounter PTC votes from the PREVIOUS slot. This function validates:
1. The attestation references the parent beacon block (`data.beacon_block_root == state.latest_block_header.parent_root`)
2. The attestation is for the previous slot (`data.slot + 1 == state.slot`)
3. The aggregate signature is valid (via `is_valid_indexed_payload_attestation`)

Note: This function only validates PTC attestations for inclusion—it does NOT update payment weights. Payment weight accumulation happens separately in `process_attestation` for same-slot regular attestations.

---

**`process_attestation(state, attestation)`** — MODIFIED

**Why it changed:** Same-slot attestations now serve double duty:
- Normal duty: Signal which block the attester considers head
- New duty: Contribute to builder payment quorum

The modification adds: if this attestation is for the current slot, add the attesters' effective balance to the pending payment's `weight` field.

---

**`get_expected_withdrawals(state)`** — MODIFIED

**Why it changed:** Withdrawals now include multiple new categories and the return type is the new `ExpectedWithdrawals` dataclass (instead of a tuple). The function builds the withdrawal list in this order:
1. **Builder withdrawals** (`get_builder_withdrawals`): Payments from builders to proposers that reached quorum
2. **Partial withdrawals** (`get_pending_partial_withdrawals`): Validator partial balance withdrawals (existing)
3. **Builder sweep withdrawals** (`get_builders_sweep_withdrawals`): Builder balance withdrawals when exiting
4. **Validator sweep withdrawals** (`get_validators_sweep_withdrawals`): Regular validator sweep (existing, at least one space reserved)

This ordering ensures builder payments are processed first, then existing withdrawal types, then builder exits.

---

**`process_proposer_slashing(state, proposer_slashing)`** — MODIFIED

**Why it changed:** If a proposer is slashed, any pending builder payments TO that proposer should be cancelled. Why? The proposer committed a slashable offense—they shouldn't profit from a block they proposed maliciously. The modification: forfeit the builder's pending payment (builder keeps their stake, but proposer doesn't get paid).

---

#### Fork Choice Functions

**`notify_forkchoice_updated` inputs** — MODIFIED

**Why it changed:** In Gloas, a confirmed beacon block's own payload may still only be a bid commitment. The execution hashes sent to the EL therefore use parent payload hashes:

```python
finalized_block_hash = finalized_block_bid.parent_block_hash
safe_block_hash = get_safe_execution_block_hash(fcr_store)
```

All other `notify_forkchoice_updated` semantics are inherited from prior forks.

---

**`on_block(store, signed_block)`** — MODIFIED

**Why it changed:** Blocks no longer contain execution payloads—they contain bids. The handler now:
1. Processes the bid commitment (not a full payload)
2. Creates a node with `PAYLOAD_STATUS_PENDING` (waiting for payload to arrive)
3. Processes any payload attestations in the block body

The block is initially pending until a valid envelope is stored by `on_execution_payload_envelope`.

---

**`on_execution_payload_envelope(store, signed_envelope)`** — NEW

**Why it exists:** This is the handler for when a payload envelope arrives (separate from the beacon block). It:
1. Finds the beacon block this payload belongs to (via `beacon_block_root`)
2. Verifies the payload matches the bid commitment
3. Verifies blob data availability and execution validity
4. Stores the verified envelope in `store.payloads`
5. Makes the FULL fork-choice child available; actual state processing waits for the child block

This handler bridges the gap between bid commitment and payload reveal.

---

**`on_payload_attestation_message(store, ptc_message, is_from_block)`** — NEW

**Why it exists:** When PTC votes arrive (via gossip or in blocks), this handler:
1. Verifies the vote is from a valid PTC member
2. Updates `store.payload_timeliness_vote[block_root]`
3. Updates `store.payload_data_availability_vote[block_root]`

The `is_from_block` flag distinguishes gossip (must be current slot) from block inclusion (can be previous slot).

---

**`get_head(store)`** — MODIFIED

**Why it changed:** Fork choice now returns a `ForkChoiceNode` (root + payload_status), not just a root. The algorithm:
1. Start from justified checkpoint
2. At each level, pick the child with highest weight
3. Consider (root, EMPTY) and (root, FULL) as different nodes
4. Use `get_payload_status_tiebreaker` if weights are equal

Note: The tiebreaker logic is nuanced—FULL doesn't always beat EMPTY. For previous-slot payloads, it depends on `should_extend_payload`.

---

**`is_supporting_vote(store, node, message)`** — EXPLAINER-ONLY

**Why it appears here:** This is a useful explanatory concept, not a normative helper in `fork-choice.md`. It answers: does an attestation support a particular fork choice node? It's not just about ancestry anymore; payload status compatibility also matters:
- If attester voted `payload_present=true`, their vote supports FULL nodes
- If attester voted `payload_present=false`, their vote supports EMPTY nodes

The normative fork-choice spec folds this logic into weight/scoring behavior rather than defining an `is_supporting_vote` function.

---

**`should_extend_payload(store, root)`** — NEW

**Why it exists:** Decides whether to extend a previous-slot block's FULL version. Returns true if ANY of:
1. Payload is locally verified and PTC supports both payload timeliness and blob data availability
2. `proposer_boost_root == Root()` — No proposer boost is currently set
3. `store.blocks[proposer_boost_root].parent_root != root` — Proposer boost conflicts with this root
4. `is_parent_node_full(store, store.blocks[proposer_boost_root])` — Proposer boost block's parent has FULL status

This determines whether fork choice should favor FULL over EMPTY for a given block.

---

**`get_payload_status_tiebreaker(store, node)`** — NEW

**Why it exists:** When two nodes have equal attestation weight, this provides a tiebreaker value. The logic is nuanced:

- **If PENDING**, or **block is NOT from previous slot**: Return raw status (EMPTY=0, FULL=1, PENDING=2)
- **If block IS from previous slot**:
  - EMPTY → returns 1
  - FULL → returns 2 if `should_extend_payload(root)` is true, else **0**

Key insight: FULL doesn't always win! If `should_extend_payload` returns false, FULL gets 0 (lowest priority), making EMPTY preferred. This prevents the chain from extending payloads that weren't timely or conflict with proposer boost.

---

**`get_attestation_score(store, node, state)`** — MODIFIED

**Why it changed:** This function computes the total effective balance of validators whose latest message supports the given `ForkChoiceNode`. It now takes a `ForkChoiceNode` instead of a plain root, since votes must match both the block root and payload status.

---

**Timing Functions** — MODIFIED

Functions like `get_attestation_due_ms`, `get_aggregate_due_ms`, etc. are all modified to implement the new GLOAS slot timeline:
- Attestations: 25% (was 33%) — earlier to leave room for PTC
- Aggregates: 50% (was 66%) — proportionally earlier
- PTC votes: 75% — new deadline for payload attestations

The shift gives builders and PTC more time while maintaining the overall slot structure.

#### Validator Functions

**`get_ptc_assignment(state, epoch, validator_index)`** — NEW

**Why it exists:** Validators need to know if they're on PTC duty for any slot. At epoch start, each validator calls this to check: "Am I on the PTC for any slot next epoch?" If yes, they must prepare to submit a payload attestation at 75% of that slot. Without this function, validators wouldn't know their PTC responsibilities.

---

**`get_upcoming_proposal_slots(state, validator_index)`** — NEW

**Why it exists:** Proposers may broadcast their preferences (`fee_recipient`, `target_gas_limit`) BEFORE builders construct bids. This function tells proposers which slots they'll propose in the lookahead, so they know when to broadcast `SignedProposerPreferences`.

---

#### Builder Functions

**`get_execution_payload_bid_signature(state, bid, privkey)`** — NEW

**Why it exists:** Builders must sign their bids to prove authenticity. This function creates the BLS signature using `DOMAIN_BEACON_BUILDER` and the builder's private key. Without valid signatures, bids would be rejected by the network.

---

**`get_data_column_sidecars(beacon_block_root, slot, cells_and_kzg_proofs)`** — MODIFIED

**Why it exists:** In GLOAS, BUILDERS (not proposers) distribute blob data. After revealing a payload, the builder must also distribute DataColumnSidecars for data availability sampling. This function generates the sidecars from the blob data, ready for gossip.

---

**`get_execution_payload_envelope_signature(state, envelope, privkey)`** — NEW

**Why it exists:** When revealing a payload, the builder must sign the envelope to prove they authored it. This signature links the revealed payload to the builder who committed via the bid. Anyone can verify the signature matches the `builder_index` in the envelope.

---

#### Fork Upgrade

**`upgrade_to_gloas(pre)`** — NEW

**Why it exists:** At `GLOAS_FORK_EPOCH`, the beacon state must transition from Fulu format to GLOAS format. This function:
1. Copies all existing state fields
2. Initializes new fields with appropriate defaults:
   - `builders`: Empty list (no builders registered yet)
   - `builder_pending_payments`: Pre-allocated array of `2 * SLOTS_PER_EPOCH` empty entries (covers current + previous epoch)
   - `execution_payload_availability`: All 1s (`0b1` for each slot)—assumes all pre-fork payloads were delivered
   - `latest_block_hash`: Set to `pre.latest_execution_payload_header.block_hash` (continuity from pre-fork)
   - `latest_execution_payload_bid`: Initialized from the pre-fork execution header with empty `ExecutionRequests()` root
   - `payload_expected_withdrawals`: Empty list
   - `ptc_window`: Initialized from the cached PTC lookahead
3. Removes `latest_execution_payload_header` (replaced by bid/envelope system)
4. Calls `onboard_builders_from_pending_deposits()` to process pending builder deposits that can be routed to the builder registry

**`onboard_builders_from_pending_deposits()`** processes pending deposits at the fork boundary using the same routing rules: existing builder pubkeys top up builders, existing validator pubkeys stay validator-side, and new 0x03 builder credentials only create builders when not blocked by pending validator-deposit state.

This is the one-time migration that enables ePBS.

---

### 9.4 Gossip Topics

#### New Topics

**`execution_payload_bid`** — Message: `SignedExecutionPayloadBid`

**Why it exists:** Builders need a way to advertise their bids to proposers. This topic creates a public marketplace where:
- Builders broadcast bids for upcoming slots
- Proposers listen and select the highest-value valid bid
- Other nodes can validate bids for fork choice purposes

**Important:** Builders should wait for `SignedProposerPreferences` before bidding—bids with wrong `fee_recipient` or incompatible `gas_limit` will be rejected.

---

**`execution_payload`** — Message: `SignedExecutionPayloadEnvelope`

**Why it exists:** After a timely beacon block is published with a bid, the winning builder is expected to reveal their payload. Honest builders may withhold if the referenced block was not timely and is not their head. This topic is where:
- The builder broadcasts the actual execution payload
- All nodes receive it to complete the block
- Fork choice updates from PENDING to FULL status

Timing is critical—builders should reveal quickly so PTC can vote "present."

---

**`payload_attestation_message`** — Message: `PayloadAttestationMessage`

**Why it exists:** PTC members need to broadcast their observations about payload timeliness and blob data availability. This topic carries:
- Individual votes on whether the payload arrived before `get_payload_due_ms()`
- Votes on whether blob data is available
- Data that will be aggregated for block inclusion

Aggregators collect these messages and create `PayloadAttestation` aggregates for the next block.

---

**`proposer_preferences`** — Message: `SignedProposerPreferences`

**Why it exists:** Before builders can create valid bids, they need to know what the proposer wants:
- `fee_recipient`: Where to send payment
- `target_gas_limit`: Preferred gas target for the payload

Proposers may broadcast preferences for upcoming proposal slots in the proposer lookahead. Without this, builders would have to guess (and likely be rejected).

---

#### Modified Topics

**`beacon_block`** — MODIFIED

**Why it changed:** GLOAS blocks no longer contain execution payloads—they contain bids. The topic type changes to the modified `SignedBeaconBlock`; p2p removes the old in-block `ExecutionPayload` checks and replaces them with bid-specific checks:
- `len(bid.blob_kzg_commitments)` is within the blob schedule limit
- parent execution payload referenced by `bid.parent_block_hash` has been seen, or the block may be queued
- if parent payload verification is complete, that parent passes validation
- `bid.parent_block_root == block.parent_root`

Full bid signature, active-builder, and balance checks are handled by block/state-transition validation, not as extra beacon-block gossip rules.

The block is smaller and arrives faster; the payload follows separately.

---

**`beacon_attestation_{subnet_id}`** and **`beacon_aggregate_and_proof`** — MODIFIED

**Why they changed:** Attestations now signal payload status via the `index` field:
- `index = 0`: Attester saw EMPTY block (or same-slot attestation)
- `index = 1`: Attester saw FULL block

Validation must check that the attested `(root, index)` pair is valid in fork choice. You can't attest to FULL status for a block whose payload never arrived.

---

**`data_column_sidecar_{subnet_id}`** — MODIFIED

**Why it changed:** Sidecars now come from BUILDERS, not proposers. Validation uses:
- `beacon_block_root` + `slot` to identify the block
- KZG commitments from `bid.blob_kzg_commitments` for verification

No proposer signature needed—the bid commitment provides authenticity. Clients SHOULD queue sidecars for deferred validation if the block hasn't been seen yet, and MUST re-broadcast after successful deferred validation.

---

### 9.5 Req/Resp Methods

#### New Methods

**`ExecutionPayloadEnvelopesByRange`** v1

**Why it exists:** During sync, nodes need to fetch execution payloads for historical slots. Since payloads are no longer embedded in blocks, a new method is needed. Request a range of slots; receive the corresponding payload envelopes.

**Use case:** Node syncing from genesis needs both blocks AND payloads. It fetches blocks via `BeaconBlocksByRange`, then payloads via this method.

---

**`ExecutionPayloadEnvelopesByRoot`** v1

**Why it exists:** Sometimes you need specific payloads by block root:
- You received a block but missed the payload on gossip
- PTC voted "present" but you never saw the payload
- You're reconstructing state and need the execution data

Request roots; receive matching envelopes. More targeted than by-range.

Clients MUST support recent root requests over:
`[max(GLOAS_FORK_EPOCH, current_epoch - compute_min_epochs_for_block_requests()), current_epoch]`.
For older roots, peers may return `ResourceUnavailable` or omit the envelope.

---

#### Updated Methods

**`BeaconBlocksByRange`** v2 and **`BeaconBlocksByRoot`** v2 — MODIFIED

**Why they changed:** Post-GLOAS blocks have a different structure (bid instead of payload). These methods now:
- Return GLOAS-format blocks for slots after `GLOAS_FORK_EPOCH`
- Return pre-GLOAS format for earlier slots
- Clients must separately fetch payloads via envelope methods

This maintains backwards compatibility while supporting the new block structure.

---

### 9.6 Companion Fork Material

**`ExecutionPayload.block_access_list: BlockAccessList`** — EIP-7928

**Why it exists:** Gloas inherits the updated payload shape that includes
`block_access_list`. The field lives inside `ExecutionPayload`, not the bid or
top-level envelope. The bid commits to it indirectly through `block_hash`.

---

**`ExecutionPayload.slot_number` / SLOTNUM** — EIP-7843

**Why it exists:** The execution payload carries the beacon slot number so the
execution layer can expose SLOTNUM semantics and the consensus layer can verify
`payload.slot_number == block.slot` during envelope validation.

---

**Slashed-validator proposer exclusion** — EIP-8045

**Why it exists:** `get_beacon_proposer_indices` filters active validators by
`not state.validators[index].slashed` before proposer selection. Slashed
validators can still exist in the active set for other lifecycle purposes, but
Gloas excludes them from newly computed proposer duties.

---

**Churn changes** — EIP-8061

**New/modified constants:**
- `CHURN_LIMIT_QUOTIENT_GLOAS`
- `CONSOLIDATION_CHURN_LIMIT_QUOTIENT`
- `MAX_PER_EPOCH_ACTIVATION_CHURN_LIMIT_GLOAS`

**New/modified helpers:**
- `get_activation_churn_limit(state)`: capped activation-only churn
- `get_exit_churn_limit(state)`: uncapped exit churn
- `get_consolidation_churn_limit(state)`: independent consolidation churn
- `compute_exit_epoch_and_update_churn(state, exit_balance)`: uses exit churn

**Why it exists:** Gloas separates activation, exit, and consolidation churn so
one flow does not consume the whole shared churn budget. Deposits still consume
activation-only churn, exits use exit churn, and consolidations use their own
limit.

---

**Weak subjectivity period** — EIP-8061

**Why it changed:** `compute_weak_subjectivity_period` now accounts for the split
churn model:

```python
delta = (
    2 * get_exit_churn_limit(state) // 3
    + get_activation_churn_limit(state) // 3
    + get_consolidation_churn_limit(state)
)
```

The weak subjectivity period uses this `delta` because validator-set churn is no
longer governed by one shared churn limit.

---

## 10. FAQ

### Q1: What is the difference between MIN_DEPOSIT_AMOUNT and MIN_ACTIVATION_BALANCE?

These are two different thresholds in the validator lifecycle:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│           MIN_DEPOSIT_AMOUNT vs MIN_ACTIVATION_BALANCE                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  MIN_DEPOSIT_AMOUNT = 1 ETH (1 * 10^9 Gwei)                                 │
│  ══════════════════════════════════════════                                 │
│                                                                             │
│  • Minimum amount for a SINGLE deposit transaction                          │
│  • You can make multiple deposits to the same validator                     │
│  • Each individual deposit must be at least 1 ETH                           │
│  • Prevents spam/dust deposits                                              │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────── │
│                                                                             │
│  MIN_ACTIVATION_BALANCE = 32 ETH (32 * 10^9 Gwei)                           │
│  ════════════════════════════════════════════════                           │
│                                                                             │
│  • Minimum TOTAL balance needed to become an active validator               │
│  • Must accumulate this much before entering activation queue               │
│  • The "full stake" required to participate in consensus                    │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────── │
│                                                                             │
│  EXAMPLE FLOW:                                                              │
│                                                                             │
│    Deposit #1:  8 ETH  ✓ (≥ MIN_DEPOSIT_AMOUNT)                             │
│    Deposit #2: 10 ETH  ✓                                                    │
│    Deposit #3: 14 ETH  ✓                                                    │
│                ───────                                                      │
│    Total:      32 ETH  → Now eligible for activation!                       │
│                          (≥ MIN_ACTIVATION_BALANCE)                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**TL;DR:**
- `MIN_DEPOSIT_AMOUNT` (1 ETH) = minimum per deposit transaction
- `MIN_ACTIVATION_BALANCE` (32 ETH) = minimum total balance to activate

---

### Q2: How much ETH do builders need?

Builders have **no minimum activation balance** (unlike validators which need 32 ETH). However, they must keep a reserve of `MIN_DEPOSIT_AMOUNT` (1 ETH). They can only bid with balance **above** the reserve plus any pending obligations.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Builder Balance Structure                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Builder with 50 ETH balance:                                               │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │   50 ETH total                                                      │   │
│  │   ┌─────────────────────────────────────────────────────────────┐  │   │
│  │   │▓▓│░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│  │   │
│  │   │1E│                    49 ETH                                │  │   │
│  │   │  │              (AVAILABLE FOR BIDS)                        │  │   │
│  │   │  │  MIN_DEPOSIT_AMOUNT reserve (1 ETH)                     │  │   │
│  │   └─────────────────────────────────────────────────────────────┘  │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  The builder can bid UP TO 49 ETH (if no pending obligations)               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### Q3: What are pending_payments and pending_withdrawals in the bid validation?

From `process_execution_payload_bid`:
```python
assert (
    can_builder_cover_bid(state, builder_index, amount)
)
```

These represent **money already committed** from previous bids. You can't spend the same ETH twice!

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONCRETE EXAMPLE                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SETUP: Builder "Bob" has 50 ETH balance                                    │
│                                                                             │
│  ═══════════════════════════════════════════════════════════════════════   │
│  SLOT 100: Bob's bid of 5 ETH is included in a block                        │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                             │
│    builder_pending_payments[slot 100] = {                                   │
│        weight: 0,          # Will accumulate from attestations              │
│        withdrawal: {                                                        │
│            amount: 5 ETH,  # ◄── This is "pending_payments"                 │
│            builder_index: Bob                                               │
│        }                                                                    │
│    }                                                                        │
│                                                                             │
│    Bob's balance is still 50 ETH (not deducted yet!)                        │
│    But 5 ETH is "earmarked" for potential payment                           │
│                                                                             │
│  ═══════════════════════════════════════════════════════════════════════   │
│  SLOT 101: Bob wants to bid again. How much can he bid?                     │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                             │
│    Check: can_builder_cover_bid(state, Bob, X)                              │
│    pending_balance = pending_payments + pending_withdrawals = 5 ETH         │
│    min_balance = MIN_DEPOSIT_AMOUNT + pending_balance                       │
│             = 1 ETH + 5 ETH = 6 ETH                                        │
│    available = 50 - 6 = 44 ETH                                             │
│    X <= 44 ETH  ◄── Max new bid!                                            │
│                                                                             │
│    Bob can bid up to 44 ETH on slot 101                                     │
│                                                                             │
│  ═══════════════════════════════════════════════════════════════════════   │
│  SLOT 101: Bob bids 10 ETH, gets included                                   │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                             │
│    Now pending_payments = 5 + 10 = 15 ETH                                   │
│                                                                             │
│  ═══════════════════════════════════════════════════════════════════════   │
│  EPOCH BOUNDARY: Slot 100's payment reaches quorum!                         │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                             │
│    • Slot 100's 5 ETH moves from pending_payments → pending_withdrawals     │
│    • Now:                                                                   │
│        pending_payments = 10 ETH (slot 101 still pending)                   │
│        pending_withdrawals = 5 ETH (confirmed, waiting to withdraw)         │
│                                                                             │
│  ═══════════════════════════════════════════════════════════════════════   │
│  SLOT 150: Bob wants to bid again. How much?                                │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                             │
│    pending_balance = 10 (pending_payments) + 5 (pending_withdrawals)        │
│    min_balance = 1 + 15 = 16 ETH                                           │
│    available = 50 - 16 = 34 ETH                                            │
│    X <= 34 ETH  ◄── Max new bid!                                            │
│                                                                             │
│  ═══════════════════════════════════════════════════════════════════════   │
│  LATER: The 5 ETH pending_withdrawal gets processed                         │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                             │
│    • 5 ETH is actually deducted from Bob's balance                          │
│    • 5 ETH is sent to the proposer's fee_recipient                          │
│    • Bob's balance: 50 - 5 = 45 ETH                                         │
│    • pending_withdrawals: 0 ETH                                             │
│                                                                             │
│    Now Bob can bid: 45 - 1 - 10 = 34 ETH (same, balance dropped too)        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Payment Lifecycle Visual

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PAYMENT LIFECYCLE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   BID INCLUDED          EPOCH BOUNDARY           WITHDRAWAL PROCESSED       │
│   IN BLOCK              (quorum check)           (actual ETH transfer)      │
│       │                      │                          │                   │
│       ▼                      ▼                          ▼                   │
│  ┌─────────┐           ┌─────────┐              ┌─────────────────┐        │
│  │ builder │  ──────►  │ builder │   ──────►    │ Actual balance  │        │
│  │ pending │  quorum   │ pending │   withdraw   │ decreased,      │        │
│  │ payments│  reached  │withdrawals  epoch     │ ETH sent to     │        │
│  │         │           │         │   reached    │ proposer        │        │
│  └─────────┘           └─────────┘              └─────────────────┘        │
│                                                                             │
│  "I might have     "I definitely       "Money actually                      │
│   to pay this"      owe this"           left my account"                    │
│                                                                             │
│  Pending obligations plus reduced balance constrain future bids!            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**TL;DR:**
- `pending_payments` = bids waiting for quorum confirmation (might pay)
- `pending_withdrawals` = confirmed payments waiting to be withdrawn (will pay)
- Both are "reserved" and can't be used for new bids

---

### Q4: What happened to attestation.data.index for committee index?

In GLOAS, `attestation.data.index` is **repurposed** to signal payload status (0 = empty, 1 = full).

The committee index moved to `attestation.committee_bits` starting in Electra (EIP-7549).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Evolution of Committee Identification                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PRE-ELECTRA (Phase0 through Deneb):                                        │
│  ═══════════════════════════════════                                        │
│                                                                             │
│    AttestationData {                                                        │
│        slot                                                                 │
│        index  ◄── Committee index (0, 1, 2, ... N-1)                        │
│        beacon_block_root                                                    │
│        source                                                               │
│        target                                                               │
│    }                                                                        │
│                                                                             │
│    One attestation = one committee                                          │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────── │
│                                                                             │
│  ELECTRA (EIP-7549):                                                        │
│  ═══════════════════                                                        │
│                                                                             │
│    Attestation {                                                            │
│        aggregation_bits                                                     │
│        data: AttestationData                                                │
│        committee_bits  ◄── NEW! Bitvector indicating which committees       │
│        signature                                                            │
│    }                                                                        │
│                                                                             │
│    AttestationData.index = 0 (always, became unused)                        │
│    Now attestations can aggregate ACROSS multiple committees!               │
│                                                                             │
│  ───────────────────────────────────────────────────────────────────────── │
│                                                                             │
│  GLOAS (EIP-7732):                                                          │
│  ═════════════════                                                          │
│                                                                             │
│    AttestationData.index REPURPOSED:                                        │
│        index = 0  →  "Payload EMPTY / not seen"                             │
│        index = 1  →  "Payload FULL / seen"                                  │
│                                                                             │
│    Committee info still in attestation.committee_bits                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**TL;DR:** Electra moved committee identification to `committee_bits`, freeing `data.index` for GLOAS to repurpose as payload status signal.

---

### Q5: Why is MAX_PAYLOAD_ATTESTATIONS = 4?

`MAX_PAYLOAD_ATTESTATIONS` is the maximum number of **aggregated `PayloadAttestation` objects** that can be included in a single beacon block.

PTC members can vote with **different data**, so multiple aggregates may be needed:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PayloadAttestation Combinations                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PayloadAttestationData has two boolean fields:                             │
│  • payload_present: true/false                                              │
│  • blob_data_available: true/false                                          │
│                                                                             │
│  The two booleans produce 2² = 4 possible combinations:                     │
│                                                                             │
│  ┌──────────────────┬─────────────────────┬─────────────────────────────┐  │
│  │ payload_present  │ blob_data_available │ Valid?                      │  │
│  ├──────────────────┼─────────────────────┼─────────────────────────────┤  │
│  │ true             │ true                │ ✓ Builder revealed all      │  │
│  │ true             │ false               │ ✓ Payload seen, blobs lost  │  │
│  │ false            │ false               │ ✓ Builder withheld all      │  │
│  │ false            │ true                │ ✓ Blobs available, payload  │  │
│  │                  │                     │   not timely/seen           │  │
│  └──────────────────┴─────────────────────┴─────────────────────────────┘  │
│                                                                             │
│  So MAX = 4 covers the full boolean state space.                            │
│                                                                             │
│  IMPORTANT: All payload attestations in a block must reference the SAME     │
│  beacon_block_root (the parent block). From process_payload_attestation:    │
│                                                                             │
│    assert data.beacon_block_root == state.latest_block_header.parent_root   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### Q6: Why is `state_root` not a top-level envelope or bid field anymore?

`state_root` still exists inside `ExecutionPayload`. It is not duplicated at the
top level of `ExecutionPayloadEnvelope` or `ExecutionPayloadBid` because the bid
commits to `block_hash`, and the execution block hash commits to the execution
header, including `state_root`.

The top-level envelope fields are for linking the payload to the beacon block and
builder: `payload`, `execution_requests`, `builder_index`, `beacon_block_root`,
and `parent_beacon_block_root`.

---

### Q7: Why does the bid commit to `block_hash`, `blob_kzg_commitments`, and `execution_requests_root` instead of carrying the whole payload?

The bid is the small commitment that can go inside the beacon block. It commits
to the important pieces without revealing the payload:

- `block_hash`: commits to the execution payload contents, including the
  execution header and transaction list root.
- `blob_kzg_commitments`: commits to blob data availability at bid time.
- `execution_requests_root`: commits to the EL-triggered requests that the child
  beacon block will later process through `parent_execution_requests`.

The full payload stays in `SignedExecutionPayloadEnvelope`, which the builder
broadcasts after seeing the beacon block.

---

### Q8: Why are execution requests processed by the child block through `parent_execution_requests`?

The parent beacon block is processed before the builder reveals its payload, so
the parent cannot safely process execution requests from that payload during its
own state transition.

Instead, the child block decides whether it is building on the parent's FULL or
EMPTY variant:

- EMPTY parent: `parent_execution_requests` must be empty.
- FULL parent: `hash_tree_root(parent_execution_requests)` must match the
  previous bid's `execution_requests_root`, then the requests are applied.

This keeps gossip-time payload verification separate from deterministic beacon
state transition.

---

### Q9: Why can a beacon block have EMPTY and FULL fork-choice variants?

In Gloas, the beacon block and execution payload arrive separately. The beacon
block can be valid even when the payload is missing, late, or not considered
available by fork choice.

Fork choice therefore tracks `(root, payload_status)` instead of only `root`:

- `(root, PENDING)`: block seen, payload decision not resolved yet.
- `(root, EMPTY)`: chain continues without applying that block's payload.
- `(root, FULL)`: chain builds on the revealed and verified payload.

This lets the chain keep moving without trusting the builder to reveal on time.

---

### Q10: Why does `should_build_on_full` sometimes build on the empty parent even if a payload exists locally?

Local availability is not enough. For previous-slot heads, fork choice also uses
the PTC view of payload timeliness and blob data availability.

`should_build_on_full` returns false when the head is EMPTY, or when the PTC view
says the payload was not timely or blob data was not available. This prevents a
single node from extending a payload that it saw locally but the committee judged
late or unavailable.

---

### Q11: Why does builder payment quorum use same-slot regular attestations instead of PTC votes?

The payment quorum answers a different question from the PTC.

- Same-slot regular attestations show that validators saw and voted for the
  beacon block containing the bid. Their effective balances accumulate into the
  builder payment's `weight`.
- PTC votes happen later and judge payload timeliness plus blob data
  availability.

Payment is about whether the accepted bid should be honored. PTC is about which
payload variant fork choice should extend.

---

### Q12: Why are builder withdrawals encoded using `BUILDER_INDEX_FLAG` in `Withdrawal.validator_index`?

Withdrawals already have a `validator_index` field. Gloas reuses that container
for builder withdrawals by setting `BUILDER_INDEX_FLAG` in the high bit.

That gives a disjoint namespace:

- normal validator withdrawal: `validator_index` without the flag
- builder withdrawal: `builder_index | BUILDER_INDEX_FLAG`

Helpers convert both ways, so execution-layer withdrawal processing can carry
builder payments without adding a separate withdrawal container.

---

### Q13: Why are withdrawals deducted before the corresponding payload is confirmed, and what is `payload_expected_withdrawals` for?

Withdrawals are deterministic from the beacon state and must be committed before
the payload that uses them is confirmed. If the CL delayed deductions until the
child slot, epoch processing or other state changes could reduce balances before
the deduction, creating supply-accounting problems.

So Gloas applies the withdrawal deduction during beacon block processing and
stores the exact list in `state.payload_expected_withdrawals`. Later, envelope
verification checks:

```python
hash_tree_root(payload.withdrawals) == hash_tree_root(state.payload_expected_withdrawals)
```

This creates a temporary CL/EL asymmetry, but it makes the committed payload
withdrawals deterministic and supply-safe.

---

### Q14: Why did blob KZG commitments move into the bid, and why did `DataColumnSidecar` lose signed header/proof fields?

Blob KZG commitments moved into `ExecutionPayloadBid` so the builder commits to
blob data availability before the bid is selected. The payload envelope then has
to match the bid's commitments.

`DataColumnSidecar` no longer needs proposer signed-header and inclusion-proof
fields because validation is anchored by:

- `sidecar.beacon_block_root` and `sidecar.slot`
- the bid in the referenced beacon block
- `bid.blob_kzg_commitments`
- `sidecar.column` and `sidecar.kzg_proofs`

The builder, not the proposer, is responsible for broadcasting the sidecars.

---

### Q15: Why is the safe execution block hash the parent payload of the confirmed beacon block under Gloas?

A confirmed Gloas beacon block contains a bid for its own execution payload, but
that payload is not applied by that same block. It becomes part of the state only
when a later child block builds on the parent as FULL.

So the fast-confirmation safe execution hash is the confirmed block's parent
payload hash:

```python
confirmed.body.signed_execution_payload_bid.message.parent_block_hash
```

The confirmed block's own `block_hash` is still only a commitment until it is
processed through the FULL-parent path.

---

### Q16: Why are slashed validators excluded from proposer selection?

EIP-8045 changes `get_beacon_proposer_indices` so newly computed proposer duties
come only from active, unslashed validators.

Slashed validators remain in the validator set until they exit, but they have
already committed a slashable offense. Excluding them from proposer selection
prevents the protocol from assigning fresh block-production authority to actors
that are already known to be slashable.

---

### Q17: Why does Gloas change churn limits?

EIP-8061 separates activation, exit, and consolidation churn. Before this style
of split, different validator lifecycle flows can compete for one shared churn
budget.

Gloas uses:

- `get_activation_churn_limit`: capped activation churn for deposits
- `get_exit_churn_limit`: uncapped exit churn for exits
- `get_consolidation_churn_limit`: independent consolidation churn

This gives the protocol more predictable control over each queue instead of
letting one lifecycle operation starve the others.

---

### Q18: Why is `slot_number` / SLOTNUM added to the execution payload?

EIP-7843 adds `slot_number` to `ExecutionPayload` so the execution payload is
explicitly bound to the beacon slot. Gloas envelope validation checks:

```python
payload.slot_number == state.slot
```

That lets the execution layer expose SLOTNUM semantics and gives consensus a
direct payload-level slot check instead of relying only on timestamp-derived
inference.

---

### Q19: Why does Gloas change the weak subjectivity period?

EIP-8061 splits activation, exit, and consolidation churn into separately tuned
limits. Weak subjectivity period calculations depend on how quickly the validator
set can change, so the old single-churn formula is no longer accurate.

Gloas computes the churn term as:

```python
delta = 2/3 * exit_churn + 1/3 * activation_churn + consolidation_churn
```

That reflects the new lifecycle queues instead of pretending all churn flows use
one shared budget.

---

### Q20: Why does `notify_forkchoice_updated` use parent execution hashes for finalized/safe values?

In Gloas, a beacon block contains a bid for its own execution payload, but that
payload is not applied until a child builds on it as FULL. Therefore the
confirmed or finalized beacon block's own `block_hash` may still only be a
commitment.

The EL receives:

```python
finalized_block_hash = finalized_block_bid.parent_block_hash
safe_block_hash = get_safe_execution_block_hash(fcr_store)
```

These point to execution payloads that are actually safe/finalized under the
Gloas parent-payload processing model.

---

### Q21: Why are there two block-timeliness indexes?

Gloas tracks two distinct timeliness deadlines:

- `ATTESTATION_TIMELINESS_INDEX = 0`: regular beacon attestation timeliness
- `PTC_TIMELINESS_INDEX = 1`: payload-attestation/PTC timeliness

`NUM_BLOCK_TIMELINESS_DEADLINES = 2` sizes the per-block vector. Keeping these
separate lets fork choice account for normal attestation timing and payload
availability timing independently.

---

### Q22: What happens if proposer preferences are missing?

For trustless gossip bids, builders need a matching `SignedProposerPreferences`
for the proposal slot and dependent root. If preferences are missing, bid gossip
validation ignores the bid rather than accepting a guessed `fee_recipient` or
`target_gas_limit`.

Practically, a proposer that does not publish preferences is signaling that it
will not accept trustless public-gossip bids for that slot, though out-of-band or
self-build paths may still exist.

---

### Q23: Is `is_supporting_vote` a real spec helper or just an explanatory concept?

It is an explanatory concept in this README, not a normative helper currently
defined in `fork-choice.md`.

The concept is still useful: a vote must match both the beacon block root and the
payload status variant being scored. But implementers should follow the
normative fork-choice functions directly rather than looking for an
`is_supporting_vote` function in the spec.
