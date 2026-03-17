# Daml Model View Diagrams

Visual diagrams of Daml contract lifecycles and design patterns. Open any `.excalidraw` file in [excalidraw.com](https://excalidraw.com) or VS Code with the Excalidraw extension.

## Diagrams

| File | What it shows |
|------|--------------|
| `AccountBalance-Model.excalidraw` | How Asset_Transfer splits Alice's balance and merges into Bob's — with party view bars |
| `UTXO-Model.excalidraw` | Three UTXO operations: SPLIT (1→2), MERGE (2→1), TRANSFER (via proposal) |
| `ProposeAccept-Pattern.excalidraw` | The fundamental Propose-Accept pattern with lifecycle alternatives |
| `AtomicSwap-Model.excalidraw` | How Alice and Bob exchange assets atomically — both transfers or neither |
| `SwapMarket-Workflow.excalidraw` | Full 5-step Swap Market flow with 4 parties and divulgence annotation |

## Visual Legend

| Visual | Meaning |
|--------|---------|
| Filled colored box | Active contract (on the ledger) |
| Colored outline, white fill | Contract being archived (leaving the ledger) |
| Yellow outline | Proposal/intermediate contract (create/archive in same flow) |
| Red color | Alice's contracts |
| Blue color | Bob's contracts |
| Green color | Bank/issuer contracts |
| Purple color | Provider/market operator |
| Party view bars (left) | What that party can see — contracts in their visibility zone |

## How to View

**In excalidraw.com**: Drag and drop the `.excalidraw` file onto the canvas.

**In VS Code**: Install the [Excalidraw extension](https://marketplace.visualstudio.com/items?itemName=pomdtr.excalidraw-editor) and open the file directly.

**In GitHub**: Files won't render inline, but can be downloaded and opened locally.

## Related Architecture Docs

- `contract-developer/02-auth-n-privacy/PATTERNS.md` — written explanations of these patterns
- `DAML-LEARNING-GUIDE.md` — complete concept reference
- `daml-philosophy/02-Daml-Workflows/Swap-Market-Example/ARCHITECTURE.md` — Swap Market deep dive
