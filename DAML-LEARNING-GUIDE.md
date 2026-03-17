# Daml Learning Guide

A comprehensive reference for everything in this repo. Every `.daml` file has been annotated with educational comments — use this guide alongside the source files.

---

## What is Daml?

**Daml** (Digital Asset Modeling Language) is a smart contract language for building **multi-party workflows** on distributed ledgers. Key properties:

- **Privacy by design**: Parties only see contracts where they're a signatory or observer
- **Authorization model**: Every action requires explicit party authorization
- **Atomic transactions**: Either all operations in a transaction succeed or none do
- **Functional language**: Based on Haskell — pure functions, no mutation, immutable data
- **Compiled to ledger**: Runs on Canton (and other Daml-compatible ledgers)

---

## Learning Path (Suggested Order)

| Step | Location | What you'll learn |
|------|----------|-------------------|
| 1 | `daml-fundamentals/certification-sample-capstone/` | End-to-end workflow, template anatomy, choices |
| 2 | `contract-developer/01-advanced-fundamentals/` | Language fundamentals, functional programming |
| 3 | `contract-developer/02-auth-n-privacy/` | Authorization model, privacy, design patterns |
| 4 | `contract-developer/03-nonfunctional-requirements/` | Interfaces, upgrades, extensibility |
| 5 | `daml-philosophy/02-Daml-Workflows/Swap-Market-Example/` | Complete real-world system |

---

## Core Daml Concepts Reference

### Templates & Contracts

| Concept | Syntax | What it does | Example |
|---------|--------|--------------|---------|
| Template | `template Foo with ... where ...` | Defines a contract type | `capstone/daml/Main.daml` |
| Signatory | `signatory alice` | Must authorize creation; owns the contract | Every template |
| Observer | `observer bob` | Can see contract; doesn't authorize creation | `TemplateStructure.daml` |
| ensure | `ensure amount > 0.0` | Precondition; throws `PreconditionFailed` if false | `Cash.daml` |
| Contract Key | `key (issuer, owner) : (Party, Party)` | Unique identifier for this contract type | `AccountBalance.daml` |
| Maintainer | `maintainer key._1` | Who enforces key uniqueness (must be signatory subset) | Every keyed template |

### Choices

| Concept | Syntax | What it does | Example |
|---------|--------|--------------|---------|
| Consuming choice | `choice Foo: Bar` | Archives contract before body runs | `ChoiceTypes.daml` |
| Nonconsuming | `nonconsuming choice Foo: Bar` | Contract stays active | `Market.daml` |
| Preconsuming | `preconsuming choice Foo: Bar` | Explicitly consuming before body | `Preconsuming.daml` |
| Postconsuming | `postconsuming choice Foo: Bar` | Archives after body (rare) | `Postconsuming.daml` |
| Controller | `controller alice` | Who can exercise this choice | Every choice |
| Multi-controller | `controller [alice, bob]` | Both must authorize simultaneously | `Cash.daml Transfer` |
| `this` | `create this with field = val` | The current contract's data record | Most choices |
| `self` | `archive self` | The current contract's ContractId | `Postconsuming.daml` |

### Data Types

| Concept | Syntax | What it does | Example |
|---------|--------|--------------|---------|
| Record (product type) | `data Foo = Foo with field: Type` | Bundle multiple fields | `Main.daml ProjectInfo` |
| Sum type (variant) | `data Foo = A \| B x` | One of several constructors | `SumVsProduct.daml` |
| Either | `Either Text Int` | Left (error) or Right (success) | `InformativeTypes.daml` |
| Optional | `Optional Decimal` | None or Some value | `InformativeTypes.daml` |
| List | `[Party]` | Ordered, homogeneous, any length | `TemplateStructure.daml` |
| Tuple | `(Party, Text)` | Fixed-size, heterogeneous | `Main.daml` key type |
| Set | `Set Text` (from `DA.Set`) | Unique elements, unordered | `CollectionDataTypes.daml` |
| Map | `Map Int Text` (from `DA.Map`) | Key-value pairs | `CollectionDataTypes.daml` |

### Typeclasses

| Concept | Syntax | What it does | Example |
|---------|--------|--------------|---------|
| Typeclass definition | `class MyClass a where method: a -> b` | Define a shared interface | `CustomTypeclass.daml` |
| Instance | `instance MyClass Foo where method x = ...` | Implement for a concrete type | `CustomTypeclass.daml` |
| deriving | `deriving (Show, Eq, Ord)` | Auto-generate standard instances | `Main.daml ProjectInfo` |
| Show | `show x` | Convert value to Text | `Typeclass.daml` |
| Eq | `x == y`, `x /= y` | Equality comparison | Everywhere |

### Functional Programming

