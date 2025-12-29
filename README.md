```text
GLOAS: Enshrined Proposer-Builder Separation (ePBS)

  A Complete Visual Guide

  ---
  1. THE BIG PICTURE: What Problem Does GLOAS Solve?

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
  │                         GLOAS SYSTEM (EIP-7732)                             │
  │                                                                             │
  │   ┌──────────────┐                                   ┌──────────────┐       │
  │   │   BUILDER    │ ─────── BID ────────────────────► │  PROPOSER    │       │
  │   │ (Validator   │    SignedExecutionPayloadBid      │ (Validator)  │       │
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
  │   │  • Builder pays only if same-slot attestations reach quorum      │      │
  │   │  • PTC (Payload Timeliness Committee) verifies payload delivery  │      │
  │   │  • No trusted third party needed!                                │      │
  │   └──────────────────────────────────────────────────────────────────┘      │
  │                                    │                                        │
  │   SPEC ENTITIES:                   ▼                                        │
  │   • BUILDER_WITHDRAWAL_PREFIX = 0x03           ┌─────────────────┐          │
  │   • has_builder_withdrawal_credential()        │   TRUSTLESS!    │          │
  │   • PTC_SIZE = 512 validators                  │  DECENTRALIZED  │          │
  │   • get_ptc(state, slot)                       │  CENSORSHIP     │          │
  │                                                │  RESISTANT      │          │
  │                                                └─────────────────┘          │
  └─────────────────────────────────────────────────────────────────────────────┘

  Key insight: The protocol itself becomes the escrow through cryptographic commitments and economic penalties.

  ---

⏺ 2. BEFORE vs AFTER: Structural Changes

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
      │  │  │  │  • blob_kzg_commitments               │  │  │  │
      │  │  │  └───────────────────────────────────────┘  │  │  │
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
      │  │  │ ║ PayloadBid      ║ │  │  │ref  │  │  blob_kzg_commitments     │  │
      │  │  │ ║ (commitment)    ║ │  │  │     │  │  execution_requests       │  │
      │  │  │ ╚═════════════════╝ │  │  │     │  │  beacon_block_root ◄──────┼──┤
      │  │  │ payload_attestations│  │  │     │  │  state_root               │  │
      │  │  └─────────────────────┘  │  │     │  └───────────────────────────┘  │
      │  └───────────────────────────┘  │     └─────────────────────────────────┘
      └─────────────────────────────────┘
             PROPOSER creates                      BUILDER creates
             (at slot start)                       (after block seen)

  Reasoning: This separation allows the builder to see the beacon block before revealing their payload, creating a commit-reveal scheme that's enforced by the protocol.

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
  │  SPEC: beacon-chain.md → BeaconState container                              │
  │  LIMIT: BUILDER_PENDING_WITHDRAWALS_LIMIT = 1,048,576                       │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  ---

⏺ 3. SLOT TIMELINE: What Happens When?

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
  │  PAYLOAD_ATTESTATION_DUE_BPS = 7500 (75% = 9s)  ← NEW! For PTC              │
  │                                                                             │
  │  FUNCTIONS (fork-choice.md): get_attestation_due_ms(epoch)                  │
  │                              get_payload_attestation_due_ms(epoch)          │
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
          │                     │              │ • 0 = empty │            │
          │                     │              │ • 1 = full  │            │
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
  │  SLOT N: Attestations Accumulate Weight                                     │
  │  ═══════════════════════════════════════                                    │
  │                                                                             │
  │  Same-slot attesters voting for the block add their effective_balance       │
  │  to the payment's "weight" field:                                           │
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
  │  EPOCH BOUNDARY: Quorum Check                                               │
  │  ════════════════════════════                                               │
  │                                                                             │
  │  At epoch processing (process_builder_pending_payments):                    │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐    │
  │  │                                                                     │    │
  │  │  quorum = (total_active_balance / SLOTS_PER_EPOCH) * 60%            │    │
  │  │                                                                     │    │
  │  │  if payment.weight >= quorum:                                       │    │
  │  │      → Move to builder_pending_withdrawals (CONFIRMED!)             │    │
  │  │      → Set withdrawable_epoch based on exit queue                   │    │
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
  │  that the block+payload were actually received and valid.                   │
  │                                                                             │
  │  AFTER withdrawable_epoch: Actual ETH Transfer                              │
  │  ═════════════════════════════════════════════                              │
  │                                                                             │
  │  When processing withdrawals, builder_pending_withdrawals are converted     │
  │  to actual Withdrawal objects that send ETH to the proposer's fee_recipient │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  ---

⏺ 4. NEW DATA STRUCTURES

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
  │  │    │ builder_index       │ ValidatorIndex - Who made bid        │  │   │
  │  │    ├─────────────────────┼──────────────────────────────────────┤  │   │
  │  │    │ slot                │ Slot - Which slot this is for        │  │   │
  │  │    ├─────────────────────┼──────────────────────────────────────┤  │   │
  │  │    │ value               │ Gwei - PAYMENT to proposer           │◄─┼───┤
  │  │    ├─────────────────────┼──────────────────────────────────────┤  │   │
  │  │    │ execution_payment   │ Gwei - Must be non-zero (see spec)   │  │   │
  │  │    ├─────────────────────┼──────────────────────────────────────┤  │   │
  │  │    │ blob_kzg_commitments│ Root - Hash of blob commitments      │  │   │
  │  │    │ _root               │                                      │  │   │
  │  │    └─────────────────────┴──────────────────────────────────────┘  │   │
  │  │  }                                                                  │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  KEY INSIGHT: The bid commits to:                                           │
  │  ═══════════════════════════════                                            │
  │  1. A specific block_hash (builder can't change payload after bid)          │
  │  2. A specific value (payment amount is locked in)                          │
  │  3. A specific parent (prevents bid reuse on different forks)               │
  │  4. Blob commitments root (ensures DA is also committed)                    │
  │                                                                             │
  │  Builder payment is only finalized if same-slot attestations reach quorum   │
  │                                                                             │
  │  SPEC: beacon-chain.md → ExecutionPayloadBid container                      │
  │        SignedExecutionPayloadBid wraps with BLS signature                   │
  │  FUNCTIONS: verify_execution_payload_bid_signature()                        │
  │             process_execution_payload_bid()                                 │
  │  GOSSIP: execution_payload_bid topic (p2p-interface.md)                     │
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
  │  │    └─────────────────────────────────────────────────────────────┘ │   │
  │  │                                                                     │   │
  │  │    execution_requests: ExecutionRequests                            │   │
  │  │      • deposits, withdrawals, consolidations (from EL)              │   │
  │  │                                                                     │   │
  │  │    builder_index: ValidatorIndex   ◄── Must match bid!              │   │
  │  │                                                                     │   │
  │  │    beacon_block_root: Root         ◄── Links to the beacon block    │   │
  │  │                                                                     │   │
  │  │    slot: Slot                      ◄── Must match block slot        │   │
  │  │                                                                     │   │
  │  │    blob_kzg_commitments: List[KZGCommitment]                        │   │
  │  │      ◄── hash_tree_root must match bid.blob_kzg_commitments_root    │   │
  │  │                                                                     │   │
  │  │    state_root: Root                ◄── Post-state after processing  │   │
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
  │                             │ blob_root    │◄─────────│ hash(comms)  │      │
  │                             └──────────────┘          └──────────────┘      │
  │                                                                             │
  │  SPEC: beacon-chain.md → ExecutionPayloadEnvelope container                 │
  │        SignedExecutionPayloadEnvelope wraps with BLS signature              │
  │  FUNCTIONS: verify_execution_payload_envelope_signature()                   │
  │             process_execution_payload()                                     │
  │  GOSSIP: execution_payload topic (p2p-interface.md)                         │
  │  HANDLER: on_execution_payload() (fork-choice.md)                           │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  4.3 Payload Attestation Structures (The PTC)

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                    PAYLOAD TIMELINESS COMMITTEE (PTC)                       │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  WHAT IS THE PTC?                                                           │
  │  ════════════════                                                           │
  │  A committee of 512 validators (PTC_SIZE = 2^9) selected per slot           │
  │  to attest whether the execution payload was delivered on time.             │
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
  │  │   │ (64 vals)   │   │         │  Selects 512 validators  │          │  │
  │  │   ├─────────────┤   │         │  weighted by stake       │          │  │
  │  │   │    ...      │ ──┘         │  (higher stake = more    │          │  │
  │  │   │ Committee N │              │   likely to be picked)  │          │  │
  │  │   └─────────────┘              └───────────┬──────────────┘          │  │
  │  │                                            │                         │  │
  │  │                                            ▼                         │  │
  │  │                                 ┌──────────────────────┐             │  │
  │  │                                 │   PTC (512 members)  │             │  │
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
  │  │   └─► Did payload       │   included in    │   └─► Signed by voter   │  │
  │  │       arrive on time?   │                  │                         │  │
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
  │  │   Withdrawal            │   contains   │   ValidatorIndex        │      │
  │  │                         │              │   └─► Who pays          │      │
  │  │                         │              │                         │      │
  │  │                         │              │ withdrawable_epoch:     │      │
  │  │                         │              │   Epoch                 │      │
  │  │                         │              │   └─► When withdrawable │      │
  │  └─────────────────────────┘              └─────────────────────────┘      │
  │                                                                             │
  │  LIFECYCLE:                                                                 │
  │  ══════════                                                                 │
  │                                                                             │
  │  ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐   │
  │  │ Bid included in  │     │ Quorum reached   │     │ Withdrawable     │   │
  │  │ block            │ ──► │ at epoch boundary│ ──► │ epoch reached    │   │
  │  │                  │     │                  │     │                  │   │
  │  │ builder_pending_ │     │ Moved to         │     │ Converted to     │   │
  │  │ payments[slot]   │     │ builder_pending_ │     │ actual Withdrawal│   │
  │  │ created          │     │ withdrawals      │     │ in execution     │   │
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
  │             is_builder_payment_withdrawable()                               │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  ---

⏺ 5. FORK CHOICE CHANGES: Empty vs Full Blocks

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
  │              │ and validates   │             │ arrive (timeout)│           │
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
  │  │ PENDING (0)       │ Fork choice doesn't know yet if it's full/empty│    │
  │  ├───────────────────┼────────────────────────────────────────────────┤    │
  │  │ PAYLOAD_STATUS_   │ Payload was NOT delivered (or not valid)       │    │
  │  │ EMPTY (1)         │ Block is "empty" - no execution happened       │    │
  │  ├───────────────────┼────────────────────────────────────────────────┤    │
  │  │ PAYLOAD_STATUS_   │ Payload WAS delivered and validated            │    │
  │  │ FULL (2)          │ Block is "full" - execution completed          │    │
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
  │      payload_status: uint8   # PENDING=0, EMPTY=1, FULL=2                   │
  │                                                                             │
  │  A single beacon block root can have MULTIPLE nodes in fork choice!         │
  │  One for each payload status (but PENDING is transitional).                 │
  │                                                                             │
  │  SPEC: fork-choice.md                                                       │
  │  CONSTANTS: PAYLOAD_STATUS_PENDING=0, PAYLOAD_STATUS_EMPTY=1,               │
  │             PAYLOAD_STATUS_FULL=2, PAYLOAD_TIMELY_THRESHOLD=256             │
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
  │    - PTC voted payload present (>256) and payload locally available, OR     │
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
  │      payload_present: bool   # NEW! From attestation.data.index             │
  │                                                                             │
  │  WHY slot instead of epoch?                                                 │
  │  More granular tracking needed because payload status can vary per slot.    │
  │                                                                             │
  │  SPEC: fork-choice.md, validator.md                                         │
  │  CONTAINERS: LatestMessage (modified to use slot instead of epoch)          │
  │  FUNCTIONS: update_latest_messages(), validate_on_attestation()             │
  │             is_attestation_same_slot(), is_supporting_vote()                │
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
  │  │      # NEW: Determine which parent state to use                     │   │
  │  │      if is_parent_node_full(store, block):                          │   │
  │  │          # Parent had payload → use post-payload state              │   │
  │  │          state = store.execution_payload_states[parent_root]        │   │
  │  │      else:                                                          │   │
  │  │          # Parent was empty → use post-block state                  │   │
  │  │          state = store.block_states[parent_root]                    │   │
  │  │                                                                     │   │
  │  │      # Process block...                                             │   │
  │  │      store.blocks[root] = block                                     │   │
  │  │      store.block_states[root] = state   # Post-block state          │   │
  │  │                                                                     │   │
  │  │      # NEW: Initialize PTC vote tracking                            │   │
  │  │      store.ptc_vote[root] = [False] * 512                           │   │
  │  │                                                                     │   │
  │  │      # NEW: Process payload attestations from previous slot         │   │
  │  │      notify_ptc_messages(store, state, block.body.payload_attestations)│
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  on_execution_payload (NEW)                                                 │
  │  ══════════════════════════                                                 │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  def on_execution_payload(store, signed_envelope):                  │   │
  │  │      envelope = signed_envelope.message                             │   │
  │  │                                                                     │   │
  │  │      # Beacon block must be known                                   │   │
  │  │      assert envelope.beacon_block_root in store.block_states        │   │
  │  │                                                                     │   │
  │  │      # Check blob data availability                                 │   │
  │  │      assert is_data_available(envelope.beacon_block_root)           │   │
  │  │                                                                     │   │
  │  │      # Get post-block state                                         │   │
  │  │      state = copy(store.block_states[envelope.beacon_block_root])   │   │
  │  │                                                                     │   │
  │  │      # Process the execution payload (full validation)              │   │
  │  │      process_execution_payload(state, signed_envelope, EXECUTION_ENGINE)│
  │  │                                                                     │   │
  │  │      # Store the post-payload state                                 │   │
  │  │      store.execution_payload_states[envelope.beacon_block_root] = state│
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
  │  │      # Verify the validator is in the PTC                           │   │
  │  │      assert msg.validator_index in ptc                              │   │
  │  │                                                                     │   │
  │  │      # Update the PTC vote tracking                                 │   │
  │  │      ptc_index = ptc.index(msg.validator_index)                     │   │
  │  │      store.ptc_vote[data.beacon_block_root][ptc_index] = data.payload_present│
  │  │      #                                                              │   │
  │  │      # True = "I saw the payload"                                   │   │
  │  │      # False = "I did NOT see the payload"                          │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  is_payload_timely - How PTC votes are evaluated:                           │
  │  ════════════════════════════════════════════════                           │
  │                                                                             │
  │  def is_payload_timely(store, root):                                        │
  │      # Payload must be locally available                                    │
  │      if root not in store.execution_payload_states:                         │
  │          return False                                                       │
  │                                                                             │
  │      # More than half of PTC must have voted "present"                      │
  │      return sum(store.ptc_vote[root]) > PAYLOAD_TIMELY_THRESHOLD  # >256    │
  │                                                                             │
  │  SPEC: fork-choice.md                                                       │
  │  HANDLERS: on_block() (modified), on_execution_payload() (new)              │
  │            on_payload_attestation_message() (new)                           │
  │  FUNCTIONS: is_payload_timely(), notify_ptc_messages()                      │
  │             get_parent_payload_status(), is_parent_node_full()              │
  │             should_extend_payload(), get_payload_status_tiebreaker()        │
  │  STORE FIELDS: execution_payload_states, ptc_vote (new)                     │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  ---

⏺ 6. NETWORKING & P2P CHANGES

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
  │         (votes on payload timeliness)                                       │
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
  │  │   • builder_index is not valid, active, and non-slashed            │   │
  │  │   • builder doesn't have 0x03 (BUILDER_WITHDRAWAL_PREFIX)          │   │
  │  │   • execution_payment is zero (reserved field)                     │   │
  │  │   • signature is invalid                                           │   │
  │  │                                                                     │   │
  │  │ IGNORE if:                                                          │   │
  │  │   • Already seen valid bid from this builder for this slot         │   │
  │  │   • Not the highest value bid for this slot+parent                 │   │
  │  │   • Builder doesn't have sufficient balance                        │   │
  │  │   • parent_block_hash unknown in fork choice                       │   │
  │  │   • parent_block_root unknown in fork choice                       │   │
  │  │   • slot is not current or next                                    │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  execution_payload                                                          │
  │  ═════════════════                                                          │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │ REJECT if:                                                          │   │
  │  │   • Referenced block doesn't pass validation                        │   │
  │  │   • slot doesn't match block.slot                                   │   │
  │  │   • builder_index doesn't match bid.builder_index                   │   │
  │  │   • payload.block_hash doesn't match bid.block_hash                 │   │
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
  │    kzg_commitments                        kzg_commitments                   │
  │    kzg_proofs                             kzg_proofs                        │
  │                                                                             │
  │    signed_block_header ❌ REMOVED                                           │
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
  │          2. Check hash(sidecar.kzg_commitments) == bid.blob_kzg_commitments_root│
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
  │  │  def verify_data_column_sidecar(sidecar):                           │   │
  │  │      # Index must be valid                                          │   │
  │  │      if sidecar.index >= NUMBER_OF_COLUMNS: return False            │   │
  │  │                                                                     │   │
  │  │      # Must have blobs                                              │   │
  │  │      if len(sidecar.kzg_commitments) == 0: return False             │   │
  │  │                                                                     │   │
  │  │      # Must respect blob limit for this epoch                       │   │
  │  │      epoch = compute_epoch_at_slot(sidecar.slot)                    │   │
  │  │      max_blobs = get_blob_parameters(epoch).max_blobs_per_block     │   │
  │  │      if len(sidecar.kzg_commitments) > max_blobs: return False      │   │
  │  │                                                                     │   │
  │  │      # Consistent lengths                                           │   │
  │  │      if len(column) != len(commitments) != len(proofs): return False│   │
  │  │                                                                     │   │
  │  │      return True                                                    │   │
  │  │                                                                     │   │
  │  │  # On gossip, ALSO check:                                           │   │
  │  │  # - beacon_block_root seen via valid bid                           │   │
  │  │  # - slot matches block slot                                        │   │
  │  │  # - hash(kzg_commitments) matches bid.blob_kzg_commitments_root    │   │
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
  │  GOSSIP: beacon_block, execution_payload_bid, execution_payload,            │
  │          payload_attestation_message, data_column_sidecar_{subnet_id}       │
  │  REQ/RESP: ExecutionPayloadEnvelopesByRange v1 (new)                        │
  │            ExecutionPayloadEnvelopesByRoot v1 (new)                         │
  │            BeaconBlocksByRange v2, BeaconBlocksByRoot v2 (updated)          │
  │  CONTAINERS: DataColumnSidecar (modified - removed signed_block_header)     │
  │  FUNCTIONS: compute_fork_version(), verify_data_column_sidecar()            │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  ---

⏺ 7. ACTOR PERSPECTIVES

  7.1 How to Become a Builder

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                    BECOMING A BUILDER: Step by Step                         │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  STEP 1: Create Builder Withdrawal Credentials                              │
  │  ═════════════════════════════════════════════                              │
  │                                                                             │
  │  Regular Validator:                   Builder:                              │
  │  ┌──────────────────────────┐        ┌──────────────────────────┐          │
  │  │ withdrawal_credentials   │        │ withdrawal_credentials   │          │
  │  │                          │        │                          │          │
  │  │ 0x01 + 11 zeros + addr   │        │ 0x03 + 11 zeros + addr   │          │
  │  │ ────                     │        │ ────                     │          │
  │  │ BLS withdrawal prefix    │        │ BUILDER_WITHDRAWAL_PREFIX│          │
  │  │                          │        │                          │          │
  │  │ OR                       │        │ The 0x03 prefix is what  │          │
  │  │                          │        │ makes you a BUILDER!     │          │
  │  │ 0x02 + 11 zeros + addr   │        │                          │          │
  │  │ ────                     │        │ Builders also get        │          │
  │  │ Compounding prefix       │        │ compounding benefits     │          │
  │  └──────────────────────────┘        └──────────────────────────┘          │
  │                                                                             │
  │  STEP 2: Submit Deposit                                                     │
  │  ══════════════════════                                                     │
  │                                                                             │
  │  Same as regular validator deposit, but with 0x03 credentials:              │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │ deposit_data = DepositData(                                         │   │
  │  │     pubkey = your_bls_pubkey,                                       │   │
  │  │     withdrawal_credentials = 0x03 + 0x00*11 + your_eth_address,     │   │
  │  │     amount = MIN_DEPOSIT_AMOUNT (or more),  # At least 1 ETH        │   │
  │  │     signature = sign(deposit_data)                                  │   │
  │  │ )                                                                   │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  │  STEP 3: Wait for Activation                                                │
  │  ═══════════════════════════                                                │
  │                                                                             │
  │  Same activation process as validators:                                     │
  │  • Deposit processed → pending_deposits queue                               │
  │  • Balance reaches MIN_ACTIVATION_BALANCE (32 ETH)                          │
  │  • Enters activation queue                                                  │
  │  • Becomes active after activation epoch                                    │
  │                                                                             │
  │  STEP 4: You're a Builder!                                                  │
  │  ═════════════════════════                                                  │
  │                                                                             │
  │  Once active, you can:                                                      │
  │  • Submit bids to proposers                                                 │
  │  • Have bids included in blocks                                             │
  │  • Reveal payloads and earn MEV                                             │
  │  • Pay proposers from your stake                                            │
  │                                                                             │
  │  KEY DIFFERENCES FROM REGULAR VALIDATORS:                                   │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  Builders:                        Regular Validators:               │   │
  │  │  • CAN submit execution bids      • CANNOT submit bids              │   │
  │  │  • CAN pay proposers              • Don't have payment mechanism    │   │
  │  │  • Have 0x03 credentials          • Have 0x01 or 0x02               │   │
  │  │  • Subject to builder penalties   • Only standard penalties         │   │
  │  │  • CANNOT propose blocks          • CAN propose blocks              │   │
  │  │    (unless also a proposer)                                         │   │
  │  │                                                                     │   │
  │  │  Note: You can be BOTH! A validator can also be a builder.          │   │
  │  │  Self-building: proposer uses their own builder identity.           │   │
  │  │                                                                     │   │
  │  └─────────────────────────────────────────────────────────────────────┘   │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  7.2 Builder Activity: Submitting Bids and Payloads

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                    BUILDER WORKFLOW: Bids and Payloads                      │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  PHASE 1: Construct the Payload (before slot)                               │
  │  ════════════════════════════════════════════                               │
  │                                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────┐   │
  │  │                                                                     │   │
  │  │  1. Call execution engine: engine_getPayloadV5                      │   │
  │  │                                                                     │   │
  │  │  2. Receive:                                                        │   │
  │  │     • execution_payload (transactions, withdrawals, etc.)           │   │
  │  │     • blobs_bundle (blobs, commitments, proofs)                     │   │
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
  │  │      parent_block_hash = state.latest_block_hash,                   │   │
  │  │      parent_block_root = hash_tree_root(state.latest_block_header), │   │
  │  │      block_hash = payload.block_hash,          # COMMITMENT!        │   │
  │  │      prev_randao = payload.prev_randao,                             │   │
  │  │      fee_recipient = proposer_address,         # Who gets paid      │   │
  │  │      gas_limit = payload.gas_limit,                                 │   │
  │  │      builder_index = my_validator_index,                            │   │
  │  │      slot = target_slot,                       # Current or next    │   │
  │  │      value = payment_amount,                   # What I'll pay      │   │
  │  │      blob_kzg_commitments_root = hash_tree_root(blobs.commitments), │   │
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
  │  │    if included_bid.builder_index == my_validator_index:             │   │
  │  │        # MY BID WAS SELECTED! 🎉                                    │   │
  │  │        # Now I MUST reveal the payload (or still pay!)              │   │
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
  │  │      builder_index = my_validator_index,                            │   │
  │  │      beacon_block_root = hash_tree_root(received_block),            │   │
  │  │      slot = received_block.slot,                                    │   │
  │  │      blob_kzg_commitments = stored_commitments,                     │   │
  │  │      state_root = compute_post_state_root(...)  # After processing │   │
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
  │  │  If builder doesn't reveal payload:                                 │   │
  │  │                                                                     │   │
  │  │  • PTC votes "payload_present = false"                              │   │
  │  │  • Payment still goes through if same-slot quorum reached!          │   │
  │  │  • Builder loses MEV opportunity                                    │   │
  │  │  • But proposer still gets paid (builder's stake)                   │   │
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
  │  │      # Must vote on payload timeliness at 75% of that slot          │   │
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
  │  │  2. Check: Did I see an ExecutionPayloadEnvelope for this block?    │   │
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
  │  │  2. Select a bid (highest value? most trusted builder?)             │   │
  │  │                                                                     │   │
  │  │  3. Verify bid is valid:                                            │   │
  │  │     • Builder is active and not slashed                             │   │
  │  │     • Builder has 0x03 credentials (unless self-build)              │   │
  │  │     • Builder has sufficient balance                                │   │
  │  │     • Bid slot matches current slot                                 │   │
  │  │     • Bid parent matches my parent                                  │   │
  │  │     • Signature valid (unless self-build with infinity sig)         │   │
  │  │                                                                     │   │
  │  │  4. Include bid in block:                                           │   │
  │  │     block.body.signed_execution_payload_bid = selected_bid          │   │
  │  │                                                                     │   │
  │  │  5. Include payload attestations from previous slot:                │   │
  │  │     block.body.payload_attestations = aggregate(ptc_messages)       │   │
  │  │                                                                     │   │
  │  │  6. NO LONGER include execution_payload in block!                   │   │
  │  │                                                                     │   │
  │  │  7. NO LONGER responsible for DataColumnSidecar distribution        │   │
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
  │  │      builder_index = proposer_index,   # Same as proposer!          │   │
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
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  ---

⏺ 8. PUTTING IT ALL TOGETHER

  8.1 Complete Message Flow Diagram

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                    COMPLETE GLOAS SLOT FLOW                                 │
  ├─────────────────────────────────────────────────────────────────────────────┤
  │                                                                             │
  │  SLOT N-1 (Preparation)                                                     │
  │  ══════════════════════                                                     │
  │                                                                             │
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
  │      │ to vote)│        │ to vote) │        │ payload)  │                   │
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
  │      │   PTC    │ 512 validators vote on payload timeliness                 │
  │      │ (512     │                                                           │
  │      │ members) │                                                           │
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
  │      • New signed_execution_payload_bid for slot N+1                        │
  │      • State reflects whether slot N had FULL or EMPTY block                │
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
  │      │        → Set withdrawable_epoch                                  │   │
  │      │        → Builder will pay proposer!                              │   │
  │      │    else:                                                         │   │
  │      │        → Discard payment                                         │   │
  │      │        → Builder keeps stake                                     │   │
  │      │        → (Maybe attack detected, or network issues)              │   │
  │      │                                                                  │   │
  │      └──────────────────────────────────────────────────────────────────┘   │
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
  │  • PTC: Verify payload delivery timeliness                                  │
  │  • All enforced at protocol level!                                          │
  │                                                                             │
  │  2. UNCONDITIONAL PAYMENT                                                   │
  │  ════════════════════════                                                   │
  │  • Builder commits to payment via bid                                       │
  │  • Payment happens regardless of payload reveal                             │
  │  • Prevents griefing attacks on proposers                                   │
  │  • Builder stake backs the payment                                          │
  │                                                                             │
  │  3. TWO-PHASE STATE TRANSITION                                              │
  │  ═════════════════════════════                                              │
  │  • Phase 1: Process beacon block (bid committed)                            │
  │  • Phase 2: Process execution payload (if revealed)                         │
  │  • Fork choice tracks EMPTY vs FULL versions                                │
  │                                                                             │
  │  4. NEW VALIDATOR TYPE: BUILDER                                             │
  │  ═════════════════════════════                                              │
  │  • 0x03 withdrawal credential prefix                                        │
  │  • Same activation as validators                                            │
  │  • Can submit bids and pay proposers                                        │
  │  • Subject to slashing (affects payment timing)                             │
  │                                                                             │
  │  5. PAYLOAD TIMELINESS COMMITTEE (PTC)                                      │
  │  ═════════════════════════════════════                                      │
  │  • 512 validators per slot                                                  │
  │  • Balance-weighted selection (sybil resistant)                             │
  │  • Vote at 75% of slot                                                      │
  │  • Votes on payload timeliness (separate from payment quorum)               │
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
  │  • Sidecar structure simplified (no header/proof)                           │
  │  • Verified via bid.blob_kzg_commitments_root                               │
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
  │  │ (same as gloas.md)│ • Modified BeaconState and BeaconBlockBody       │   │
  │  │                   │ • State transition (process_block, etc.)         │   │
  │  │                   │ • Payment quorum and withdrawal logic            │   │
  │  │                   │ • Builder credentials (0x03 prefix)              │   │
  │  ├───────────────────┼──────────────────────────────────────────────────┤   │
  │  │ fork-choice.md    │ • PayloadStatus (PENDING/EMPTY/FULL)             │   │
  │  │                   │ • ForkChoiceNode with payload tracking           │   │
  │  │                   │ • Modified get_head() algorithm                  │   │
  │  │                   │ • on_execution_payload handler                   │   │
  │  │                   │ • PTC vote tracking (is_payload_timely)          │   │
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
  │  │ builder.md        │ • How to become a builder (0x03 credentials)     │   │
  │  │                   │ • Bid construction workflow                      │   │
  │  │                   │ • Payload envelope construction                  │   │
  │  │                   │ • DataColumnSidecar creation                     │   │
  │  │                   │ • Honest withholding (when to not reveal)        │   │
  │  ├───────────────────┼──────────────────────────────────────────────────┤   │
  │  │ fork.md           │ • GLOAS_FORK_VERSION and GLOAS_FORK_EPOCH        │   │
  │  │                   │ • upgrade_to_gloas() function                    │   │
  │  │                   │ • State migration from Fulu                      │   │
  │  └───────────────────┴──────────────────────────────────────────────────┘   │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘

  ---
  Summary

  GLOAS (EIP-7732) is a fundamental upgrade that enshrines Proposer-Builder Separation (ePBS) directly into the Ethereum consensus protocol. The key innovation is replacing trusted relays with protocol-enforced commitments:

  1. Builders stake ETH and submit cryptographic commitments (bids)
  2. Proposers select bids and include them in beacon blocks
  3. Builders reveal payloads after seeing their bid was selected
  4. PTC (512 validators) votes on whether payloads arrived on time
  5. Protocol finalizes payment via same-slot attestation quorum

  This creates a trustless, decentralized block building market where:
  - Builders can't grief proposers if quorum is reached
  - Proposers can't steal MEV (builder reveals after block)
  - No centralized relay needed (protocol is the escrow)
```

