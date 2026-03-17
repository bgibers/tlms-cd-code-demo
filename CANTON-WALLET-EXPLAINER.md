# Canton Wallets & Payments Explainer

> A mental model for developers coming from EVM/DeFi backgrounds.

---

## Canton is NOT a public blockchain

There's no global public state, no "connect MetaMask" flow, no public key = account. Instead:

- **A Party** is a ledger identity (a string like `Alice::abc123`)
- Parties live on **validator nodes** (think: institutional participants)
- Contracts on Canton are **private by default** — only signatories/observers see them

The traditional crypto wallet concept (public key → address → holds tokens) does not exist here.

---

## How "Money" Works on Canton

Amulet/Canton Coin (CC) are **UTXO Daml contracts**, not a token balance on a shared ledger. You don't "hold CC" in a wallet address — you **are a Party that is the `owner` signatory on a set of Amulet contracts**.

In practice:
- Your "balance" = sum of your Amulet UTXO contracts
- To pay someone = exercise a choice on those UTXOs, creating new ones for the recipient
- Nobody else can see your holdings unless you explicitly disclose them

---

## The "Wallet Problem" on Canton

The challenge is: **how does a dApp interact with a user's Party without holding their signing keys?**

On Ethereum, the user signs locally in MetaMask. On Canton, signing requires access to the user's participant node credentials — which is an institutional-grade key management problem.

### What the Splice Wallet Kernel Solves

It defines **CIP-103** — a standard JSON-RPC protocol between your dApp and a Wallet Gateway:

```
Your dApp (browser)
    ↓ JSON-RPC (CIP-103)
Wallet Gateway  ← user approves transactions here
    ↓
Canton Validator Node  ← holds the party's signing key
```

The Gateway is the "MetaMask equivalent" — it brokers transaction approval between your dApp and the Canton node. The user approves in a UI, then the Gateway submits on their behalf.

Key packages:
- `@canton-network/dapp-sdk` — browser SDK for your frontend
- `@canton-network/wallet-sdk` — for wallet providers / exchanges needing lower-level control

---

## Is It More Like TradFi Deposits?

Yes — much closer to TradFi than DeFi.

| DeFi (EVM) | Canton |
|---|---|
| User holds private key in wallet | Institution/validator holds keys |
| Anyone can read balances on-chain | Holdings are private by default |
| "Approve token spend" via smart contract | Party-level authorization via Daml choices |
| Anyone deploys a contract | DARs uploaded to specific participant nodes |
| Permissionless participation | Parties onboarded by validator operators |

**Depositing to a protocol** works more like: your protocol creates a Daml choice that the user exercises, which transfers UTXO ownership. No "approve + transferFrom" like ERC-20 — the authorization is baked into the Daml template design.

---

## Summary

| Concept | Canton equivalent |
|---|---|
| Wallet | Party hosted on a validator node |
| Private key | Managed by the validator (institutional key management) |
| Token balance | Sum of Amulet UTXO contracts where you are `owner` |
| MetaMask | Splice Wallet Gateway (CIP-103) |
| ERC-20 approve + transfer | Daml choice exercised on UTXO contracts |
| Public chain state | Private contracts — need-to-know only |

**Think custodial bank account model, not permissionless DeFi.** Users interact through their validator's Wallet Gateway, and your protocol receives Daml contract ownership transfers rather than token deposits.

---

## Further Reading

- [Splice Wallet Kernel repo](https://github.com/hyperledger-labs/splice-wallet-kernel)
- [CIP-103 spec](https://github.com/hyperledger-labs/splice/blob/main/canton-improvement-proposals/CIP-103.md)
- [CIP-0056 Canton Token Standard](https://docs.global.canton.network.sync.global/app_dev/token_standard/index.html)
- [Canton / Splice App Dev Guide](https://hyperledger-labs.github.io/splice/index.html)
- [Canton docs](https://docs.digitalasset.com/build/3.4/index.html)
