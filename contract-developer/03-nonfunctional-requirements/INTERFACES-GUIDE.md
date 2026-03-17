# Daml Interfaces — Complete Guide

Interfaces are Daml's mechanism for **polymorphism**: write code once that works for any contract type implementing the interface.

---

## Why Interfaces?

### Without Interfaces (Concrete Types Only)

```daml
-- Must write separate logic for Cash and Bond
swapCash : ContractId Cash -> ContractId Cash -> Update ()
swapBond : ContractId Bond -> ContractId Bond -> Update ()

-- Adding a new asset type (Stock) requires updating every function
```

### With Interfaces (Polymorphic)

```daml
-- One function works for ANY IAsset implementor
swap : ContractId IAsset -> ContractId IAsset -> Update ()

-- Adding Stock: just implement IAsset for Stock. No other changes.
```

---

## Interface Definition Anatomy

```daml
-- Define the interface
interface IAsset where

  -- REQUIRED: viewtype declaration
  -- Specifies the data type returned by `view this`
  viewtype VAsset

  -- Interface methods: pure functions that implementors must provide
  getOwner : Party
  getIssuer : Party
  getQuantity : Decimal

  -- Interface choices: behaviors available on any IAsset
  -- These are automatically available without re-implementing in each concrete type
  nonconsuming choice SetObservers : ContractId IAsset
    with newObs : [Party]
    controller (view this).owner   -- uses viewtype to get owner
    do ...

  choice Transfer : ContractId IAsset
    with newOwner : Party
    controller (view this).owner, newOwner
    do ...
```

### viewtype

The `viewtype` is a record that summarizes the interface's "public data":

```daml
data VAsset = VAsset with
  owner   : Party
  issuer  : Party
  quantity : Decimal
    deriving (Eq, Show)
```

Inside interface choices, access it with `(view this).fieldName`:
```daml
controller (view this).owner  -- get owner from the viewtype
let owner = (view this).owner -- assign it
```

---

## Implementing an Interface

```daml
template Cash
  with
    issuer : Party
    owner  : Party
    obs    : [Party]
    quantity : Decimal
  where
    signatory [issuer, owner]
    observer obs

    -- Interface implementation block
    interface instance IAsset for Cash where
      -- Required: implement viewtype
      view = VAsset with
        owner    = this.owner
        issuer   = this.issuer
        quantity = this.quantity

      -- Required: implement each method
      getOwner    = this.owner
      getIssuer   = this.issuer
      getQuantity = this.quantity
```

### Rules for interface instances

1. Must implement the `viewtype` (via `view = ...`)
2. Must implement every declared method
3. Interface choices are inherited — you don't re-implement them
4. The concrete type can ALSO have its own (non-interface) choices

---

## Using Interface ContractIds

### Upcast: Concrete → Interface

```daml
cashCid   : ContractId Cash
iAssetCid : ContractId IAsset

-- Upcast (narrows what operations are available, but enables polymorphism)
iAssetCid = toInterfaceContractId @IAsset cashCid

-- Or with type application syntax
iAssetCid = toInterfaceContractId @IAsset cashCid
```

### Type Application Syntax

```daml
-- @TypeName specifies which type parameter to use
toInterface @IAsset cashValue           -- upcast value
toInterfaceContractId @IAsset cashCid   -- upcast ContractId
(toInterfaceContractId @ITransferProposal proposal._1)  -- inline
```

### Exercise on Interface ContractId

```daml
-- Works for ANY contract implementing IAsset
exercise iAssetCid SetObservers with newObs = [bob]

-- The ledger dispatches to the correct concrete implementation
-- (Cash.SetObservers or Bond.SetObservers etc.)
```

### Downcast: Interface → Concrete (use carefully)

```daml
-- Downcast (might fail at runtime if wrong type)
case fromInterface @Cash iAssetValue of
  Some cash -> ...  -- it was a Cash
  None      -> ...  -- it wasn't a Cash
```

---

## Interface Transfer Proposal Pattern

A common pattern is to define both `IAsset` and `ITransferProposal`:

```daml
interface ITransferProposal where
  viewtype VTransferProposal

  -- Accept the transfer proposal, returning an interface ContractId
  choice AcceptTransfer : ContractId IAsset
    controller (view this).newOwner
    do ...
```

Then `CashTransferProposal` and `BondTransferProposal` both implement `ITransferProposal`. Code that accepts proposals doesn't care which type:

```daml
-- Accept ALL transfer proposals regardless of concrete type
proposals <- query @CashTransferProposal alice  -- or BondTransferProposal
forA_ proposals \(cid, _) -> do
  exercise (toInterfaceContractId @ITransferProposal cid) AcceptTransfer
```

---

## AssetSwapProposal: Interface-Based Swap

From `Swap.daml`:

```daml
template AssetSwapProposal
  with
    requester : Party
    receiver  : Party
    offerCid  : ContractId IAsset    -- ANY IAsset implementor
    requestedSpec : (Party, Text, Decimal)  -- (issuer, assetType, quantity)
    offerSpec     : (Party, Text, Decimal)
  where
    signatory requester
    observer receiver

    choice Settle : ()
      with requestedCid : ContractId IAsset  -- ANY IAsset implementor
      controller receiver
      do
        -- Validate the provided asset matches the requested spec
        let rView = view (toInterface @IAsset (fetch requestedCid))
        assertMsg "..." (rView.issuer == requestedSpec._1)
        ...
        -- Transfer both assets (polymorphic — works for any IAsset)
        exercise offerCid Transfer with newOwner = receiver
        exercise requestedCid Transfer with newOwner = requester
```

This swap works for Cash↔Bond, Cash↔Cash, Bond↔Stock, or any future asset type — without modification.

---

## File Index

| File | Contents |
|------|----------|
| `ExtensibleSwap/Interfaces.daml` | `IAsset`, `VAsset`, `ITransferProposal` definitions |
| `ExtensibleSwap/Assets.daml` | `Cash` and `Bond` implementing `IAsset` |
| `ExtensibleSwap/Swap.daml` | `AssetSwapProposal` using `ContractId IAsset` |
| `ExtensibleSwap/SwapTestScripts.daml` | Tests: `toInterfaceContractId`, `replicateA_`, `queryFilter` |

---

## Common Mistakes

### 1. Forgetting viewtype implementation

```daml
interface instance IAsset for Cash where
  -- WRONG: forgot `view = ...`
  getOwner = this.owner

-- ERROR: "Missing viewtype implementation"
```

### 2. Using concrete ContractId when interface is expected

```daml
-- Function expects ContractId IAsset
doSwap : ContractId IAsset -> ...

cashCid : ContractId Cash

doSwap cashCid          -- WRONG: type mismatch
doSwap (toInterfaceContractId @IAsset cashCid)  -- CORRECT
```

### 3. Trying to call concrete-type choices via interface ContractId

```daml
iAssetCid : ContractId IAsset

-- Cash has a Cash-specific choice CashSpecificChoice
exercise iAssetCid CashSpecificChoice  -- WRONG: this choice isn't on IAsset

-- Must downcast first:
case fromInterfaceContractId @Cash iAssetCid of
  Some cashCid -> exercise cashCid CashSpecificChoice
  None -> ...
```
