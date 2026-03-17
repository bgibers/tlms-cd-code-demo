# Authorization & Privacy — Architecture Documentation

## Overview

This section covers Daml's core security properties: **who can do what** (authorization) and **who can see what** (privacy). These aren't optional features — they're foundational guarantees enforced by the ledger itself.

## Section Structure

```
01-authorization/   ← Who can perform actions; multi-party auth; non-transitivity
02-design-patterns/ ← Propose-Accept and Delegation patterns
03-privacy/         ← Sub-transaction privacy, UTXO model, atomic swaps, divulgence
```

---

## Section 1: Authorization

### The Authorization Model

Every Daml action requires explicit authorization from the relevant parties. The ledger REJECTS unauthorized actions — there's no way to bypass this in contract code.

```
Action Type      | Required Authorization
-----------------|----------------------------------------------
Create contract  | All signatories must authorize
Archive contract | All signatories must authorize
Exercise choice  | Controller(s) must authorize
                 | + all signatories of the hosting contract
```

### Key Concepts

**Signatory**: Must authorize contract creation. "Signs" the contract = takes on obligations. Can always see the contract.

**Observer**: Can see the contract. Cannot authorize creation. Useful for giving visibility without requiring consent.

**Controller**: Can exercise a specific choice. Authorization is LOCAL to that choice.

**Multi-controller**: `controller alice, bob` — BOTH must authorize in the SAME transaction.

### Non-Transitivity (Critical Security Property)

Being authorized on contract A does NOT give you authority on contract B.

```
Setup: Alice has nt1 (she's signatory), Bob controls ExerciseSwapOnOther on nt1

Bob exercises ExerciseSwapOnOther on nt1.
Inside: tries to call SwapSigAndObs on nt2 (where ALICE is the controller).

RESULT: FAILS — Bob's authority over nt1 does NOT extend to nt2.
Bob can only do what he's explicitly authorized to do on each specific contract.
```

See: `NonTransitive.daml`

### Files

| File | Concept |
|------|---------|
| `AuthDemo.daml` | `fetch`, `assert`, mutual controllers, PaymentChannel |
| `NonTransitive.daml` | Non-transitivity of authorization — why cascading auth fails |

---

## Section 2: Design Patterns

### Pattern 1: Propose-Accept

**Problem**: Two parties must agree to something, but they can't submit simultaneously.

**Solution**:
1. Party A creates a Proposal (only A must sign)
2. Proposal sits on ledger — visible to Party B
3. Party B exercises Accept choice → creates the final contract (both A + B sign)
4. Or Party B exercises Reject → proposal archived

```
ProposalContract (signatory=A, observer=B)
    ├── [Accept] by B → FinalContract (signatory=A,B) ✓
    ├── [Reject] by B → archived (both informed)
    └── [Cancel] by A → archived (A withdraws)
```

**Why it works**: A's signing of the Proposal counts as A's authorization for the final contract. When B accepts, B adds their authorization. Together = both parties agreed.

### Pattern 2: Delegation

**Problem**: An agent needs to act on behalf of a principal, but the ledger tracks authorization per-party.

**Solution**: Principal creates a DelegationAgreement (principal signs it). This agreement grants the agent authority within a specific scope.

```
DelegationAgreement (signatory=principal, observer=agent)
    └── [ActOnBehalf] by agent → uses principal's authority
```

**Security property**: The agent can ONLY act within the scope of the agreement. They can't act as the principal in other contexts.

### Files

| File | Concept |
|------|---------|
| `ProposeAccept.daml` | Full Propose-Accept with lifecycle |
| `Delegation.daml` | DelegationAgreement, principal-agent pattern |

---

## Section 3: Privacy

### Daml's Privacy Model

Privacy in Daml is **structural**, not optional. Parties only see contracts where they're a stakeholder (signatory or observer). This isn't encryption — it's ledger-level access control via sub-transaction privacy.

### Sub-Transaction Privacy

```
Transaction T:
  sub-transaction A: Alice transfers to Bob
    - Carol is signatory of Alice's contract
    - Carol SEES this sub-transaction
  sub-transaction B: Bob transfers to Charlie
    - Carol is NOT involved in Bob→Charlie
    - Carol DOES NOT see sub-transaction B

Result: Carol knows Alice→Bob happened, but NOT Bob→Charlie
```

### Privacy Patterns Compared

| Model | Key Feature | Files |
|-------|-------------|-------|
| AccountBalance | Contract key `(issuer, owner)` — one balance per issuer/owner pair | `AccountBalance.daml` |
| UTXO | Each unit = separate contract; split/merge creates new contracts | `UTXO.daml` |
| AtomicSwap | UTXO + SwapProposal; both transfers in one transaction | `AtomicSwap.daml` |
| Divulgence | `fetch` inside a choice reveals contract to all authorizers | `Divulgence.daml` |

### Divulgence Deep Dive

```daml
template DivulgenceTest
  with owner: Party; witness: Party
  where
    signatory owner
    observer witness

    nonconsuming choice FetchContract: Asset
      controller witness
      do
        fetch assetCid  -- ← THIS CAUSES DIVULGENCE
                        -- witness authorized this choice
                        -- fetch reveals assetCid to witness
```

**Before divulgence**: `queryContractId witness assetCid` → `None`
**After divulgence**: `queryContractId witness assetCid` → `Some asset`

### Contract Keys and Privacy

`lookupByKey @Asset (issuer, owner)` returns:
- `None` if contract doesn't exist **OR** is not visible to the caller
- This prevents inferring which contracts exist from failed lookups

### AtomicSwap Authorization Analysis

```
Asset: signatory = issuer, owner (both must sign all operations)

Asset_Transfer:    controller = owner
                   → creates AssetTransferProposal (recipient is observer)

AssetTransferProposal_Accept: controller = recipient
                              → creates new Asset with owner = recipient

SwapProposal_Accept: controller = recipient
                     → exercises both Asset_Transfer choices + accepts both proposals
                     → ALL in one transaction (atomic)
```

### Test Files

| File | What It Tests |
|------|---------------|
| `Test_AccountBalance.daml` | queryContractKey, partial transfers, chained transfers |
| `Test_UTXO.daml` | merge, split, SplitResult, transfer lifecycle |
| `Test_AtomicSwap.daml` | UTXO lifecycle + atomic swap with SetObservers |
| `Test_Divulgence.daml` | Before/after divulgence: None → Some after fetch |
| `Setup.daml` | TestParties fixture (alice, bob, ub=US_Bank, eb=EU_Bank) |

---

## Security Checklist

When reviewing authorization in Daml code, ask:

- [ ] Who are the signatories? Do they ALL need to consent?
- [ ] Who is the controller of each choice? Is it the right party?
- [ ] Can a party exercise a choice they shouldn't? (check `submitMustFail` tests)
- [ ] Does any choice `fetch` a contract? Who learns about it? (divulgence)
- [ ] Are contract keys used? Can someone infer existence from `lookupByKey`?
- [ ] Is delegation needed? Use DelegationAgreement, not hacks.
