# Non-Functional Requirements — Architecture Documentation

## Overview

This section covers advanced Daml features for building production-grade systems: **interfaces** (polymorphism), **contract upgrades** (schema evolution), and real-world design patterns for extensibility.

## Directory Structure

```
daml/
├── InitialState/           ← Baseline: Position contract (v1), setup scripts
│   ├── Cash.daml           ← Position template (no currency field)
│   └── Scripts.daml        ← initCash: parties, users, transfer proposals
│
├── Upgrade1/               ← Additive upgrade: new KYC contract type
│   ├── KYC.daml
│   └── KYCScript.daml
│
├── Upgrade2/               ← Rule contract: extend behavior without schema change
│   ├── AdditionalRules.daml
│   └── AddObserversScript.daml
│
├── Upgrade3/               ← Schema upgrade: Position gains `currency` field
│   ├── Currency.daml       ← New Position template with currency
│   └── CurrencyScript.daml ← Migration script using UpgradeModule
│
├── UpgradePackage/         ← The upgrade machinery
│   └── UpgradeModule.daml  ← UpgradeProposal, Upgrader, RunUpgrade
│
└── ExtensibleSwap/         ← Interfaces: polymorphic asset swaps
    ├── Interfaces.daml     ← IAsset, ITransferProposal interface definitions
    ├── Assets.daml         ← Cash, Bond as IAsset implementations
    ├── Swap.daml           ← AssetSwapProposal using ContractId IAsset
    └── SwapTestScripts.daml
```

---

## Part 1: Interfaces

### Why Interfaces?

Without interfaces, swap logic must be written separately for Cash and Bond:

```daml
swapCash : ContractId Cash -> ContractId Cash -> Update ()
swapBond : ContractId Bond -> ContractId Bond -> Update ()
-- ... and again for every new asset type
```

With interfaces, one implementation works for all asset types:

```daml
swapAny : ContractId IAsset -> ContractId IAsset -> Update ()
```

### Interface Anatomy

```daml
interface IAsset where
  viewtype VAsset          ← data returned by `view this`

  -- Interface methods (require implementation)
  getOwner : Party
  getIssuer : Party

  -- Interface choices (can be exercised on any IAsset)
  choice SetObservers : ContractId IAsset
    with newObs : [Party]
    controller (view this).owner
    do ...
```

### Implementing an Interface

```daml
template Cash
  with
    issuer: Party
    owner: Party
    obs: [Party]
    quantity: Decimal
  where
    signatory [issuer, owner]
    observer obs

    interface instance IAsset for Cash where
      view = VAsset with      ← required: implement the viewtype
        owner = this.owner
        issuer = this.issuer
        ...
      getOwner = this.owner   ← required: implement each method
      getIssuer = this.issuer
```

### Using Interface ContractIds

```daml
-- Upcast: Cash → IAsset
cashCid : ContractId Cash
iAssetCid : ContractId IAsset = toInterfaceContractId @IAsset cashCid

-- Exercise interface choice (works for ANY IAsset implementor)
exercise iAssetCid SetObservers with newObs = [bob]

-- Type application syntax
toInterface @IAsset cashValue  -- upcast the VALUE (not ContractId)
```

### Interface Files

| File | What it demonstrates |
|------|---------------------|
| `Interfaces.daml` | `interface`, `viewtype`, interface methods, interface choices |
| `Assets.daml` | `interface instance IAsset for Cash`, `toInterface @IAsset`, `toInterfaceContractId` |
| `Swap.daml` | `ContractId IAsset` polymorphic parameter, `view this` inside choice |
| `SwapTestScripts.daml` | `toInterfaceContractId @ITransferProposal`, `replicateA_`, `queryFilter` |

---

## Part 2: Contract Upgrades

### The Upgrade Problem

Once contracts are on a ledger, their schema is **immutable**. You cannot add a field to existing contracts. The only way to "update" them is to:
1. Archive the old contract
2. Create a new contract with the updated schema