---

## 9. SPEC REFERENCE INDEX

Quick lookup for constants, containers, and functions defined in the GLOAS specs.

### 9.1 Constants

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CONSTANTS                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  beacon-chain.md - Domain Types                                              │
│  ───────────────────────────────                                             │
│  DOMAIN_BEACON_BUILDER              DomainType('0x0B000000')                 │
│  DOMAIN_PTC_ATTESTER                DomainType('0x0C000000')                 │
│                                                                              │
│  beacon-chain.md - Misc                                                      │
│  ─────────────────────                                                       │
│  BUILDER_PAYMENT_THRESHOLD_NUMERATOR      uint64(6)                          │
│  BUILDER_PAYMENT_THRESHOLD_DENOMINATOR    uint64(10)                         │
│                                                                              │
│  beacon-chain.md - Withdrawal Prefixes                                       │
│  ─────────────────────────────────────                                       │
│  BUILDER_WITHDRAWAL_PREFIX          Bytes1('0x03')                           │
│                                                                              │
│  beacon-chain.md - Preset                                                    │
│  ────────────────────────                                                    │
│  PTC_SIZE                           uint64(512)                              │
│  MAX_PAYLOAD_ATTESTATIONS           4                                        │
│  BUILDER_PENDING_WITHDRAWALS_LIMIT  uint64(1,048,576)                        │
│                                                                              │
│  fork-choice.md - Constants                                                  │
│  ──────────────────────────                                                  │
│  PAYLOAD_TIMELY_THRESHOLD           PTC_SIZE // 2 (= 256)                    │
│  PAYLOAD_STATUS_PENDING             PayloadStatus(0)                         │
│  PAYLOAD_STATUS_EMPTY               PayloadStatus(1)                         │
│  PAYLOAD_STATUS_FULL                PayloadStatus(2)                         │
│                                                                              │
│  validator.md - Time Parameters                                              │
│  ──────────────────────────────                                              │
│  ATTESTATION_DUE_BPS_GLOAS          uint64(2500)  (25% = 3s)                 │
│  AGGREGATE_DUE_BPS_GLOAS            uint64(5000)  (50% = 6s)                 │
│  SYNC_MESSAGE_DUE_BPS_GLOAS         uint64(2500)  (25% = 3s)                 │
│  CONTRIBUTION_DUE_BPS_GLOAS         uint64(5000)  (50% = 6s)                 │
│  PAYLOAD_ATTESTATION_DUE_BPS        uint64(7500)  (75% = 9s)                 │
│                                                                              │
│  p2p-interface.md - Configuration                                            │
│  ────────────────────────────────                                            │
│  MAX_REQUEST_PAYLOADS               uint64(128)                              │
│                                                                              │
│  fork.md - Configuration                                                     │
│  ───────────────────────                                                     │
│  GLOAS_FORK_VERSION                 Version('0x07000000')                    │
│  GLOAS_FORK_EPOCH                   TBD                                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.2 Containers

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CONTAINERS                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  beacon-chain.md - New Containers                                            │
│  ────────────────────────────────                                            │
│  BuilderPendingPayment         { weight, withdrawal }                        │
│  BuilderPendingWithdrawal      { fee_recipient, amount, builder_index,       │
│                                  withdrawable_epoch }                        │
│  PayloadAttestationData        { beacon_block_root, slot, payload_present,   │
│                                  blob_data_available }                       │
│  PayloadAttestation            { aggregation_bits, data, signature }         │
│  PayloadAttestationMessage     { validator_index, data, signature }          │
│  IndexedPayloadAttestation     { attesting_indices, data, signature }        │
│  ExecutionPayloadBid           { parent_block_hash, parent_block_root,       │
│                                  block_hash, prev_randao, fee_recipient,     │
│                                  gas_limit, builder_index, slot, value,      │
│                                  execution_payment, blob_kzg_commitments_root }│
│  SignedExecutionPayloadBid     { message, signature }                        │
│  ExecutionPayloadEnvelope      { payload, execution_requests, builder_index, │
│                                  beacon_block_root, slot,                    │
│                                  blob_kzg_commitments, state_root }          │
│  SignedExecutionPayloadEnvelope { message, signature }                       │
│                                                                              │
│  beacon-chain.md - Modified Containers                                       │
│  ─────────────────────────────────────                                       │
│  BeaconBlockBody               + signed_execution_payload_bid                │
│                                + payload_attestations                        │
│                                - execution_payload                           │
│                                - blob_kzg_commitments                        │
│                                - execution_requests                          │
│  BeaconState                   + latest_execution_payload_bid                │
│                                + execution_payload_availability              │
│                                + builder_pending_payments                    │
│                                + builder_pending_withdrawals                 │
│                                + latest_block_hash                           │
│                                + payload_expected_withdrawals                │
│                                - latest_execution_payload_header             │
│                                                                              │
│  fork-choice.md - New/Modified                                               │
│  ─────────────────────────────                                               │
│  ForkChoiceNode                { root, payload_status }                      │
│  LatestMessage                 { slot, root, payload_present }               │
│  Store                         + execution_payload_states                    │
│                                + ptc_vote                                    │
│                                                                              │
│  p2p-interface.md - Modified                                                 │
│  ───────────────────────────                                                 │
│  DataColumnSidecar             + slot                                        │
│                                + beacon_block_root                           │
│                                - signed_block_header                         │
│                                - kzg_commitments_inclusion_proof             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.3 Functions

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              FUNCTIONS                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  beacon-chain.md - Predicates (New)                                          │
│  ──────────────────────────────────                                          │
│  is_builder_withdrawal_credential(withdrawal_credentials)                    │
│  has_builder_withdrawal_credential(validator)                                │
│  is_attestation_same_slot(state, data)                                       │
│  is_valid_indexed_payload_attestation(state, indexed_payload_attestation)    │
│  is_parent_block_full(state)                                                 │
│                                                                              │
│  beacon-chain.md - Predicates (Modified)                                     │
│  ───────────────────────────────────────                                     │
│  has_compounding_withdrawal_credential(validator)                            │
│                                                                              │
│  beacon-chain.md - Misc (New)                                                │
│  ────────────────────────────                                                │
│  compute_balance_weighted_selection(state, indices, seed, size, shuffle)     │
│  compute_balance_weighted_acceptance(state, index, seed, i)                  │
│                                                                              │
│  beacon-chain.md - Misc (Modified)                                           │
│  ─────────────────────────────────                                           │
│  get_pending_balance_to_withdraw(state, validator_index)                     │
│  compute_proposer_indices(state, epoch, seed, indices)                       │
│                                                                              │
│  beacon-chain.md - State Accessors (New)                                     │
│  ───────────────────────────────────────                                     │
│  get_ptc(state, slot)                                                        │
│  get_indexed_payload_attestation(state, payload_attestation)                 │
│  get_builder_payment_quorum_threshold(state)                                 │
│                                                                              │
│  beacon-chain.md - State Accessors (Modified)                                │
│  ────────────────────────────────────────────                                │
│  get_next_sync_committee_indices(state)                                      │
│  get_attestation_participation_flag_indices(state, data, inclusion_delay)    │
│                                                                              │
│  beacon-chain.md - State Transition (New)                                    │
│  ─────────────────────────────────────────                                   │
│  process_builder_pending_payments(state)                                     │
│  is_builder_payment_withdrawable(state, withdrawal)                          │
│  get_builder_withdrawable_balance(builder, balance)                          │
│  get_builder_withdrawals(state, withdrawal_index, prior_withdrawals)         │
│  update_payload_expected_withdrawals(state, withdrawals)                     │
│  update_builder_pending_withdrawals(state, count)                            │
│  verify_execution_payload_bid_signature(state, signed_bid)                   │
│  process_execution_payload_bid(state, block)                                 │
│  process_payload_attestation(state, payload_attestation)                     │
│  verify_execution_payload_envelope_signature(state, signed_envelope)         │
│  process_execution_payload(state, signed_envelope, execution_engine)         │
│                                                                              │
│  beacon-chain.md - State Transition (Modified)                               │
│  ──────────────────────────────────────────────                              │
│  process_slot(state)                                                         │
│  process_epoch(state)                                                        │
│  get_expected_withdrawals(state)                                             │
│  process_withdrawals(state)                                                  │
│  process_operations(state, body)                                             │
│  process_attestation(state, attestation)                                     │
│  process_proposer_slashing(state, proposer_slashing)                         │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  fork-choice.md - Helpers (New)                                              │
│  ──────────────────────────────                                              │
│  notify_ptc_messages(store, state, payload_attestations)                     │
│  is_payload_timely(store, root)                                              │
│  get_parent_payload_status(store, block)                                     │
│  is_parent_node_full(store, block)                                           │
│  is_supporting_vote(store, node, message)                                    │
│  should_extend_payload(store, root)                                          │
│  get_payload_status_tiebreaker(store, node)                                  │
│  get_node_children(store, blocks, node)                                      │
│  get_payload_attestation_due_ms(epoch)                                       │
│                                                                              │
│  fork-choice.md - Helpers (Modified)                                         │
│  ───────────────────────────────────                                         │
│  update_latest_messages(store, attesting_indices, attestation)               │
│  get_forkchoice_store(anchor_state, anchor_block)                            │
│  get_ancestor(store, root, slot)                                             │
│  get_checkpoint_block(store, root, epoch)                                    │
│  get_weight(store, node)                                                     │
│  get_head(store)                                                             │
│  get_attestation_due_ms(epoch)                                               │
│  get_aggregate_due_ms(epoch)                                                 │
│  get_sync_message_due_ms(epoch)                                              │
│  get_contribution_due_ms(epoch)                                              │
│                                                                              │
│  fork-choice.md - Handlers (New)                                             │
│  ────────────────────────────────                                            │
│  on_execution_payload(store, signed_envelope)                                │
│  on_payload_attestation_message(store, ptc_message, is_from_block)           │
│                                                                              │
│  fork-choice.md - Handlers (Modified)                                        │
│  ─────────────────────────────────────                                       │
│  on_block(store, signed_block)                                               │
│  validate_on_attestation(store, attestation, is_from_block)                  │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  validator.md - Functions (New)                                              │
│  ──────────────────────────────                                              │
│  get_ptc_assignment(state, epoch, validator_index)                           │
│  get_payload_attestation_message_signature(state, attestation, privkey)      │
│                                                                              │
│  validator.md - Functions (Modified)                                         │
│  ───────────────────────────────────                                         │
│  prepare_execution_payload(state, safe_block_hash, ...)                      │
│  get_data_column_sidecars_from_column_sidecar(sidecar, cells_and_kzg_proofs) │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  builder.md - Functions (New)                                                │
│  ────────────────────────────                                                │
│  get_execution_payload_bid_signature(state, bid, privkey)                    │
│  get_data_column_sidecars(beacon_block_root, slot, kzg_commitments, ...)     │
│  get_data_column_sidecars_from_block(signed_block, blob_kzg_commitments, ...)│
│  get_execution_payload_envelope_signature(state, envelope, privkey)          │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  p2p-interface.md - Functions (Modified)                                     │
│  ───────────────────────────────────────                                     │
│  compute_fork_version(epoch)                                                 │
│  verify_data_column_sidecar(sidecar)                                         │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  fork.md - Functions (New)                                                   │
│  ─────────────────────────                                                   │
│  upgrade_to_gloas(pre)                                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.4 Gossip Topics

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           GOSSIP TOPICS (p2p-interface.md)                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  NEW Global Topics                                                           │
│  ─────────────────                                                           │
│  execution_payload_bid          SignedExecutionPayloadBid                    │
│  execution_payload              SignedExecutionPayloadEnvelope               │
│  payload_attestation_message    PayloadAttestationMessage                    │
│                                                                              │
│  MODIFIED Topics                                                             │
│  ───────────────                                                             │
│  beacon_block                   SignedBeaconBlock (modified type)            │
│  beacon_aggregate_and_proof     + index validation for payload status        │
│  beacon_attestation_{subnet_id} + index validation for payload status        │
│  data_column_sidecar_{subnet_id} Modified validation rules                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.5 Req/Resp Methods

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       REQ/RESP METHODS (p2p-interface.md)                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  NEW Methods                                                                 │
│  ───────────                                                                 │
│  ExecutionPayloadEnvelopesByRange v1                                         │
│    Protocol: /eth2/beacon_chain/req/execution_payload_envelopes_by_range/1/  │
│    Request:  { start_slot, count }                                           │
│    Response: List[SignedExecutionPayloadEnvelope]                            │
│                                                                              │
│  ExecutionPayloadEnvelopesByRoot v1                                          │
│    Protocol: /eth2/beacon_chain/req/execution_payload_envelopes_by_root/1/   │
│    Request:  List[Root, MAX_REQUEST_PAYLOADS]                                │
│    Response: List[SignedExecutionPayloadEnvelope, MAX_REQUEST_PAYLOADS]      │
│                                                                              │
│  UPDATED Methods                                                             │
│  ───────────────                                                             │
│  BeaconBlocksByRange v2    + GLOAS_FORK_VERSION → gloas.SignedBeaconBlock    │
│  BeaconBlocksByRoot v2     + GLOAS_FORK_VERSION → gloas.SignedBeaconBlock    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

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

