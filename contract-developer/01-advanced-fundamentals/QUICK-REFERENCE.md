# Daml Advanced Fundamentals — Quick Reference

A cheatsheet for the most commonly needed syntax in this section.

---

## Functional Programming

### Collections

```daml
-- List
xs : [Int] = [1, 2, 3]
head xs            -- 1
tail xs            -- [2, 3]
x :: xs            -- prepend: [x, 1, 2, 3]
length xs          -- 3
elem 2 xs          -- True
notElem 5 xs       -- True
map (\x -> x*2) xs -- [2, 4, 6]
filter (>1) xs     -- [2, 3]
foldl (+) 0 xs     -- 6

-- Set (import DA.Set)
s = DA.Set.fromList [1, 2, 3, 2]  -- {1, 2, 3} (deduped)
DA.Set.member 2 s                 -- True
DA.Set.insert 4 s                 -- {1, 2, 3, 4}

-- Map (import DA.Map)
m = DA.Map.fromList [("a", 1), ("b", 2)]
DA.Map.lookup "a" m               -- Some 1
DA.Map.insert "c" 3 m             -- adds entry

-- Tuple
t : (Int, Text) = (42, "hello")
fst t   -- 42
snd t   -- "hello"
t._1    -- 42 (dot notation)
t._2    -- "hello"
```

### Conditionals

```daml
-- if-then-else (must have both branches, same type)
if x > 0 then "positive" else "non-positive"

-- Guards (in function definitions)
classify x
  | x < 0    = "negative"
  | x == 0   = "zero"
  | otherwise = "positive"

-- case-of
case maybeVal of
  None   -> "nothing"
  Some x -> "got: " <> show x

-- where clause
result = helper 42
  where helper n = n * 2
```

### Lambda

```daml
\x -> x + 1          -- lambda: increment
\x y -> x + y        -- two args
map (\x -> x * 2) [1,2,3]

-- Backtick infix
2 `elem` [1,2,3]     -- True (same as elem 2 [1,2,3])

-- Operator sections
(+) 2 3              -- 5
(2+)                 -- partial: function that adds 2
```

### Either and Optional

```daml
-- Either: Left=error, Right=success
myFunc : Int -> Either Text Int
myFunc x = if x < 0 then Left "negative" else Right (x + 1)

case myFunc 5 of
  Left err -> ...
  Right val -> ...

-- Optional: None=absent, Some=present
safeHead : [a] -> Optional a
safeHead []    = None
safeHead (x::_) = Some x

case safeHead xs of
  None   -> ...
  Some x -> ...
```

---

## Types

### Record Definition

```daml
data Person = Person with
  name : Text
  age  : Int
    deriving (Eq, Show, Ord)  -- enables ==, show, comparison

-- Construction
p : Person = Person with name = "Alice"; age = 30

-- Field access
p.name   -- "Alice"

-- Record update (creates new record)
p with age = 31
```

### Sum Type Definition

```daml
data Shape
  = Circle { radius : Decimal }
  | Rectangle { width : Decimal; height : Decimal }
    deriving (Eq, Show)

-- Pattern match
area s = case s of
  Circle{radius}   -> 3.14159 * radius * radius
  Rectangle{w, h}  -> w * h
```

### Typeclasses

```daml
-- Define
class Describable a where
  describe : a -> Text

-- Implement
instance Describable Person where
  describe p = p.name <> " age " <> show p.age

-- Use
describe myPerson   -- "Alice age 30"

-- Common built-in typeclasses
show x              -- requires Show
x == y              -- requires Eq
x < y               -- requires Ord
```

---

## Templates

### Template Skeleton

```daml
template MyContract
  with
    owner  : Party
    value  : Text
    viewers : [Party]
  where
    signatory owner
    observer viewers
    key owner : Party
    maintainer key
    ensure value /= ""

    choice UpdateValue : ContractId MyContract
      with newValue : Text
      controller owner
      do
        create this with value = newValue
```

### Choice Types

```daml
-- Consuming (default): archives contract before body
choice Foo : Bar
  controller party
  do ...

-- Nonconsuming: contract stays active
nonconsuming choice Bar : Baz
  controller party
  do ...

-- Manual archive in nonconsuming
nonconsuming choice Qux : ContractId MyContract
  controller owner
  do
    archive self
    create this with ...

-- Multi-controller (ALL must sign same tx)
choice Multi : ()
  controller [alice, bob]
  do ...
```

---

## Daml Script Testing

### Basic Script Structure

```daml
myTest = script do
  alice <- allocateParty "Alice"
  bob   <- allocateParty "Bob"

  cid <- submit alice do
    createCmd MyContract with owner = alice; value = "hi"

  result <- submit alice do
    exerciseCmd cid UpdateValue with newValue = "bye"

  -- Unhappy path
  submitMustFail bob do
    exerciseCmd cid UpdateValue with newValue = "hacked"
```

### Querying State

```daml
-- Get all contracts of type T visible to party
contracts <- query @MyContract alice
-- Returns: [(ContractId MyContract, MyContract)]

-- Get specific contract
mContract <- queryContractId alice cid
-- Returns: Optional MyContract

-- Get by key
mResult <- queryContractKey @MyContract alice myKey
-- Returns: Optional (ContractId MyContract, MyContract)

-- Filter
filtered <- queryFilter @MyContract alice (\c -> c.value /= "")
```

### Multi-Party Submit

```daml
-- Both alice and bob must authorize
submitMulti [alice, bob] [] do
  exerciseCmd cid SomeChoice with ...

-- With explicitly disclosed contracts (for privacy-preserving reads)
submitMulti [alice] [provider] do
  exerciseCmd cid ReadFromProvider with ...

-- Must fail with insufficient signatories
submitMultiMustFail [alice] [] do
  exerciseCmd cid NeedsBothParties with ...
```

### Time

```daml
now <- getTime           -- get current ledger time
passTime (minutes 5)     -- advance time by 5 minutes
passTime (hours 1)       -- advance time by 1 hour
passTime (days 1)        -- advance time by 1 day
addRelTime now (hours 2) -- add relative time to absolute time
```

---

## Exception Handling

```daml
-- try-catch
try do
  archive self
  cid <- create this with alias = newAlias
  pure (Right cid)
catch
  (PreconditionFailed _) -> do
    pure (Left "precondition failed")
  (e : AnyException) -> do
    debug $ message e
    pure (Left (message e))

-- Define custom exception
exception MyError
  with reason : Text
  where
    message "Something went wrong: " <> reason

-- Throw
throw MyError with reason = "bad input"

-- Conditional throw
when (value == "") $ throw MyError with reason = "empty value"
```

---

## Useful Imports

```daml
import Daml.Script          -- Script, submit, allocateParty, etc.
import DA.Date              -- date, Jan, Feb, ...
import DA.Time              -- hours, minutes, days, addRelTime
import DA.Set               -- Set operations
import DA.Map               -- Map operations
import DA.Action            -- replicateA_, when, unless
import DA.Foldable          -- forA_, mapA_
import DA.Assert            -- (===) operator
import DA.Exception         -- exception handling utilities
```