| Concept | Syntax | What it does | Example |
|---------|--------|--------------|---------|
| Lambda | `\x -> x + 1` | Anonymous function | `LambdaInfix.daml` |
| map | `map f xs` | Apply f to each element | `Iteratives.daml` |
| foldl | `foldl f init xs` | Reduce list to single value | `Iteratives.daml` |
| mapA | `mapA f xs` | Apply action-returning f to each | `RetrievingContracts.daml` |
| forA_ | `forA_ xs f` | Like mapA but discards results | `UpgradeModule.daml` |
| Recursion | Top-level function calls itself | Iteration without loops | `Iteratives.daml` |
| Guards | `f x \| cond = val` | Multiple conditions | `Conditionals.daml` |
| case-of | `case x of { A -> ...; B -> ... }` | Pattern matching | `Conditionals.daml` |
| where | `where helper = ...` | Local bindings | `Conditionals.daml` |
| let | `let x = val in ...` | Local bindings in expressions | Most scripts |

### Exception Handling

| Concept | Syntax | What it does | Example |
|---------|--------|--------------|---------|
| try-catch | `try do ... catch (Ex _) -> ...` | Catch exceptions and recover | `TryCatch.daml` |
| PreconditionFailed | Built-in exception | Thrown when `ensure` clause fails | `TryCatch.daml` |
| Custom exception | `exception Foo with field: Text where message "..."` | Define your own exception type | `CustomException.daml` |
| throw | `throw MyException with ...` | Immediately abort with exception | `ThrowingException.daml` |
| when | `when condition $ action` | Conditional action (throw guard) | `ThrowingException.daml` |
| AnyException | `(e: AnyException) -> ...` | Catch-all handler | `HandlingCustomException.daml` |
| message | `message e` | Get text description of any exception | `HandlingCustomException.daml` |

### Interfaces

| Concept | Syntax | What it does | Example |
|---------|--------|--------------|---------|
| Interface definition | `interface IFoo where viewtype V; method: Arg -> IFoo` | Define contract behavior contract | `Interfaces.daml` |
| viewtype | `viewtype VAsset` | Data returned by `view this` | `Interfaces.daml` |
| Interface instance | `interface instance IFoo for Bar where view = ...; method = ...` | Implement interface | `Assets.daml` |
| toInterface | `toInterface (this with f = v)` | Upcast concrete type to interface | `Assets.daml` |
| ContractId IFoo | `ContractId IAsset` | Polymorphic contract reference | `Swap.daml` |
| view | `(view this).owner` | Read interface data from inside choice | `Interfaces.daml` |
| type application | `toInterface @IAsset cash` | Specify which interface to upcast to | `Assets.daml` |

### Daml Script (Testing)

| Concept | Syntax | What it does | Example |
|---------|--------|--------------|---------|
| Script | `foo = script do ...` | A test/setup function | Every test file |
| allocateParty | `alice <- allocateParty "Alice"` | Create a party | Every test file |
| allocatePartyWithHint | `allocatePartyWithHint "Alice" (PartyIdHint "ALI")` | Create party with readable ID | `Setup.daml` |
| submit | `submit alice do createCmd ...` | Execute commands as alice | Every test |
| submitMustFail | `submitMustFail alice do ...` | Assert that the command FAILS | `HappyUnhappy.daml` |
| submitMulti | `submitMulti [alice, bob] [] do ...` | Multi-party authorization | `RetrievingContracts.daml` |
| createCmd | `createCmd Foo with ...` | Create a contract | Every test |
| exerciseCmd | `exerciseCmd cid Choice with ...` | Exercise a choice | Every test |
| query | `query @Foo alice` | Get all contracts visible to alice | `Test.daml` |
| queryContractId | `queryContractId alice cid` | Get specific contract | `Test.daml` |
| passTime | `passTime (hours 1)` | Advance ledger time | `TestCardIssuance.daml` |
| getTime | `now <- getTime` | Get current ledger time | `Market.daml` |
| debug | `debug $ someValue` | Print to IDE console (dev only) | Various files |
| trace | `trace "msg" action` | Log and run action (dev only) | `TryCatch.daml` |

---

## Daml's Authorization Model

### The Rule
Every transaction must be **authorized** by the appropriate parties. The ledger rejects unauthorized actions.

### Signatory
- Required to **create** the contract
- Required to **archive** the contract
- Required for **consuming choices**
- "Signs" the contract → takes on obligations

### Observer
- Can **see** the contract in their active contract set
- Does **NOT** authorize creation
- Cannot exercise choices unless also a controller

### Controller
- Can **exercise** the specific choice
- Authorization is **local** to that choice — doesn't grant authority over other contracts

