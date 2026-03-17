# Authorization & Privacy Design Patterns

Daml enforces authorization and privacy at the protocol level. This document catalogs the patterns used in this section with annotations explaining **why** each design decision was made.

---

## Pattern 1: Propose-Accept

**Location**: `02-design-patterns/ProposeAccept.daml`

### Problem
Two parties need to agree to a contract. They cannot submit simultaneously.

### Solution

```
Party A                    Ledger                    Party B
  |                          |                          |
  |-- createCmd Proposal -->  |                          |
  |                          | Proposal (A signs)       |
  |                          |<-- observable to B ------>|
  |                          |                          |
  |                          |<-- exerciseCmd Accept ---|
  |                          |                          |
  |                          | FinalContract (A+B sign) |
```

### Key Code

```daml
template Proposal
  with
    proposer: Party
    acceptor: Party
    details: Text
  where
    signatory proposer
    observer acceptor

    choice Accept: ContractId FinalContract
      controller acceptor
      do create FinalContract with proposer; acceptor; details

    choice Reject: ()
      controller acceptor
      do pure ()

    choice Cancel: ()
      controller proposer
      do pure ()
```

### Why it Works

When `proposer` submits `createCmd Proposal`, the ledger records `proposer`'s authorization.
When `acceptor` exercises `Accept`, the ledger records `acceptor`'s authorization.
The `FinalContract` created inside `Accept` is authorized by BOTH — the proposer's signature carried through from the Proposal.

### Used In

- `CardIssuance.daml`: IssueCardProposal → Card
- `Market.daml`: MarketAppInstallRequest → MarketAppInstall
- `AtomicSwap.daml`: AssetTransferProposal → Asset (new owner)
- `UpgradeModule.daml`: UpgradeProposal → Upgrader

---

## Pattern 2: Delegation

**Location**: `02-design-patterns/Delegation.daml`

### Problem
An agent needs to act on behalf of a principal within a specific scope.

### Solution

```
Principal creates DelegationAgreement (they sign it)
    → this grants the agent authority within the agreement's scope

Agent exercises choices ON the DelegationAgreement
    → agent's action is "backed by" principal's signature on the agreement
```

### Key Code

```daml
template DelegationAgreement
  with
    principal: Party
    agent: Party
  where
    signatory principal
    observer agent

    nonconsuming choice ActOnBehalf: ContractId SomeContract
      with payload: Text
      controller agent   -- agent controls this
      do
        create SomeContract with owner = principal; data = payload
        -- This create is authorized by:
        -- 1. principal (via their signature on DelegationAgreement)
        -- 2. agent (they exercised this choice)
```

### Why Delegation is Safe

The principal CONTROLS what the agent can do:
- The principal chooses what choices to put on the DelegationAgreement
- The agent can only do what those specific choices allow
- The agent cannot "escape" the scope and act as the principal elsewhere

### Contrast with Non-Transitivity

If you try delegation WITHOUT a DelegationAgreement:
```
Bob exercises nt1.ExerciseSwapOnOther with nt2Cid
→ tries to call nt2.SwapSigAndObs (controller = Alice)
→ FAILS: Bob's control over nt1 doesn't transfer to nt2
```

With delegation:
```
Alice creates DelegationAgreement(principal=Alice, agent=Bob)
Bob exercises DelegationAgreement.ActOnBehalf
→ SUCCEEDS: Alice's signature on the agreement authorizes the action
```

### Used In

- `Market.daml`: `MarketAppInstall_OfferCard` — Alice delegates to provider's market contract to create CardOffer
- `AdditionalRules.daml`: Bank delegates SetObservers capability to Position owners

---

## Pattern 3: Multi-Party Lock

**Location**: `SwapMarket/CardIssuance.daml`

### Problem
During a negotiation, an asset must be "held" but not transferred. Multiple parties need assurance the asset won't disappear.

### Solution

```daml
template LockedCard
  with
    cardDetails: CardDetails
    owner: Party
    lock: TimeLock         -- who holds the lock and until when
  where
    signatory owner, lock.holders  -- ALL parties sign (joint custody)
    observer []

    choice LockedCard_Unlock: ContractId Card
      controller lock.holders    -- all holders must agree to unlock
      do create Card with cardDetails; owner

    choice LockedCard_LockExpire: ContractId Card
      with actor: Party
      controller actor           -- any stakeholder can trigger expiry
      do
        now <- getTime
        assertMsg "Lock expired" (lock.expiresAt <= now)
        create Card with cardDetails; owner
```

