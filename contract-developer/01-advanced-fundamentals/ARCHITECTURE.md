# Advanced Fundamentals — Architecture Documentation

## Overview

This section covers the foundational Daml concepts needed before tackling authorization and privacy. It progressively builds from functional programming basics to template/choice mechanics to testing and exceptions.

## Module Progression

```
01-functional-programming(1)    ← Pure FP concepts (no ledger)
        ↓
01-functional-programming(2)    ← Types: records, sum types, typeclasses
        ↓
01-functional-programming(3)    ← Querying the ledger from scripts
        ↓
02-template-choice              ← Templates, choices, contract keys
        ↓
03-transaction-tree-testing     ← Daml Script test patterns
        ↓
04-exception-handling(1)        ← try-catch, PreconditionFailed
        ↓
04-exception-handling(2)        ← Custom exceptions, throw, Either returns
```

---

## Section 1: Functional Programming

### fp(1) — Core FP Operations

| File | Key Concept | What to Look For |
|------|-------------|------------------|
| `CollectionDataTypes.daml` | List, Tuple, Set, Map | `DA.Set.fromList`, `DA.Map.fromList`, operations |
| `Conditionals.daml` | if-then-else, guards, case-of | `\| condition = result` guards |
| `InformativeTypes.daml` | Either, Optional | `Left`/`Right`, `None`/`Some` |
| `Iteratives.daml` | Recursion, map, foldl | No `for` loops — only recursion and higher-order functions |
| `LambdaInfix.daml` | Lambda, backtick infix, sections | `\x -> x + 1`, `` `elem` ``, `(+) 2` |

### fp(2) — Type System Deep Dive

| File | Key Concept | What to Look For |
|------|-------------|------------------|
| `Notes.daml` | Namespace separation | `data Confusing = Int` — constructor named like a built-in type |
| `Record.daml` | Record syntax | `with` vs `{}` syntax |
| `RecordTypes.daml` | Generic records | `data Coordinate a b = Coordinate {first: a; second: b}` |
| `BuiltInVariants.daml` | Sum types from scratch | Custom Bool and Optional |
| `SumVsProduct.daml` | OR vs AND semantics | Cardinality analysis |
| `VariantTypes.daml` | Sum types with args | `MySumConstructor1 Int \| MySumConstructor2 (Text, Bool)` |
| `Typeclass.daml` | Show and Eq | `debug $ show x`, `x == y` |
| `CustomTypeclass.daml` | `class` and `instance` | Define and implement custom typeclasses |

### fp(3) — Querying Contracts

| File | Key Concept | What to Look For |
|------|-------------|------------------|
| `RetrievingContracts.daml` | `query @T`, `submitMulti`, `mapA` | How to read ledger state in scripts |

---

## Section 2: Template and Choice Mechanics

Template = contract definition. Think of it as a class that becomes an instance when created.

### Template Anatomy

```daml
template MyContract
  with
    owner: Party        ← fields (immutable state)
    value: Text
  where
    signatory owner     ← who must authorize creation/archival
    observer [viewer]   ← who can see but not create
    key owner : Party   ← uniqueness constraint
    maintainer key      ← who enforces key uniqueness
    ensure value /= ""  ← creation precondition

    choice DoSomething: ContractId MyContract  ← choice definition
      with
        newValue: Text
      controller owner
      do
        create this with value = newValue
```

### Choice Types Compared

| Type | Syntax | Contract Archived? | When? |
|------|--------|--------------------|-------|
| Consuming (default) | `choice Foo: Bar` | Yes | Before body runs |
| Nonconsuming | `nonconsuming choice Foo: Bar` | No | Never (unless body does it) |
| Preconsuming | `preconsuming choice Foo: Bar` | Yes | Before body (same as default) |
| Postconsuming | `postconsuming choice Foo: Bar` | Yes | After body runs |

**Critical gotcha with postconsuming + contract keys**: If the body creates a new contract with the same key, both would exist temporarily → key conflict → fails. Always use consuming (default) when recreating same-key contracts.

### Files in 02-template-choice

| File | Concept | Example |
|------|---------|---------|
| `TemplateStructure.daml` | Complete template skeleton | SocialNetworkUser with all clauses |
| `ChoiceTypes.daml` | Consuming choice | assertMsg, notElem, :: cons |
| `Preconsuming.daml` | preconsuming keyword | Side-by-side comparison |
| `Postconsuming.daml` | postconsuming + key conflict | Why it fails; manual archive self pattern |

---

## Section 3: Testing Patterns

### submit vs submitMustFail

```daml
-- Happy path: expects SUCCESS
cid <- submit alice do
  createCmd MyContract with owner = alice ...

-- Unhappy path: expects FAILURE (will FAIL the test if it unexpectedly succeeds)
submitMustFail alice do
  createCmd MyContract with owner = bob ...  -- alice doesn't have bob's authorization
```

### Script Composition Pattern

```
setupTestParties → setupProjectInfos → setupAccomplishments → testCreateProjectProposal ...
```

Each function calls the previous, building up state. This avoids re-creating the world for each test while keeping tests independent.

### Files in 03-transaction-tree-testing

| File | Concept |
|------|---------|
| `Setup.daml` | Template definition (separate from test) |
| `HappyUnhappy.daml` | submit vs submitMustFail pattern |
| `Modularizing.daml` | TestParties fixture, `@TestParties{..}` destructuring |
| `Testing.daml` | postconsuming choice in action |

---

## Section 4: Exception Handling

### Exception Handling Hierarchy

```
PreconditionFailed (built-in)
    ↑ thrown by ensure clause violations
    ↑ caught by try-catch

GeneralError (built-in)
    ↑ thrown by assertMsg / assert
    ↑ caught by try-catch

Custom exceptions (user-defined)
    ↑ thrown by throw MyException with ...
    ↑ caught by try-catch
    ↑ caught by AnyException handler

AnyException (catch-all)
    ↑ catches everything
```

### try-catch Structure

```daml
try do
  -- risky operations
  create this with alias = newAlias  -- might throw PreconditionFailed
catch
  (PreconditionFailed _) -> do ...   -- handle specifically
  (e: ChangeFailed) -> do ...        -- handle custom exception
  (e: AnyException) -> do ...        -- catch-all (must be last)
```

### Files in 04-exception-handling

| File | Concept |
|------|---------|
| `TryCatch.daml` | try-catch, PreconditionFailed, trace |
| `InformativeTypes.daml` (ep1) | Either/Optional as alternatives to exceptions |
| `Setup.daml` (ep1) | Intentional error: ensure clause violation |
| `CustomException.daml` | `exception` keyword, `message` clause |
| `ThrowingException.daml` | `throw`, `when condition $ action` |
| `HandlingCustomException.daml` | Multiple catch handlers, AnyException, Either return |
| `ReturningCustomException.daml` | Why `throw` can't return a value (common misconception) |

---

## Key Insights for Learners

1. **Daml has no loops** — use `map`, `foldl`, `mapA`, `forA_`, or recursion
2. **Contracts are immutable** — "update" by archiving + creating new
3. **`ensure` fires on every creation** — including inside choice bodies
4. **`postconsuming` with same key fails** — always a surprise to newcomers
5. **`throw` is terminal** — it doesn't return a value; you can't do `x <- throw ...`
6. **Either/Optional encode failure in types** — prefer these over exceptions when failure is expected
