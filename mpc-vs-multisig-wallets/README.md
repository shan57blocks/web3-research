# MPC Wallets vs Multisig Wallets

Research notes, November 2026.

Contents:

1. [Basics: MPC vs multisig](#1-basics-mpc-vs-multisig)
2. [The Stellar ecosystem](#2-the-stellar-ecosystem)
3. [OpenZeppelin Smart Account on Stellar, in depth](#3-openzeppelin-smart-account-on-stellar-in-depth)
4. [OpenZeppelin Smart Account vs Safe](#4-openzeppelin-smart-account-vs-safe)
5. [Open questions](#5-open-questions)
6. [Sources](#6-sources)

---

## 1. Basics: MPC vs multisig

### Background: what a wallet key is

A crypto wallet is controlled by a **private key**, a long secret number. Whoever has the key can move the money. If you lose it, the money is gone. If someone steals it, the money is theirs.

Both wallet types fix the same weakness: **one key is one point of failure**. They fix it in different places.

### Multisig wallet ("multi-signature")

**The idea:** there are several separate, complete keys. A rule says how many must agree before money can move, for example "any 2 of these 3 keys".

**How it works:**

1. Alice, Bob and Carol each hold their own full key.
2. Alice approves a payment with her key.
3. Bob approves the same payment with his key.
4. The blockchain checks that 2 of the 3 approved, then lets the money move.

**The key point:** the blockchain itself enforces the rule. Everyone can see that this is a 2-of-3 wallet and which keys signed.

- On Bitcoin, multisig is built into the system.
- On Ethereum it is a **smart contract** (a small program stored on the blockchain). The best-known one is **Safe**, formerly called Gnosis Safe.

### MPC wallet ("multi-party computation")

**The idea:** there is only one key, but it never exists in one place. It is created as separate pieces called **shares**, and each party holds one share. No single share can sign anything.

**How it works:**

1. Your phone, the company's server and a backup server each hold a share.
2. To sign, 2 of them run a shared math process. They talk to each other, but no one ever sends their share.
3. The output is **one normal signature**, the same kind a regular wallet produces.
4. The blockchain sees an ordinary single-key wallet and has no idea several parties were involved.

**The key point:** the rule is enforced off the blockchain, by the software and the math.

Examples: Fireblocks, ZenGo, Coinbase's wallet tools. You may also see the term **TSS** ("threshold signature scheme"). It is the specific MPC technique used for signing.

### The differences side by side

| | **Multisig** | **MPC** |
|---|---|---|
| Number of keys | Several full keys | One key, split into shares that are never joined |
| Who enforces the rule | The blockchain | The software, off the blockchain |
| What the public sees | The "2 of 3" setup and who signed | A normal single wallet (more private) |
| Which blockchains | Only ones that support it, with a separate version for each chain | Almost any chain, because the signature looks normal |
| Fees | Higher. Ethereum: the contract must be set up first, and each payment costs more. Bitcoin: transactions are larger. | Same as a normal wallet |
| Changing who can sign | A transaction on the blockchain, which costs a fee and is public | Shares are re-split privately. The wallet address stays the same. |
| How easy it is to check | Very easy. The code is open and has protected billions of dollars for years. | Harder. The math is complex, and vendor code is often closed. |
| Typical risk | Bugs in the smart contract | Bugs in the cryptography code |
| Depending on one company | Low. Safe is open and you can use it without the company. | Often high. Leaving the provider can be hard. |

### Real failures that show the risks

- **Parity, November 6, 2017 (multisig):** many Parity multisig wallets relied on one shared code library. A user took control of that library and deleted it, which froze about 513,774 ETH in more than 500 wallets permanently. Nobody stole the money, but nobody can move it either.
- **BitForge and TSSHOCK, 2023 (MPC):** security researchers (Fireblocks for BitForge, Verichains for TSSHOCK) found flaws in several widely used MPC code libraries. In some setups an attacker who controlled just one party could rebuild the full key. BitForge affected more than 15 wallet providers and projects, including Coinbase, ZenGo and Binance. No theft from these flaws was reported.
- **Bybit, February 21, 2025 (multisig):** about 401,000 ETH (roughly $1.4–1.5 billion) was stolen from a Safe wallet. The smart contract was not broken. Attackers put malicious code into the Safe website two days earlier, so the signers were shown a harmless-looking payment while actually approving the theft.
  - **Lesson:** both wallet types are only as safe as the screen signers trust. If people approve without seeing what they are actually signing, splitting the key does not protect them.

### When to choose which

- **Multisig** fits a project treasury, a DAO (a group that runs a project through on-chain votes), or anyone who wants rules that are public and checkable, and who mostly uses one chain such as Ethereum.
- **MPC** fits an exchange or company moving money across many chains, a consumer app where users should not have to manage seed phrases (the backup word lists for a key), or cases where low fees and privacy matter.
- **Both together** is common at large firms: an MPC wallet acts as one of the signers inside a multisig.

> Section 1 was first written from general knowledge. The three incidents above were later checked against web sources (see section 6).

---

## 2. The Stellar ecosystem

### How Stellar differs

On most blockchains, multisig is an add-on. On Ethereum it needs a smart contract, such as Safe.

**On Stellar, multisig is built into every account.** No extra program is needed. Stellar has three ways to share control of a wallet:

1. **Built-in multisig** on normal accounts
2. **Smart accounts**, which are wallets written as Soroban programs. Soroban is Stellar's smart-contract system. These are closest to Ethereum's Safe.
3. **MPC wallets**, where one key is split into shares and the blockchain sees a normal wallet.

### 2.1 Built-in multisig

**How it works:**

- Every Stellar account starts with one key, called the **master key**, with a "weight" of 1.
- You can add more keys (**signers**) and give each one a weight.
- You set three **thresholds**, each a number from 0 to 255. A transaction goes through only if the weights of the keys that signed it add up to the threshold:
  - **Low** for small housekeeping actions
  - **Medium** for payments
  - **High** for changing the signers or thresholds themselves
- Example: three keys with weight 1 each and a medium threshold of 2 means "2 of 3 must approve any payment."

**Other signer types** (also built in):

- A pre-approved future transaction
- A secret code: anyone who knows it can sign, and it becomes public once used

**Limits and costs:**

- At most 20 extra signers per account, and at most 20 signatures per transaction.
- Each extra signer locks 0.5 XLM in the account as a required minimum balance.
- If extra, unneeded signatures are attached, the transaction fails.
- Lockout risk: if you set the master key's weight to 0 without other working signers, the account can be locked forever.
- Everyone can see the signer setup.
- Rules can only be "enough weight approved." There is nothing like "max $100 per day."

**Tools people use:**

- **LOBSTR Vault:** a phone app that acts as a co-signer. It is open source.
- **Refractor:** a service from the StellarExpert team that stores a pending transaction while people add their signatures.
- **BitGo:** a regulated custody company (one that holds crypto for clients), offering multisig on Stellar since October 2018.

### 2.2 Smart accounts (Soroban)

The wallet is a program on the blockchain. Its address starts with **C**, while normal accounts start with **G**.

**What the program can enforce:**

- Signers can be normal keys, **passkeys** (the fingerprint or face login on your phone), or other accounts.
- Rules can be scoped to specific actions, for example "this app may withdraw 100 USDC per month for one year."
- Built-in rule types: threshold multisig, weighted threshold, spending limits.

**Current tools:**

- **OpenZeppelin Smart Accounts**, the recommended choice, used through the **smart-account-kit** SDK. See section 3.
- **Passkey Kit** is the older option. It is now marked "legacy."
- **Crossmint** also has a Stellar smart account.

### 2.3 MPC wallets

**Why it works on Stellar:** Stellar signs with a method called **Ed25519**. Several parties holding shares can jointly produce a normal Ed25519 signature, so the blockchain sees an ordinary G-address account. Nothing needs to be set up on-chain.

**Providers found:**

| Provider | What I found |
|---|---|
| **Fireblocks** | Supports Stellar wallets. The customer must deposit 1 XLM to activate each new wallet. A Fystack blog says Fireblocks stopped paying this deposit for customers in January 2024. |
| **Dfns** | Made Stellar and Soroban a top-level supported chain in May 2024. |
| **Cobo** | Stellar wallet built on MPC, aimed at AI agents. |
| **Portal** | Docs show sending XLM signed with an Ed25519 share. |
| **Fystack** | Open-source 2-of-3 Stellar MPC wallet, September 2026. Its own post says it is **unaudited and testnet only** (testnet is Stellar's practice network). |
| **Turnkey** | Lists Stellar, but it is **not MPC**: it keeps the whole key inside a sealed secure hardware zone. |
| **Privy** | Reported to support Stellar, but it splits keys differently (Shamir's secret sharing combined with secure hardware). Only found in a third-party source. |
| **Coinbase** | The Coinbase Wallet help page reportedly lists Stellar as unsupported (seen in search results; the page itself blocked direct reading). |

### 2.4 How the three compare on Stellar

| | Built-in multisig | Smart account | MPC |
|---|---|---|---|
| Who enforces the rules | The Stellar network | A program on Stellar | The provider's software, off-chain |
| Public can see the setup | Yes | Yes | No. It looks like a normal wallet. |
| Rule types | Only "enough weight signed" | Rich: limits, per-action rules, passkeys | Whatever the provider offers |
| Extra cost | 0.5 XLM locked per signer | Program setup and running fees | None on-chain. The provider charges a fee. |
| Main risk | Lockout from wrong weights | Bugs in the program | Bugs in the crypto code, and depending on one company |
| Maturity | Oldest and simplest | Newest and still changing | Mature at big vendors. Fystack is early. |

### 2.5 Stellar-specific points that affect any wallet

- **The company that issues a token can control it.** Issuers can choose who may hold their token, freeze it, or take it back. A wallet's balance can change without any of its signers approving. This applies to all three types.
- **Validators may be able to freeze entries.** One vendor blog (Fystack) says that since May 2026 the network's validators (the computers that run Stellar) can freeze specific ledger entries in an emergency, using a feature called "Quorum Freeze." Not confirmed anywhere else.

### 2.6 Bottom line for Stellar

- **Simple shared control** (a team treasury, a cold-storage backup): built-in multisig with LOBSTR Vault or Refractor.
- **Spending limits, passkeys, or per-app permissions:** a smart account built on OpenZeppelin.
- **An institution or app needing privacy and one setup across many chains:** an MPC provider such as Fireblocks or Dfns.

---

## 3. OpenZeppelin Smart Account on Stellar, in depth

### What it is

A **smart account** is a wallet that is itself a small program on Stellar. When something tries to spend from it, Stellar asks the program "is this allowed?" and the program checks its own rules.

**OpenZeppelin** is a well-known security company. It writes a ready-made, reusable version of this wallet program. The code package is called `stellar-accounts`, and the current version is **0.8.0**. A companion TypeScript toolkit for building apps is **smart-account-kit**, published by the Stellar organization itself.

### The three building blocks

Think of a company bank account with a rulebook.

#### Signers: who can approve

- **Delegated signer:** any Stellar address (a normal G account or another smart account). It approves in its own usual way.
- **External signer:** a raw key plus a **verifier**, a separate shared program that checks that kind of signature.
  - Built-in verifiers: **Ed25519** (normal Stellar keys) and **WebAuthn** (passkeys).
  - A verifier is deployed once and shared by everyone. Supporting a new kind of key later only means pointing to a new verifier, so the wallet never has to be moved.

#### Context rules: what each group may do

Each rule says "**for this kind of action**, **these signers** must approve, under **these policies**, **until this date**."

The kinds of action:

- **Default:** anything. Meant for admin control.
- **CallContract:** only actions on one specific app. Example: "this game may act on my behalf."
- **CreateContract:** only deploying one specific program.

Examples from OpenZeppelin's docs:

- "This app key may withdraw 100 USDC per month for one year"
- "2 of 3 normal keys can deploy a contract"
- "3 of 3 passkeys can vote on one specific contract"

**Important design choice:** the person submitting the transaction must **name which rule** they are using. The wallet does not search for a matching rule. This stops an attacker from quietly switching to a weaker rule.

#### Policies: extra conditions on a rule

| Policy | What it does | Watch out |
|---|---|---|
| **Simple threshold** | N of the rule's signers must approve | — |
| **Weighted threshold** | Each signer has a weight. Approved when the weights add up to the target. Example: "1 admin, OR 2 managers, OR 3 staff." | Changes must happen in the right order or they fail |
| **Spending limit** | Caps total transfers of one token over a rolling time window. Example: 5 USDC per day. | Remembers at most 815 recent transfers. When that list is full, new transfers fail until old ones age out. |

You can also write your own policies.

#### Limits

- Up to **15 signers** per rule
- Up to **5 policies** per rule
- **No limit** on the number of rules
- Rule names: at most 20 bytes

### How a payment gets approved, step by step

1. An app builds a transaction and says "use rule #3."
2. The signers sign a fingerprint of the transaction. Since 0.7.0 that fingerprint includes the chosen rule numbers, and since 0.8.0 it also includes the wallet's address, so a signature cannot be reused for another rule or another wallet.
3. The wallet checks that rule #3 exists, has not expired, and fits this kind of action.
4. It checks each signature, calling the verifier for passkeys and keys.
5. It runs each policy in order. If any one says no, the whole transaction is cancelled.

### Traps to avoid (from OpenZeppelin's own warnings)

1. **You can lock yourself out forever.** Changing the wallet's own settings needs a valid rule. If you delete or let expire the last rule that covers this, the wallet can never be changed again.
2. **Thresholds do not update themselves.**
   - Remove 2 signers from a "5 of 5" and you can never reach 5 again. That action is blocked forever.
   - Add 2 signers to a "3 of 3" and it silently becomes "3 of 5," which is weaker.
   - Fix: adjust the threshold in the same transaction as the signer change.
3. **No delegation loops.** A wallet must not list itself as its own signer. Version 0.8.0 now blocks the known loops, which in some setups could have let the wallet be approved without any real key.
4. **Version mismatch.** Apps built for 0.7.x cannot sign for a 0.8.0 wallet. Match the toolkit to the wallet version.

### Security review status

**The audit.** OpenZeppelin's own security team reviewed version 0.7.0 from March 2 to March 19, 2026, and published the report on April 28, 2026.

- Results: **21 findings**, made up of 0 critical, 1 high, 6 medium, 9 low and 5 informational. 18 are fixed.
- The **high** finding was about the smart account. The rule number was not covered by the signatures, so whoever submitted the transaction could swap in a weaker rule after people had signed, for example one without the spending limit. It is fixed.
- One **medium** finding is "acknowledged, not fixed": keys are compared as raw bytes, so the same key written two different ways may not be counted.

**Gaps a third-party team found** while reviewing the audits (a GitHub pull request from qomiralabs):

- Every audit is OpenZeppelin checking **its own** code, so none is independent.
- The audits cover the library, **not the example wallet** that people actually deploy.
- 61 lines of code, including the delegation-loop fix, changed **after** the last audit.
- The public security advisory for that fix (GHSA-jpg9-m7w4-c7wc) could not be found.

**Bug bounty.** OpenZeppelin runs a bug bounty on Immunefi: up to $25,000 for critical bugs, live since May 2025. Not confirmed whether the smart account code is covered.

**smart-account-kit** labels itself **"unaudited integration software."** It tells users not to keep money in it that they cannot afford to lose.

### How mature is it?

- **Being built on:** yes, actively. Projects found include AI-agent wallets and passkey wallets, but the ones found are **testnet or alpha**.
- **Real production use on the main network:** **no named wallet** found running it with real money.
- **Version churn:** 0.6.0 (January 2026), 0.7.0 (April 2026) and 0.8.0 (October 20, 2026), each with breaking changes. OpenZeppelin itself says to pin an exact version.

### Compared with Stellar's built-in multisig

| | Built-in multisig | OpenZeppelin smart account |
|---|---|---|
| Signers | Normal keys only | Normal keys, passkeys, other wallets, custom key types |
| Rules | Only "enough weight approved," at 3 levels | Per app, per action, time limits, spending caps, custom rules |
| Signer limit | 20 extra signers per account, 20 signatures per transaction | 15 per rule, unlimited rules |
| Cost | 0.5 XLM locked per extra signer | Program setup and storage fees. Fee sponsoring is possible. |
| Maturity | Years in production | New. Pre-1.0, testnet-heavy. |
| Main risk | Wrong weights lock you out | Program bugs, wrong configuration, version mismatch |

### Bottom line

- **Good for:** passkey login (no seed phrase to write down), app "session keys" with limits, and AI agents with daily spending caps.
- **Not yet ready for:** large amounts of money without your own independent audit of the exact version you deploy.

---

## 4. OpenZeppelin Smart Account vs Safe

### The short version

Both are wallets that are programs on the blockchain, and both can require "several people must approve." They start from opposite designs:

- **Safe** begins with **one list of owners and one number**, for example "3 of 5 owners." Everything else, such as spending limits or automation, is bolted on as an add-on.
- **OpenZeppelin** begins with **many separate rules**. Each rule has its own approvers and conditions, for example "the CFO alone may pay up to $1,000 per day, but changing the wallet needs 3 of 5 directors."

The other big difference is **maturity**. Safe holds tens of billions of dollars and has years of outside audits. OpenZeppelin's smart account is new, below version 1.0, and mostly used on testnet.

### How Safe works, in plain words

- **Owners plus threshold:** a list of owners and one number. "3 of 5" applies to everything the wallet does.
- **Modules:** add-on programs that are allowed to move money **without the owners' signatures**. The spending-limit feature (the "Allowance module") is one of these. They are powerful, and dangerous if a bad one is added.
- **Guards:** checkpoint programs that inspect each transaction and can block it. Until version 1.5.0, **modules could go around guards**. Version 1.5.0 (released July 2025) added "module guards" to close that gap.
- **Fallback handler:** a helper that answers requests the core wallet doesn't understand.
- **Account abstraction:** an add-on module (the "4337 module") lets someone else pay the fees and supports passkey login.
- **Passkeys:** done through separate signer contracts, because a Safe owner must be an address and a passkey is not one. Ethereum's main network can check passkey signatures cheaply since its December 2025 "Fusaka" upgrade, but other networks vary.

### Side by side

| | **Safe** | **OpenZeppelin Smart Account** |
|---|---|---|
| Chain | Ethereum and 300+ similar networks | Stellar (Soroban) |
| Core idea | One "M of N owners" rule for everything | Many scoped rules, each with its own approvers and conditions |
| Different rules for different actions | Only through add-on modules or guards | Built in: a rule can cover all actions, one specific app, or deploying one specific program |
| Weighted votes ("CEO counts as 2") | Not built in. All owners count equally. | Built in (weighted threshold) |
| Spending limits | Allowance module: per token, per delegate, refills over time | Spending-limit policy: per token, rolling time window |
| Rules that expire | Not built in. Needs a module. | Built in: every rule can have an end date |
| Passkey login | Via extra signer contracts and the 4337 module | Built in through a passkey verifier |
| Someone else pays the fees | Needs the 4337 module and an outside service that bundles and submits transactions | Stellar asks the wallet directly. A relayer service submits the transaction and can pay the fees. |
| Biggest power risk | A module skips owner signatures entirely | Wrong rule setup (lockout, out-of-date thresholds) |
| Upgrades | Owners can switch the account to a new Safe version | The account can upgrade itself if its signers approve |
| Same address on many chains | Yes | Stellar only |
| Fees | Ethereum main network can be expensive. Other networks are cheap. | Stellar fees are tiny |

### Maturity and trust

| | **Safe** | **OpenZeppelin Smart Account** |
|---|---|---|
| Live since | 2018–2019, first as Gnosis Safe (Safe's site says 2019) | 2025–2026, version 0.8.0 |
| Usage | 63M+ accounts, $27B+ held, 1.1B+ transactions (Q2 2026) | No named wallet with real money on the main network found |
| Audits | "Over 10" plus formal verification (mathematical proof that key code is correct). Version 1.5.0 was audited by Certora and Ackee, two outside firms. | OpenZeppelin auditing **its own** code. The 0.7.0 audit had 1 high finding (fixed). Code changed after the last audit. |
| Breaking changes | Rare. The core design has been stable for years. | Three releases with breaking changes in about ten months (January to October 2026) |
| Known large incident | **Bybit, February 2025, about $1.4–1.5B.** Attackers changed the Safe website, so signers approved a malicious transaction they thought was routine. The wallet code itself was not broken. | None found |

### "If you know Safe, here is the equivalent"

| Safe | OpenZeppelin |
|---|---|
| Owners plus "3 of 5" | A rule covering all actions, with 5 signers and a "3 of N" policy |
| Allowance module (spending limit) | A spending-limit policy on a rule scoped to one token |
| Guard | A policy (but it only applies to the rules you attach it to) |
| Module (a bot or app that acts for the wallet) | A rule scoped to that app, with its own key and limits |
| 4337 module plus passkey signer | Native passkey verifier plus a fee-paying relayer |

**One real safety gain on the OpenZeppelin side:** there is no "module that skips signatures." **Every** action has to pass through a rule, with signers and policies checked. Safe needed version 1.5.0 just to stop modules from going around guards.

**One real weakness on the OpenZeppelin side:** more freedom means more ways to set it up wrong, for example:

- deleting the last rule that covers admin changes, which locks the wallet forever
- removing signers without lowering the threshold, which blocks that action forever

### When to choose which

- **On Ethereum or similar chains:** Safe. It is the standard and battle-tested.
- **On Stellar with simple team approvals and real money:** Stellar's built-in multisig is still the safer bet today.
- **On Stellar, needing passkeys, per-app permissions, spending caps, or AI-agent limits:** the OpenZeppelin smart account is the best design available. Pin an exact version and get an independent audit before holding significant money.

---

## 5. Open questions

Things not verified during this research:

- Whether Fireblocks offers Stellar-specific MPC features beyond basic wallets.
- Whether Turnkey and Privy fully support Stellar (address format, transaction building), beyond curve-level signing.
- The "Quorum Freeze" validator feature (May 2026). Only one vendor source mentions it.
- The OpenZeppelin smart account audit reports' full text in the repo's `audits/` folder.
- Whether the Immunefi bug bounty covers the smart account code.
- The Fireblocks January 2024 reserve change and the Coinbase "unsupported" listing: both official pages blocked direct reading.
- The general Safe design (modules skipping signatures, upgrade path, same address across chains) comes from prior knowledge and was only spot-checked.
- Several facts come from vendor marketing, mainly Fystack's blog, which is a competitor writing about the others.

---

## 6. Sources

### Incidents

- [Parity: 513,774 ETH frozen (DN Institute)](https://hjklvfr.github.io/dn-institute/attacks/posts/2017-11-06-Parity)
- [Parity: TechCrunch, December 2017](https://techcrunch.com/2017/12/05/parity-ceo-says-shes-confident-that-its-280m-in-frozen-ethereum-isnt-lost-forever)
- [BitForge and TSShock (Threshold Network)](https://www.threshold.network/blog/bitforge-and-tsshock)
- [TSSHOCK (Verichains)](https://verichains.io/tsshock/)
- [BitForge (The Record)](https://therecord.media/multiple-vulnerabilities-affecting-crypto-platform-bitforge)
- [Bybit incident technical analysis (CertiK)](https://www.certik.com/resources/blog/bybit-incident-technical-analysis)
- [Bybit hack analysis (NCC Group)](https://nccgroup.com/dk/research-blog/in-depth-technical-analysis-of-the-bybit-hack)

### Stellar built-in multisig

- [Stellar docs: Signatures and Multisig](https://developers.stellar.org/docs/learn/fundamentals/transactions/signatures-multisig)
- [Stellar docs: Accounts (base reserve, subentries)](https://developers.stellar.org/docs/learn/fundamentals/stellar-data-structures/accounts)
- [Stellar XDR: MAX_SIGNERS = 20](https://github.com/stellar/stellar-xdr/blob/curr/Stellar-ledger-entries.x)
- [BitGo launches support for Stellar](https://bitgo.com/resources/blog/bitgo-launches-support-for-stellar)
- [LOBSTR multisig intro](https://lobstr.freshdesk.com/support/solutions/articles/151000001275-introduction-to-multisignature-concept)
- [Refractor (SCF public goods)](https://scf-public-goods-maintenance.github.io/projects/refractorspace/)

### Stellar MPC providers

- [Fystack: MPC wallet on Stellar](https://fystack.io/blog/mpc-wallet-on-stellar)
- [fystack/stellar-wallet on GitHub](https://github.com/fystack/stellar-wallet)
- [Fystack: Stellar custody in 2026](https://fystack.io/blog/stellar-custody-in-2026-ecosystem-risks-and-custody-providers-compared)
- [Fireblocks support: Stellar base reserve](https://support.fireblocks.io/hc/en-us/articles/11096124643484)
- [Dfns Stellar tier 1 announcement](https://x.com/dfnsHQ/status/1793241628968144961)
- [Cobo Stellar wallet](https://www.cobo.com/assets/stellar-agentic-wallet)
- [Portal: Sign Stellar Transaction](https://docs.portalhq.io/guides/enclave-mpc-api/sign-stellar-transaction)
- [Turnkey: other networks](https://docs.turnkey.com/networks/others.md)
- [Crossmint agent wallets compared](https://www.crossmint.com/learn/agent-wallets-compared)
- [Coinbase Wallet supported networks](https://help.coinbase.com/wallet/browser-extension/supported-networks-and-assets)

### OpenZeppelin Smart Account

- [OpenZeppelin docs: Smart Account](https://docs.openzeppelin.com/stellar-contracts/accounts/smart-account)
- [Context Rules](https://docs.openzeppelin.com/stellar-contracts/accounts/context-rules)
- [Signers and Verifiers](https://docs.openzeppelin.com/stellar-contracts/accounts/signers-and-verifiers)
- [Policies](https://docs.openzeppelin.com/stellar-contracts/accounts/policies)
- [stellar-contracts accounts package (GitHub)](https://github.com/OpenZeppelin/stellar-contracts/tree/main/packages/accounts)
- [v0.8.0 release notes](https://github.com/OpenZeppelin/stellar-contracts/releases/tag/v0.8.0)
- [Stellar Contracts RC v0.7.0 Audit](https://www.openzeppelin.com/news/stellar-contracts-rc-v0.7.0-audit)
- [Stellar Contracts Library 0.1.0 Audit](https://www.openzeppelin.com/news/stellar-contracts-library-0.1.0-audit)
- [qomiralabs audit-coverage review PR #161](https://github.com/qomiralabs/ujira-contracts/pull/161)
- [OpenZeppelin on Stellar bug bounty (Immunefi)](https://immunefi.com/bug-bounty/openzeppelin-stellar/information/)
- [stellar/smart-account-kit](https://github.com/stellar/smart-account-kit)
- [Passkey Kit (legacy)](https://socket.dev/npm/package/passkey-kit)
- [Stellar docs: Smart wallets](https://developers.stellar.org/docs/build/guides/contract-accounts/smart-wallets)
- [Stellar docs: OpenZeppelin contracts](https://developers.stellar.org/docs/tools/openzeppelin-contracts)
- [Stellar protocol meeting, Apr 30 2026 (CAP-71 delegation)](https://developers.stellar.org/meetings/2026/04/30)
- [Crossmint Stellar smart account](https://awesome.ecosyste.ms/projects/github.com%2Fcrossmint%2Fstellar-smart-account)
- [Example: stellar-agent-wallet (testnet)](https://github.com/ArbitrageScanner/stellar-agent-wallet)
- [Example: pennyops stellar-agent-wallet (alpha)](https://github.com/pennyops/stellar-agent-wallet)

### Safe

- [Safe v1.5.0: Module Guards (Safe Foundation)](https://safefoundation.org/blog/introducing-safe-v1-5-0-module-guards-enhanced-smart-account-features)
- [Safe contracts v1.5.0 in Safe{Wallet}](https://safe.global/blog/safe-contract-version-1-5-0)
- [safe-smart-account releases](https://github.com/safe-fndn/safe-smart-account/releases)
- [Safe Smart Contracts: audits and stats](https://safefoundation.org/smart-contracts)
- [Safe Q2 2026 report](https://safefoundation.org/blog/safe-q2-2026-quarterly-report)
- [Safe Q1 2026 report](https://safefoundation.org/blog/safe-q1-2026-quarterly-report)
- [The Block: Safe nearly 130M transactions in Q2](https://theblock.co/post/410023/safe-smart-accounts-process-nearly-130-million-transactions-in-record-quarter)
- [Safe passkeys with Safe{Core} SDK](https://docs.safe.global/home/passkeys-guides/safe-sdk)
- [Safe Passkeys Signer](https://docs.safe.global/sdk/signers/passkeys)
- [Safe spending limits help](https://help.safe.global/en/articles/40842-set-up-and-use-spending-limits)
- [ERC-7579 vs Safe Modules](https://crypto.training/blog/2026-02-19-smart-account-modules/)
- [Ethereum Fusaka mainnet announcement (passkey precompile, EIP-7951)](https://blog.ethereum.org/2025/11/06/fusaka-mainnet-announcement)