### Authorization Analysis

```
Card_Lock: controller = owner, lock.holders
  - ALL of {owner, holder1, holder2, ...} must sign simultaneously
  - Any subset fails

LockedCard_Unlock: controller = lock.holders
  - ALL lock holders must sign (owner NOT needed for unlock)
  - This is intentional: holders control the release of the locked asset

LockedCard_LockExpire: controller = actor (any stakeholder)
  - Anyone involved can trigger expiry once the time has passed
  - Time-based release doesn't need negotiation
```

---

## Pattern 4: Three-Party Signatory

**Location**: `SwapMarket/Market.daml` — SwapProposal

### Problem
A market transaction involves a provider (operator), an offerer, and a proposer. All three need to be accountable.

### Solution

```daml
template SwapProposal
  with
    provider: Party
    offerer: Party
    proposer: Party
    ...
  where
    signatory provider, offerer, proposer
```

### Why All Three Sign

| Party | Why they sign |
|-------|--------------|
| `provider` | Market operator must approve — prevents unauthorized market usage, ensures platform integrity |
| `offerer` | Their card is at stake — they're committed to the offer |
| `proposer` | They initiated — accountable for locking their card |

### How Three-Party Authorization Happens Without Simultaneous Submission

The authorization is accumulated across multiple transactions:
1. `CardOffer_ProposeSwap` is exercised — provider and offerer already signed the CardOffer (they're its signatories). When this choice runs, it creates SwapProposal. The proposer (Bob) is the controller of this choice — so Bob's authorization is added when he exercises it. The provider + offerer authorization carries through from CardOffer.
2. All three are now signatories of SwapProposal.

---

## Pattern 5: Atomic Swap

**Location**: `03-privacy/AtomicSwap.daml`, `SwapMarket/Market.daml`

### Problem
Alice gives X, Bob gives Y — both must happen or neither does.

### Solution

Put both transfers in ONE transaction. Daml guarantees atomicity.

```daml
choice SwapProposal_Accept: ()
  controller recipient
  do
    -- Transfer 1: initiator's asset → recipient
    proposalA <- exercise offerCid Asset_Transfer with recipient
    -- Transfer 2: recipient's asset → initiator
    proposalB <- exercise assetCid Asset_Transfer with recipient = initiator
    -- Accept both in same transaction
    exercise proposalA AssetTransferProposal_Accept
    exercise proposalB AssetTransferProposal_Accept
    -- If any of the above fails → ALL roll back
    pure ()
```

### Why Atomicity is Guaranteed

Every `do` block in a Daml choice is a single transaction. Either ALL operations in the transaction succeed, or ALL roll back. There's no "half committed" state.

---

## Pattern 6: Sub-Transaction Privacy Proof

**Location**: `SwapMarket/Tests/TestMarket.daml`

### The Claim

Even in a multi-party swap, parties only see their own parts of the transaction.

### How to Verify

Run `TestMarket.testSwap` in Daml Studio. Open the transaction tree.

```
Transaction: SwapProposal_Accept
├── [visible to IssuerA]:    AliceCard archived, new Card(owner=bob, issuer=issuerA) created
├── [visible to IssuerB]:    BobCard archived, new Card(owner=alice, issuer=issuerB) created
├── [hidden from IssuerA]:   BobCard operations
└── [hidden from IssuerB]:   AliceCard operations
```

IssuerA sees only contracts where they're a signatory (issuerA is on AliceCard). They don't see BobCard (where issuerB is the signatory).

This is **sub-transaction privacy**: parties see the net effects relevant to them, not the entire transaction.

---

## Anti-Pattern: Trusting Implicit Authorization

Don't assume authorization cascades. Always be explicit.

```daml
-- WRONG: assumes controller of outer choice can do inner thing
nonconsuming choice BadPattern: ContractId OtherContract
  controller alice
  do
    exercise otherCid SomeChoice with ...
    -- This FAILS if SomeChoice requires a different controller!
```

```daml
-- RIGHT: ensure the other contract's controller is also involved
nonconsuming choice GoodPattern: ContractId OtherContract
  controller alice
  do
    -- alice must be the controller of SomeChoice on otherCid,
    -- OR alice must have been delegated authority via a DelegationAgreement
    exercise otherCid SomeChoice with ...
```

The key rule: **Authorization in Daml is non-transitive and non-ambient.** Every action on every contract requires explicit authorization for that specific contract.
