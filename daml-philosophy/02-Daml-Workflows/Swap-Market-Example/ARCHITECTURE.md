# Swap Market Example — Architecture Documentation

## Overview

The Swap Market is the most complete example in this repo — a **trading card marketplace** where users can offer cards for swap and propose/accept trades. It demonstrates advanced Daml patterns including three-party authorization, time locks, divulgence, and atomic settlement.

## System Parties

| Party    | Role                                                          |
|----------|---------------------------------------------------------------|
| IssuerA/B| Issues cards (creates IssueCardProposal)                      |
| Alice    | Market user; creates offers and accepts swap proposals        |
| Bob      | Market user; creates swap proposals                           |
| Provider | Market operator; coordinates offers and proposals             |

## Contract Hierarchy

```
IssueCardProposal ──[Accept]──► Card
    │                              │
    │                              ├──[Card_Transfer]──► Card (new owner)
    │                              │
    │                              └──[Card_Lock]──► LockedCard
    │                                                    │
    │                                                    ├──[LockedCard_Unlock]──► Card
    │                                                    └──[LockedCard_LockExpire]──► Card
    │
MarketAppInstallRequest ──[Accept]──► MarketAppInstall
    │                                        │
    │                                        └──[OfferCard]──► CardOffer
    │                                                             │
    │                                                             └──[ProposeSwap]──► SwapProposal + LockedCard
    │                                                                                      │
    │                                                                                      ├──[Accept]──► 2x Card (swapped)
    │                                                                                      ├──[Reject]──► Card (unlocked)
    │                                                                                      ├──[Withdraw]──► Card (unlocked)
    │                                                                                      └──[Expire]──► (archived)
```

## Complete Swap Workflow

### Step 1: Join the Market
```
Alice ──► MarketAppInstallRequest(provider, user=Alice)
Provider accepts ──► MarketAppInstall(provider, Alice)  [both sign]
```

### Step 2: Make an Offer
```
Alice exercises MarketAppInstall_OfferCard ──► CardOffer(provider, offerer=Alice, cardDetails)
```
`CardOffer` is visible to the provider's interface (the "order book").

### Step 3: Propose a Swap
```
Bob sees CardOffer via provider's interface.
Bob exercises CardOffer_ProposeSwap with:
  - proposedCardId: Bob's card
  - proposalExpiresAt: 30 min from now

Internally:
  - Validates: offer not expired, Bob is market member, Bob owns proposed card
  - Locks Bob's card: Card_Lock(holders=[provider, Alice, Bob], expiresAt)
  - Creates: SwapProposal(provider, offerer=Alice, proposer=Bob, lockedCardId)
```

### Step 4: Accept the Swap
```
Alice exercises SwapProposal_Accept with offeredCardId = Alice's card.

Atomically:
  1. Validate: proposal not expired, Alice owns the card, details match
  2. Transfer offered card (Alice's) to Bob
  3. Unlock Bob's card
  4. Transfer unlocked card (Bob's) to Alice
  5. Return SwapResult with both new card IDs
```

## Authorization Analysis

### Card_Transfer — Mutual Consent
```daml
controller owner, newOwner
```
Both current owner AND new owner must authorize. Prevents unwanted card spam.

### Card_Lock — Multi-party Lock
```daml
controller owner, lock.holders
```
Owner + ALL lock holders must agree to lock. Joint custody during negotiation.

### SwapProposal — Three-party Signatory
```daml
signatory provider, offerer, proposer
```
Why all three:
- **provider**: market operator must approve; prevents unauthorized market usage
- **offerer**: their card is at stake; they commit to honoring the offer
- **proposer**: they initiated; they must be accountable for locking their card

### SwapProposal_Accept — Atomic Settlement
```daml
exercise offeredCardId Card_Transfer with newOwner = proposer
exercise proposedCardId LockedCard_Unlock
exercise proposedCardId Card_Transfer with newOwner = offerer
```
All operations in one transaction: if any fails, ALL roll back. No partial settlement.

## Privacy Analysis

```
IssuerA ──knows──► AliceCard (they issued it)
Alice ──knows──► AliceCard (she owns it)
IssuerB ──knows──► BobCard (they issued it)
Bob ──knows──► BobCard (he owns it)

Provider ──knows──► CardOffer (they're co-signatories)
Provider ──learns──► BobCard (via divulgence during ProposeSwap)
Alice ──learns──► BobCard (via divulgence during ProposeSwap)

IssuerA ──NEVER learns──► BobCard (sub-transaction privacy!)
IssuerB ──NEVER learns──► AliceCard (sub-transaction privacy!)
```

This demonstrates Daml's sub-transaction privacy model: parties only learn what they need to.

## Key Daml Concepts Demonstrated

### Time-based Authorization
```daml
now <- getTime
assertMsg "Lock expired" (lock.expiresAt <= now)
```
`getTime` returns ledger time (monotonic, controlled by consensus).

### Divulgence
When `CardOffer_ProposeSwap` does `fetch proposedCardId`, the offerer and provider (as signatories of CardOffer) learn about the Card — even though they're not observers.

### Utility Functions with Type Constraints
```daml
expireAs : (HasSignatory t, HasObserver t) => Party -> t -> Text -> Time -> Update ()
require : CanAssert m => Text -> Bool -> m ()
```
Generic helpers that work on any template with the right typeclasses.

### WORKFLOW DIAGRAMS

```
CARD ISSUANCE:
IssuerA ──create──► IssueCardProposal ──[Alice accepts]──► Card(Alice, IssuerA)

MARKET JOIN:
Alice ──create──► MarketAppInstallRequest ──[Provider accepts]──► MarketAppInstall

OFFER:
Alice ──exercise MarketAppInstall_OfferCard──► CardOffer(Alice, Provider)

PROPOSE SWAP:
Bob ──exercise CardOffer_ProposeSwap──► [lock Bob's Card] + SwapProposal(Provider,Alice,Bob)

ACCEPT SWAP:
Alice ──exercise SwapProposal_Accept──►
    [Alice's card transferred to Bob] +
    [Bob's card unlocked] +
    [Bob's card transferred to Alice]
    ──► SwapResult(aliceNewCard, bobNewCard)
```