### Non-Transitivity (Critical!)
Being authorized on contract A does NOT give you authority on contract B.
```
Bob controls ExerciseSwapOnOther on nt1
Bob exercises it → tries to call SwapSigAndObs on nt2
FAILS: Alice is the controller of SwapSigAndObs on nt2
Bob's control over nt1 does NOT cascade to nt2
```
See: `NonTransitive.daml`

---

## Daml's Privacy Model

### Sub-transaction Privacy
Parties only see the **net effects** of transactions relevant to them.

```
Alice transfers to Bob, Bob transfers to Charlie.
Carol is signatory of the Alice→Bob contract.
Carol DOES NOT see the Bob→Charlie sub-transaction.
```

### Divulgence
If a signatory exercises a choice that **fetches** contract X, all signatories of that choice's contract learn X's details.
```
offerer and provider are signatories of CardOffer.
CardOffer_ProposeSwap fetches proposedCard (Bob's card).
→ offerer and provider now know Bob's card details (divulgence).
→ IssuerA still never learns about Bob's card (privacy preserved).
```
See: `Divulgence.daml`, `Market.daml CardOffer_ProposeSwap`

### Contract Keys and Privacy
`lookupByKey @Asset (issuer, owner)` returns:
- `None` if the contract doesn't exist **OR** is not visible to the caller
- This prevents inferring what contracts exist from failed lookups

---

## Common Design Patterns

### 1. Propose-Accept
**Problem**: Two parties must agree, but can't act simultaneously.
**Solution**: Proposer creates proposal (one sig), acceptor exercises choice (adds their auth).

```
ProposeAccept.daml, Delegation.daml, CardIssuance.daml
```