### Q2: Do builders need 64 ETH (32 + 32)?

**No!** Builders need **32 ETH to activate** (same as any validator). The 32 ETH is a **reserve** they must keep - they can only use balance **above** 32 ETH for payments.

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
│  │   │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│░░░░░░░░░░░░░░░░░░░░░░░░░░│  │   │
│  │   │         32 ETH                 │        18 ETH              │  │   │
│  │   │      (LOCKED RESERVE)          │   (AVAILABLE FOR BIDS)     │  │   │
│  │   │   MIN_ACTIVATION_BALANCE       │                            │  │   │
│  │   └─────────────────────────────────────────────────────────────┘  │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  The builder can bid UP TO 18 ETH (if no pending obligations)               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### Q3: What are pending_payments and pending_withdrawals in the bid validation?

From `process_execution_payload_bid`:
```python
assert (
    amount == 0
    or state.balances[builder_index]
    >= amount + pending_payments + pending_withdrawals + MIN_ACTIVATION_BALANCE
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
│    Check: balance >= amount + pending_payments + pending_withdrawals        │
│                       + MIN_ACTIVATION_BALANCE                              │
│                                                                             │
│            50 ETH >= X + 5 ETH + 0 ETH + 32 ETH                              │
│            50 ETH >= X + 37 ETH                                             │
│            X <= 13 ETH  ◄── Max new bid!                                    │
│                                                                             │
│    Bob can bid up to 13 ETH on slot 101                                     │
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
│    Check: 50 ETH >= X + 10 ETH + 5 ETH + 32 ETH                              │
│            50 ETH >= X + 47 ETH                                             │
│            X <= 3 ETH  ◄── Max new bid!                                     │
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
│    Now Bob can bid: 45 - 10 - 0 - 32 = 3 ETH (same, balance dropped too)    │
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
│  All three stages count against available balance for new bids!             │
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
│  Theoretically 2² = 4 combinations, but one is impossible:                  │
│                                                                             │
│  ┌──────────────────┬─────────────────────┬─────────────────────────────┐  │
│  │ payload_present  │ blob_data_available │ Valid?                      │  │
│  ├──────────────────┼─────────────────────┼─────────────────────────────┤  │
│  │ true             │ true                │ ✓ Builder revealed all      │  │
│  │ true             │ false               │ ✓ Payload seen, blobs lost  │  │
│  │ false            │ false               │ ✓ Builder withheld all      │  │
│  │ false            │ true                │ ✗ IMPOSSIBLE - can't have   │  │
│  │                  │                     │   blobs without payload     │  │
│  └──────────────────┴─────────────────────┴─────────────────────────────┘  │
│                                                                             │
│  So we have 3 meaningful combinations. MAX = 4 provides headroom.           │
│                                                                             │
│  IMPORTANT: All payload attestations in a block must reference the SAME     │
│  beacon_block_root (the parent block). From process_payload_attestation:    │
│                                                                             │
│    assert data.beacon_block_root == state.latest_block_header.parent_root   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```