### Upgrade Strategies (Progression)

```
Upgrade1: Add new contract types    ← Simplest: no existing contracts change
     ↓
Upgrade2: Add new behavior          ← Use "rule contracts" as behavior extensions
     ↓
Upgrade3: Schema migration          ← Migrate existing contracts to new template
```

### Upgrade1: Additive New Contract Type

No migration needed. Just add a new template (`KYC`) that references the same parties. Existing `Position` contracts are unchanged.

```
Before: [Position(Alice)] [Position(Bob)]
After:  [Position(Alice)] [Position(Bob)] [KYC(Alice, bank)]
                                          ↑ new, doesn't touch existing
```

### Upgrade2: Rule Contract (Behavior Extension)

Add a `AdditionalRules` contract that provides NEW choices operating on existing contracts. No schema change to Position needed.

```
AdditionalRules (bank creates, persists)
    └── SetObservers(owner, positionCid, new_obs)
        controller owner
        do
          archive positionCid
          create position with obs = new_obs
```

Key insight: The `AdditionalRules` contract is a "permission delegation" — bank endorses these rules by signing the contract, and owners can use it to modify their own positions within those rules.

### Upgrade3: Schema Migration

The full migration flow using `UpgradeModule`:

```
Phase 1: Propose
  Bank → UpgradeProposal(bank, alice)
  Bank → UpgradeProposal(bank, bob)

Phase 2: Accept
  Alice accepts → Upgrader(bank, alice)  [both sign]
  Bob accepts   → Upgrader(bank, bob)   [both sign]

Phase 3: Execute (bank runs for each Upgrader)
  Bank exercises RunUpgrade on Upgrader:
    forA_ positionCids \cid -> do
      Old.Position{..} <- fetch cid           ← read old data
      archive cid                             ← remove old contract
      create New.Position with               ← create with new field
        issuer; owner; obs; quantity
        currency = "USD"                     ← new field added
```

**Why two-party agreement for migration?**
Both bank AND the contract holder must consent. This protects holders from having assets silently modified. The Upgrader contract (signed by both) is the proof of consent.

### Upgrade Files

| File | Role |
|------|------|
| `InitialState/Cash.daml` | Original Position (no currency) |
| `InitialState/Scripts.daml` | Setup: parties, users, distribute positions |
| `Upgrade1/KYC.daml` | New contract type (no migration needed) |
| `Upgrade2/AdditionalRules.daml` | Rule contract for observer management |
| `Upgrade3/Currency.daml` | New Position with `currency: Text` field |
| `UpgradePackage/UpgradeModule.daml` | Migration machinery: UpgradeProposal, Upgrader, RunUpgrade |
| `Upgrade3/CurrencyScript.daml` | Full migration test using UpgradeModule |

---

## Key Concepts Quick Reference

### Interface Keywords

| Syntax | What it does |
|--------|-------------|
| `interface IFoo where` | Define interface |
| `viewtype VFoo` | Required: specify the view data type |
| `view = VFoo with ...` | Implement the viewtype in interface instance |
| `interface instance IFoo for Bar where` | Implement interface for concrete type |
| `toInterfaceContractId @IFoo cid` | Upcast ContractId to interface |
| `toInterface @IFoo value` | Upcast concrete value to interface |
| `(view this).field` | Read interface view data inside a choice |
| `ContractId IFoo` | Polymorphic contract reference |

### Upgrade Concepts

| Concept | Explanation |
|---------|-------------|
| Schema immutability | Can't modify a deployed template's fields |
| Archive + recreate | The only way to "update" a contract |
| Qualified import (`as Old`) | Reference old module to fetch/archive old contracts |
| forA_ | Apply an action to each item in a list, discarding results |
| Two-party upgrade | Both bank and holder must agree (Upgrader contract) |
| Additive upgrade | Add new types; never modify existing types |