### 2. Delegation
**Problem**: Agent needs to act on principal's behalf, but authorization is per-party.
**Solution**: Principal creates DelegationAgreement (signing it grants authority to agent within the agreement's scope).

```
Delegation.daml, MarketAppInstall_OfferCard
```

### 3. Atomic Swap
**Problem**: Alice gives X, Bob gives Y — must both happen or neither.
**Solution**: Both transfers in one transaction (Daml guarantees atomicity).

```
AtomicSwap.daml, SwapProposal_Accept in Market.daml
```

### 4. UTXO Model
**Problem**: Divisible assets (split, merge, transfer without residual).
**Solution**: Each asset = separate contract. Split creates two. Merge combines two into one.

```
UTXO.daml
```

### 5. Time Lock
**Problem**: Guarantee an asset's availability during negotiation without transferring ownership.
**Solution**: Lock contract held by multiple parties, expires after deadline.

```
CardIssuance.daml LockedCard, Market.daml CardOffer_ProposeSwap
```

### 6. Interfaces (Polymorphism)
**Problem**: Swap logic should work with Cash, Bond, and future asset types.
**Solution**: Define IAsset interface; all implementors can be used interchangeably.

```
ExtensibleSwap/Interfaces.daml, Assets.daml, Swap.daml
```

### 7. Contract Upgrades
**Problem**: Contracts on the ledger can't be modified in place.
**Solution**: Propose migration (Upgrader contract), fetch old, archive, create new with updated schema.

```
UpgradePackage/UpgradeModule.daml
```

---

## Model View Diagrams

Visual diagrams of contract lifecycle flows (styled like the TLMS course visuals):

| Diagram | What it shows |
|---------|--------------|
| [`diagrams/AccountBalance-Model.svg`](./diagrams/AccountBalance-Model.svg) | Asset_Transfer: splits Alice's balance, merges into Bob's |
| [`diagrams/UTXO-Model.svg`](./diagrams/UTXO-Model.svg) | SPLIT (1→2 contracts), MERGE (2→1), TRANSFER (via proposal) |
| [`diagrams/ProposeAccept-Pattern.svg`](./diagrams/ProposeAccept-Pattern.svg) | Propose-Accept lifecycle with Reject/Cancel alternatives |
| [`diagrams/AtomicSwap-Model.svg`](./diagrams/AtomicSwap-Model.svg) | Alice and Bob exchange assets atomically |
| [`diagrams/SwapMarket-Workflow.svg`](./diagrams/SwapMarket-Workflow.svg) | Full 5-step Swap Market with divulgence annotation |

Open any `.svg` in a browser or VS Code preview (right-click → Open Preview).

---

## Architecture Documentation

Each major section has dedicated architecture docs for deeper reference:

| Document | Contents |
|----------|----------|
| `daml-fundamentals/certification-sample-capstone/ARCHITECTURE.md` | Contract lifecycle, choice types table, test structure |
| `contract-developer/01-advanced-fundamentals/ARCHITECTURE.md` | Module progression, template anatomy, exception handling |
| `contract-developer/01-advanced-fundamentals/QUICK-REFERENCE.md` | **Syntax cheatsheet** — collections, types, templates, testing |
| `contract-developer/02-auth-n-privacy/ARCHITECTURE.md` | Authorization model, privacy model, pattern overview |
| `contract-developer/02-auth-n-privacy/PATTERNS.md` | Deep-dive: Propose-Accept, Delegation, Multi-party Lock, Atomic Swap |
| `contract-developer/03-nonfunctional-requirements/ARCHITECTURE.md` | Upgrade strategies, interface overview |
| `contract-developer/03-nonfunctional-requirements/INTERFACES-GUIDE.md` | **Complete interfaces guide** — definition, implementation, common mistakes |
| `daml-philosophy/02-Daml-Workflows/Swap-Market-Example/ARCHITECTURE.md` | Full swap market workflow, auth analysis, privacy analysis |

---

## Running the Examples

```bash
# Start the Daml sandbox with the navigator
daml start

# Run all tests
daml test

# Open the IDE
daml studio

# Build the project (compile .daml files)
daml build
```

Each subdirectory has its own `daml.yaml` — run these commands from within that directory.

### SDK Versions
- `daml-fundamentals/certification-sample-capstone/`: SDK **2.5.0**
- `contract-developer/01-advanced-fundamentals/` through `03-nonfunctional-requirements/`: SDK **2.5.0–2.7.x**
- `daml-philosophy/02-Daml-Workflows/Swap-Market-Example/`: SDK **2.7.3**

---

## File Index

| File | Key Concepts |
|------|-------------|
| `capstone/daml/Main.daml` | Template anatomy, Either, Optional, typeclass, contract key, multi-step workflow |
| `capstone/daml/Setup.daml` | Script composition, allocatePartyWithHint, createUser, test fixtures |
| `capstone/daml/Test.daml` | Happy/unhappy paths, submitMustFail, query @Template |
| `fp(1)/CollectionDataTypes.daml` | List, Tuple, Set, Map operations |
| `fp(1)/Conditionals.daml` | if-then-else, guards, case-of, where |
| `fp(1)/InformativeTypes.daml` | Either, Optional |
| `fp(1)/Iteratives.daml` | Recursion, map, foldl |
| `fp(1)/LambdaInfix.daml` | Lambdas, infix notation, operator sections |
| `fp(2)/BuiltInVariants.daml` | Sum types, Bool and Optional from scratch |
| `fp(2)/CustomTypeclass.daml` | class, instance, SHA256, custom Show |
| `fp(2)/SumVsProduct.daml` | Sum vs product types, cardinality |
| `fp(3)/RetrievingContracts.daml` | query @Template, submitMulti, mapA |
| `02-template-choice/TemplateStructure.daml` | Template skeleton, key, ensure |
| `02-template-choice/ChoiceTypes.daml` | Consuming choice, assertMsg, notElem, :: cons |
| `02-template-choice/Preconsuming.daml` | preconsuming keyword |
| `02-template-choice/Postconsuming.daml` | nonconsuming + manual archive self |
| `03-testing/HappyUnhappy.daml` | submit vs submitMustFail |
| `03-testing/Modularizing.daml` | TestParties, @TestParties{..}, script composition |
| `04-exceptions(1)/TryCatch.daml` | try-catch, PreconditionFailed, trace |
| `04-exceptions(2)/CustomException.daml` | exception keyword, message clause |
| `04-exceptions(2)/ThrowingException.daml` | throw, when |
| `04-exceptions(2)/HandlingCustomException.daml` | Multiple catch handlers, AnyException, Either return |
| `01-auth/AuthDemo.daml` | Mutual controllers, fetch, assert, PaymentChannel |
| `01-auth/NonTransitive.daml` | Non-transitivity of authorization |
| `02-patterns/ProposeAccept.daml` | Propose-Accept pattern |
| `02-patterns/Delegation.daml` | Delegation pattern |
| `03-privacy/AccountBalance.daml` | lookupByKey, balance model, privacy |
| `03-privacy/UTXO.daml` | UTXO model, split, merge, SplitResult |
| `03-privacy/AtomicSwap.daml` | Atomic swap, observer list, validation |
| `03-privacy/Divulgence.daml` | Divulgence via fetch |
| `ExtensibleSwap/Interfaces.daml` | interface, viewtype, interface methods, interface choices |
| `ExtensibleSwap/Assets.daml` | interface instance, toInterface, @TypeApp |
| `ExtensibleSwap/Swap.daml` | ContractId IAsset, polymorphic dispatch |
| `UpgradePackage/UpgradeModule.daml` | Contract migration, forA_, qualified import |
| `SwapMarket/CardIssuance.daml` | TimeLock, multi-party lock, getTime, stakeholder |
| `SwapMarket/Market.daml` | 3-party signatory, nonconsuming offers, divulgence, atomic settlement, require utility |
