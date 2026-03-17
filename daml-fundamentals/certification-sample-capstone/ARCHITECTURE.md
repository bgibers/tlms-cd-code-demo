# DamlForge — Architecture Documentation

## Overview

DamlForge is a **project proposal and approval system** implemented in Daml. It demonstrates a complete multi-step workflow between two parties: a **Proposer** (project lead) and an **Evaluator** (reviewer/approver).

## Party Roles

| Party    | Role                                                    |
|----------|---------------------------------------------------------|
| Proposer | Submits project proposals, can revise after rejection   |
| Evaluator| Reviews proposals, approves/rejects, evaluates projects |

## Contract Lifecycle

```
ProjectProposal (proposer signs)
    │
    ├──[Reject]──► ProjectProposal (with feedback in note)
    │                   │
    │               [Revise]──► ProjectProposal (new projectInfo)
    │
    └──[Approve]──► Project (evaluator + lead sign)
                        │
                        ├──[Evaluate readyToPublish=False]──► Project (new date)
                        │
                        ├──[Evaluate readyToPublish=True]──► Accomplishment
                        │
                        ├──[ChangeDates]──► Project (new dates)
                        │
                        └──[CancleProject]──► (archived)
```

## Key Daml Concepts Demonstrated

### 1. Template Anatomy
Every Daml template has: `signatory`, `observer`, `key`, `maintainer`, `ensure`, and `choice` clauses.

```daml
template ProjectProposal
  with
    proposer: Party      -- the state (immutable fields)
  where
    signatory proposer   -- who must authorize creation
    observer evaluator   -- who can see it
    key (proposer, projectInfo) : (Party, ProjectInfo)  -- uniqueness
    maintainer key._1    -- who enforces uniqueness
    ensure startDate < endDate  -- creation precondition
    choice Approve: ProjectId   -- an operation on this contract
```

### 2. Contract Keys
```daml
key (proposer, projectInfo) : (Party, ProjectInfo)
maintainer key._1
```
- Prevents a proposer from having two identical proposals active simultaneously
- Keys must be `Ord` (requires `deriving (Show, Eq, Ord)` on `ProjectInfo`)
- `maintainer` must be a subset of signatories

### 3. Choice Types
| Choice | Type | Who exercises | Effect |
|--------|------|--------------|--------|
| `Propose` | nonconsuming | proposer | Returns self (reminder) |
| `ProposerAccomplishments` | nonconsuming | evaluator | Returns list of text |
| `Reject` | consuming | evaluator | Archives + recreates with feedback |
| `Revise` | nonconsuming + manual archive | proposer | Validates, archives, recreates |
| `Cancel` | consuming | proposer | Archives (returns unit) |
| `Approve` | consuming | evaluator | Creates `Project` |
| `Evaluate` | consuming | evaluator | Returns `Either Project Accomplishment` |
| `ChangeDates` | consuming | evaluator, lead | Both must authorize |
| `CancleProject` | consuming | evaluator, lead | Both must authorize |

### 4. Custom Typeclass: Duration
```daml
class Duration i o where
    getDuration : i -> o

instance Duration ProjectInfo Int where
    getDuration proj = ... -- compute working days
```
Used to automatically calculate project duration when approving.

### 5. Either Type for Choice Results
```daml
choice Evaluate: Either (ContractId Project) (ContractId Accomplishment)
```
- `Left (ContractId Project)` = not ready, re-scheduled
- `Right (ContractId Accomplishment)` = ready, published

### 6. Derived Signatories
```daml
template Accomplishment where
    signatory (signatory project)  -- inherits Project's signatories
```
Prevents direct creation of Accomplishment — must go through `Evaluate` choice.

### 7. Either and Optional Fields
```daml
participants: Either Party [Party]    -- solo or group
budgetRequirement: Optional Decimal   -- may or may not have a budget
```

## Test Structure

```
Test.daml
├── Happy Paths
│   ├── testCreateProjectProposal  (create + ProposerAccomplishments)
│   ├── testProjectProposal        (Reject + Revise)
│   ├── testCreateProject          (Approve)
│   └── testCreateAccomplishment   (Evaluate readyToPublish=True)
└── Unhappy Paths
    ├── cantCreateWrongDates       (ensure clause: startDate > endDate)
    ├── cantApproveMyself          (proposer cannot exercise evaluator's choice)
    └── cantCreateAccomplishment   (proposer cannot directly create Accomplishment)
```

## Script Composition Pattern

```
setupTestParties
    ↓
setupProjectInfos
    ↓
setupAccomplishments
    ↓
testCreateProjectProposal
    ↓
testProjectProposal
    ↓
testCreateProject
    ↓
testCreateAccomplishment
```

Each script builds on the previous — a chain of composable setup functions.
