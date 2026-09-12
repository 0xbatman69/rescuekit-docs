# RescueKit — Authoritative Protocol & Technical Documentation

> **Document Status:** Authoritative & Production-Verified  
> **Protocol Version:** 2.4.0 (EIP-7702 / ERC-7821 Batch Execution Standard)  
> **Target Audience:** Security Researchers, Smart Contract Engineers, Protocol Integrators, Node Operators, and End Users  
> **Verification Basis:** Live Blockchain State, Contract Bytecode (`solc 0.8.37`), Frontend Service Bundles, and Official EVM Standards  

---

## Table of Contents

1. [Introduction & Executive Overview](#1-introduction--executive-overview)
   - 1.1 The MEV Sweeper Bot Threat Model
   - 1.2 The EIP-7702 Paradigm Shift
   - 1.3 Core Protocol Principles & Trust Boundaries
2. [Quick Start & Operational Workflow](#2-quick-start--operational-workflow)
   - 2.1 Prerequisites & Operational Requirements
   - 2.2 End-to-End Execution Sequence
   - 2.3 Post-Recovery Account Sanitation
3. [Core Technical Architecture & EIP-7702 Mechanics](#3-core-technical-architecture--eip-7702-mechanics)
   - 3.1 EIP-7702 Type-4 Transaction Anatomy
   - 3.2 Ephemeral In-Transaction Delegation
   - 3.3 ERC-7821 Execution Standard (`execute(bytes32,bytes)`)
   - 3.4 In-Lock Transient Storage & Reentrancy Guards
4. [Rescue Products Catalog](#4-rescue-products-catalog)
   - 4.1 Token & Native Asset Rescue (Mode 1 & Mode 2)
   - 4.2 ERC-721 & ERC-1155 NFT Rescue (Mode 3)
   - 4.3 Airdrop, Vesting & Staking Claims Rescue (Mode 6 & Mode 9)
   - 4.4 DeFi Lending Collateral Recovery (Mode 4 Flash Loans)
   - 4.5 NFT Mint & Sweep Recovery (Mode 7 & Mode 8)
5. [Authoritative Network & Deployment Directory](#5-authoritative-network--deployment-directory)
   - 5.1 The 19-Chain Mainnet Deployment Matrix
   - 5.2 Deterministic CREATE2 Deployment (`0x0000000004C9B572E8aB03C7A7377AaadEfd3502`)
   - 5.3 Per-Chain RPC, Gas Pricing & Mempool Topologies
6. [Supported Assets & Protocol Integrations](#6-supported-assets--protocol-integrations)
   - 6.1 Token Standard Compatibility (ERC-20, Rebasing, Fee-on-Transfer)
   - 6.2 NFT Standard Compatibility (ERC-721, ERC-1155, Soulbound Rejections)
   - 6.3 Integrated Lending Protocols (Aave v3, Morpho Blue, Moonwell, Curvance)
   - 6.4 Flash Loan Providers & Routing Priority (Balancer v3, Morpho, Uniswap v4, Aave v3)
   - 6.5 On-Chain DEX Swaps & Liquidity Routing
7. [Protocol Economics & Mathematical Fee Specification](#7-protocol-economics--mathematical-fee-specification)
   - 7.1 Protocol Fee Structure (1500 BPS / 15.00%)
   - 7.2 Affiliate Referral Split (600 BPS / 6.00%)
   - 7.3 Mathematical Implementation & Safe Division
   - 7.4 Zero-Fee Exemptions & Asset Exclusions
   - 7.5 Asset Valuation Dynamics & Oracle Independence
8. [Transaction Construction, Gas Dynamics & Type-4 Lifecycle](#8-transaction-construction-gas-dynamics--type-4-lifecycle)
   - 8.1 The Two-Component Gas Equation (Gas Units vs. Gwei Price)
   - 8.2 Analytical Gas-Unit Fallback Formulas (`rescueGasFallback`)
   - 8.3 Live Fee Pricing Multipliers & Network Spikes
   - 8.4 Receipt Polling, Confirmation Timeouts & Nonce Coordination
9. [Security Architecture, Threat Vectors & Trust Boundaries](#9-security-architecture-threat-vectors--trust-boundaries)
   - 9.1 Private Key Isolation & Client-Side Ephemeral Signing
   - 9.2 Cryptographic Replay Protection & Chain ID Domain Separation
   - 9.3 Frontrunning, Mempool Leakage & Private Relay Routing
   - 9.4 What RescueKit Protects Against vs. What It Does Not Protect Against
10. [Sponsor Wallet Architecture & Operational Mechanics](#10-sponsor-wallet-architecture--operational-mechanics)
    - 10.1 Sponsor Wallet Definition & Isolation
    - 10.2 Sponsor Funding Requirements & Calculations
    - 10.3 Unused Gas Balance Reclamation
11. [Referral & Affiliate Commission System](#11-referral--affiliate-commission-system)
    - 11.1 On-Chain Affiliate Attribution
    - 11.2 Self-Referral Prevention Logic
    - 11.3 Gas-Stipended Transfer Isolation (50,000 Gas Guard)
    - 11.4 Fallback Protocol Routing upon Affiliate Reversion
12. [REST API Specification & Execution Models](#12-rest-api-specification--execution-models)
    - 12.1 `/api/build` — Calldata & Batch Construction
    - 12.2 `/api/execute` — Server-Side Execution Pipeline
    - 12.3 `/api/broadcast` — Direct Raw Transaction Relay
    - 12.4 Rate Limiting & Denial-of-Service Mitigations
13. [Smart Contracts & Technical Interface Reference](#13-smart-contracts--technical-interface-reference)
    - 13.1 `SponsorableBatchExecutor` Contract Specification
    - 13.2 Execution Modes & Bitmask Flags
    - 13.3 Contract Administrative Controls & Upgradability
    - 13.4 Interface Identifiers (`supportsInterface`)
14. [User Input Field Reference & Validation Matrix](#14-user-input-field-reference--validation-matrix)
    - 14.1 Compromised Wallet Address & Key
    - 14.2 Safe Destination Address
    - 14.3 Sponsor Wallet Address & Key
    - 14.4 Contract Addresses, Token IDs & Call Data
15. [Error Reference, Contract Reverts & Failure Resolutions](#15-error-reference-contract-reverts--failure-resolutions)
    - 15.1 Smart Contract Custom Revert Errors
    - 15.2 Client-Side Validation & Simulation Rejections
    - 15.3 EVM JSON-RPC & Broadcast Rejections
16. [The "What Happens If..." Real-World Edge Case Directory](#16-the-what-happens-if-real-world-edge-case-directory)
    - 16.1 Asset & State Alterations
    - 16.2 Transaction Execution & Network Failures
    - 16.3 Protocol & Financial Edge Cases
17. [Architectural Comparisons & Industry Alternatives](#17-architectural-comparisons--industry-alternatives)
    - 17.1 RescueKit vs. Flashbots Private Bundles
    - 17.2 RescueKit vs. ERC-4337 Account Abstraction
    - 17.3 RescueKit vs. Direct Account Gas Funding
18. [Comprehensive FAQ](#18-comprehensive-faq)
19. [Protocol Limitations & Known Issues](#19-protocol-limitations--known-issues)
20. [Technical Glossary](#20-technical-glossary)
21. [Documentation Fact & Verification Ledger](#21-documentation-fact--verification-ledger)
22. [Version History & Protocol Changelog](#22-version-history--protocol-changelog)

---

## 1. Introduction & Executive Overview

RescueKit is a mission-critical asset recovery protocol engineered specifically for compromised Ethereum Virtual Machine (EVM) accounts. By harnessing Ethereum Improvement Proposal 7702 (EIP-7702), RescueKit enables users to recover tokens, non-fungible tokens (NFTs), unclaimed airdrops, vesting releases, and collateral locked in decentralized finance (DeFi) lending markets without ever funding the compromised account with native gas.

### 1.1 The MEV Sweeper Bot Threat Model

When an account's private key or seed phrase leaks, automated MEV sweeper bots monitor the address across public transaction mempools and block builders. Sweeper bots maintain persistent RPC subscriptions listening for inbound transfers:

```
[Attacker Bot] ──(Listens)──> Compromised Account
      │
[User sends 0.01 ETH Gas] ──> Compromised Account
      │
[Attacker Bot detects 0.01 ETH in Mempool/Block]
      │
[Attacker Bot frontruns with 10x gas] ──> Sweeps 0.01 ETH to Attacker Wallet
      │
[Result] Compromised Account Balance = 0 ETH. Trapped Assets remain stuck.
```

Under this hostile condition, traditional transfer transactions (`eth_sendRawTransaction`) fail because the account owner cannot fund the account with the native gas token required to sign and broadcast an ERC-20 `transfer` or ERC-721 `transferFrom`. Any gas sent to the address is stolen within milliseconds by the bot.

### 1.2 The EIP-7702 Paradigm Shift

EIP-7702 introduces an ephemeral smart contract delegation standard for Externally Owned Accounts (EOAs). Instead of altering account ownership or deploying an expensive proxy contract, EIP-7702 permits an EOA to sign an authorization tuple designating an external smart contract implementation (`0x0000000004C9B572E8aB03C7A7377AaadEfd3502`) to execute code on behalf of the EOA for the duration of a single transaction.

Crucially, EIP-7702 transactions can be sponsored by an independent, uncompromised account (the "Sponsor Wallet"). The sponsor wallet pays 100% of the native gas required to broadcast and execute the transaction. The compromised account never receives, holds, or spends native gas tokens, completely starving the sweeper bot of an exploitation vector.

### 1.3 Core Protocol Principles & Trust Boundaries

RescueKit is governed by five absolute technical rules:

1. **Zero Compromised Gas Funding:** Under no operational circumstance is native gas deposited into the compromised account.
2. **Transaction-Level Atomicity:** Approvals, claim executions, flash loans, debt repayments, asset transfers, fee distributions, and implementation delegations occur atomically in one single EVM transaction. If any critical sub-step fails, the entire transaction reverts, ensuring no stranded allowances or half-executed states.
3. **Client-Side Key Isolation:** Private keys entered into the browser client are executed strictly in ephemeral memory within the local browser context via Web Workers or cryptographic libraries. Private keys are never logged, serialized, or transmitted to any remote backend server.
4. **Autonomous Fee Deduction:** Protocol recovery fees are deducted on-chain directly from the recovered assets in the same atomic transaction. The user never prepays protocol fees out of pocket.
5. **No Absolute Security Guarantees:** Recovery viability is constrained by blockchain state, transaction inclusion speed, mempool architecture, and whether the sweeper bot has already executed on-chain liquidation or transfer calls.

---

## 2. Quick Start & Operational Workflow

### 2.1 Prerequisites & Operational Requirements

To execute an asset recovery via RescueKit, the operator must prepare:

- **Compromised Account Key:** The private key of the compromised EOA (required to cryptographically sign the EIP-7702 authorization tuple).
- **Clean Safe Destination Address:** A newly generated, uncompromised wallet address (hardware wallet, fresh EOA, or Safe multisig) where recovered assets will be sent.
- **Clean Sponsor Wallet:** An uncompromised EOA with a small native gas balance on the target network to broadcast and sponsor the transaction.

### 2.2 End-to-End Execution Sequence

```
1. Operator Connects Sponsor Wallet (Pays Gas)
2. Operator Inputs Safe Destination Address (Receives Assets)
3. Operator Inputs Compromised Account Private Key (Client-Side Only)
4. Scanner Queries RPC Multicall3 for Balances, NFTs, Claims, and DeFi Debts
5. Operator Selects Assets to Recover
6. Builder Constructs ERC-7821 Batch Calldata
7. Client Requests EIP-7702 Authorization Signature from Compromised Key
8. Sponsor Signs & Broadcasts Type-4 Transaction via Public RPC or Relay
9. Executor Contract Atomically Sweeps Assets to Safe Destination (85% Net)
10. Protocol Fee (15%) and Affiliate Commission (6%) Settled On-Chain
```

### 2.3 Post-Recovery Account Sanitation

Because an EIP-7702 authorization temporarily delegates an EOA's code execution to `SponsorableBatchExecutor`, the delegation naturally expires or remains pointing to the executor contract. 

- **State Independence:** Even if the delegation remains active, the compromised account cannot be exploited through RescueKit because every call requires an authorized cryptographic digest or signature matching the sender context.
- **Undelegation (Optional):** If desired, the compromised account can sign an EIP-7702 authorization tuple designating `address(0)` as the implementation address, clearing any delegated bytecode upon execution.

---

## 3. Core Technical Architecture & EIP-7702 Mechanics

### 3.1 EIP-7702 Type-4 Transaction Anatomy

An EIP-7702 transaction is designated in the EVM as transaction type `0x04`. It introduces a novel field known as the `authorizationList`:

```
TransactionType: 0x04
Fields:
  - chainId: uint256
  - nonce: uint256 (Sponsor Nonce)
  - maxPriorityFeePerGas: uint256
  - maxFeePerGas: uint256
  - gasLimit: uint256
  - to: address (Compromised Account)
  - value: uint256 (0)
  - data: bytes (ERC-7821 execute calldata)
  - accessList: AccessList
  - authorizationList: [
      {
        chainId: uint256,
        address: 0x0000000004C9B572E8aB03C7A7377AaadEfd3502,
        nonce: uint256 (Compromised Account Nonce),
        yParity: uint8,
        r: bytes32,
        s: bytes32
      }
    ]
  - sponsorSignature (v, r, s signed over Type-4 payload)
```

### 3.2 Ephemeral In-Transaction Delegation

When the EVM processes an EIP-7702 transaction:

1. **Validation:** The EVM validates that the authorization tuple's `(r, s, yParity)` signature recovers to the compromised account address, that `chainId` matches the active chain (or is `0`), and that `nonce` matches the compromised account's on-chain transaction count.
2. **Bytecode Delegation:** The EVM prepends designated delegation bytecode (`0xef0100 || implementationAddress`) to the compromised account's state account record.
3. **Execution Context:** The transaction's `to` address is the compromised account. The call executes against the compromised account's storage and address context (`address(this) == compromisedAccount`), but executes the bytecode defined by `SponsorableBatchExecutor`.
4. **Caller Semantics:** The `msg.sender` inside the initial call frame is the Sponsor Wallet.

### 3.3 ERC-7821 Execution Standard (`execute(bytes32,bytes)`)

RescueKit strictly implements the ERC-7821 Minimal Batch Executor interface:

```solidity
function execute(bytes32 mode, bytes calldata executionData) external payable;
function supportsExecutionMode(bytes32 mode) external pure returns (bool);
```

The `mode` parameter is a 32-byte bitmask that instructs the executor how to parse and process `executionData`.

```
ERC-7821 Mode Layout (32 bytes):
Byte 0: Call Type (0x01 = Single Batch Call)
Bytes 1-5: Reserved / Flags (0x0000000000)
Bytes 6-7: Standard Identifier (0x7821)
Bytes 8-9: Protocol Sub-Mode (0x0001 = OpData, 0x0003 = NFT, 0x0004 = FlashLoan, 0x0006 = Claim, 0x0007 = Mint721, 0x0008 = Mint1155, 0x0009 = MultiClaim)
Bytes 10-31: Unused / Zero Padding
```

### 3.4 In-Lock Transient Storage & Reentrancy Guards

The protocol leverages EIP-1153 transient storage (`TLOAD` / `TSTORE`) where available, and deterministic storage slots when compiling for legacy target chains, to manage state during complex flash loan callbacks:

- **Slot `0x4d696e742d666f72776172642d6163746976652d763100...`:** Active NFT mint context flag.
- **Slot `0x466c6173682d4c6f616e2d436f6e746578742d763100...`:** Active flash loan validation context flag.
- **Transient Reset:** Storage slots are asserted upon entry and unconditionally cleared prior to transaction completion, preventing cross-call reentrancy and storage contamination.

---

## 4. Rescue Products Catalog

RescueKit delivers five specialized recovery tools, each optimized for specific asset types and smart contract states.

### 4.1 Token & Native Asset Rescue (Mode 1 & Mode 2)

- **Target Assets:** Native gas tokens (ETH, BNB, POL, MON, S, BERA, etc.) and standard ERC-20 tokens.
- **ERC-7821 Mode:** 
  - `MODE_1` (`0x0100000000000000000000000000000000000000000000000000000000000000`)
  - `MODE_2` (`0x0100000000007821000100000000000000000000000000000000000000000000`)
- **Execution Mechanism:**
  1. For each ERC-20 token, queries balance via `balanceOf(address(this))`.
  2. Calculates fee: `feeAmount = (balance * 1500) / 10000` (15%).
  3. Sends `sendAmount = balance - feeAmount` (85%) directly to `safeDestination`.
  4. Distributes `feeAmount` between protocol treasury (9% or 15%) and affiliate referrer (6% if active).
  5. For native tokens, checks account balance above `nativeReserveBalance`. Splits net balance and transfers native currency.

### 4.2 ERC-721 & ERC-1155 NFT Rescue (Mode 3)

- **Target Assets:** Existing non-fungible tokens sitting in the compromised wallet.
- **ERC-7821 Mode:** `MODE_3` (`0x0100000000007821000300000000000000000000000000000000000000000000`)
- **Execution Mechanism:**
  1. Unpacks token calls containing `transferFrom(compromisedAddress, safeDestination, tokenId)` or `safeTransferFrom(...)`.
  2. Executes transfers sequentially in a single loop.
  3. **Zero-Fee Guarantee:** No protocol fee is deducted from NFTs. Every NFT is transferred 100% intact to `safeDestination`.

### 4.3 Airdrop, Vesting & Staking Claims Rescue (Mode 6 & Mode 9)

- **Target Assets:** Unclaimed tokens locked in Merkle distributors, vesting contracts (Sablier, Hedgey), or staking reward pools.
- **ERC-7821 Mode:** 
  - `MODE_6` (Direct single-claim contracts)
  - `MODE_9` (Multi-claim batches across distinct protocols)
- **Execution Mechanism:**
  1. Calls the target distributor contract with the user's Merkle proof or claim calldata.
  2. The claimed tokens land in `address(this)` (the compromised account).
  3. Within the exact same execution frame, executes an immediate sweep call transferring net tokens to `safeDestination` and settling the protocol fee.
  4. Sweeper bots cannot frontrun the claimed balance because the claim and sweep happen in the same atomic block step.

### 4.4 DeFi Lending Collateral Recovery (Mode 4 Flash Loans)

- **Target Assets:** Collateral locked in DeFi lending positions where debt prevents direct withdrawal.
- **ERC-7821 Mode:** `MODE_4` (`0x0100000000007821000400000000000000000000000000000000000000000000`)
- **Execution Mechanism:**
  1. Borrows outstanding debt tokens via uncollateralized flash loan (Balancer v3, Morpho Blue, Uniswap v4, or Aave v3).
  2. Approves lending protocol and executes `repay()` on behalf of the compromised account.
  3. Executes `withdraw()` or `redeem()` to release 100% of deposited collateral.
  4. If collateral matches debt asset: repays flash loan principal plus premium directly from recovered collateral.
  5. If collateral differs from debt asset: routes collateral through an in-lock Uniswap v3/v4 swap to acquire the exact repayment amount.
  6. Sweeps remaining surplus collateral to `safeDestination` (85% net surplus).

### 4.5 NFT Mint & Sweep Recovery (Mode 7 & Mode 8)

- **Target Assets:** High-value or allowlisted NFT mints allocated to the compromised wallet.
- **ERC-7821 Mode:** 
  - `MODE_7` (ERC-721 Mint & Forward)
  - `MODE_8` (ERC-1155 Mint & Forward)
- **Execution Mechanism:**
  1. Sets active mint context storage slot with `safeDestination` and `nftCollection`.
  2. Invokes external mint function on NFT contract.
  3. NFT contract calls `onERC721Received` or `onERC1155Received` on the compromised account.
  4. The executor intercepts the hook and immediately redirects (`safeTransferFrom`) the freshly minted token ID to `safeDestination`.
  5. Clears mint context slot.

---

## 5. Authoritative Network & Deployment Directory

### 5.1 The 19-Chain Mainnet Deployment Matrix

The following matrix represents the verified production deployments across all 19 supported EVM mainnets. All deployments are live mainnets; testnets are strictly excluded.

| # | Network | Chain ID | Native Gas Token | Contract Address | Public RPC Endpoint | 7702 Readiness | Status |
|---|---|---|---|---|---|---|---|
| 1 | **Ethereum** | 1 | ETH | `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | `https://eth.drpc.org` | Live | Production |
| 2 | **Base** | 8453 | ETH | `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | `https://mainnet.base.org` | Live | Production |
| 3 | **BNB Chain** | 56 | BNB | `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | `https://binance.nodereal.io` | Live | Production |
| 4 | **Arbitrum One** | 42161 | ETH | `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | `https://arb1.arbitrum.io/rpc` | Live | Production |
| 5 | **Polygon** | 137 | POL | `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | `https://polygon-bor-rpc.publicnode.com` | Live | Production |
| 6 | **Optimism** | 10 | ETH | `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | `https://mainnet.optimism.io` | Live | Production |
| 7 | **Monad** | 143 | MON | `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | `https://rpc.monad.xyz` | Live | Production |
| 8 | **Sonic** | 146 | S | `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | `https://sonic.drpc.org` | Live | Production |
| 9 | **Robinhood** | 4663 | ETH | `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | `https://rpc.mainnet.chain.robinhood.com` | Live | Production |
| 10 | **Berachain** | 80094 | BERA | `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | `https://rpc.berachain.com` | Live | Production |
| 11 | **MegaETH** | 4326 | ETH | `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | `https://mainnet.megaeth.com/rpc` | Live | Production |
| 12 | **Linea** | 59144 | ETH | `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | `https://rpc.linea.build` | Live | Production |
| 13 | **Ink** | 57073 | ETH | `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | `https://rpc-qnd.inkonchain.com` | Live | Production |
| 14 | **Unichain** | 130 | ETH | `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | `https://mainnet.unichain.org` | Live | Production |
| 15 | **Sei** | 1329 | SEI | `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | `https://evm-rpc.sei-apis.com` | Live | Production |
| 16 | **World Chain** | 480 | ETH | `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | `https://worldchain-mainnet.g.alchemy.com/public` | Live | Production |
| 17 | **Somnia** | 5031 | SOMI | `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | `https://api.infra.mainnet.somnia.network` | Live | Production |
| 18 | **Plasma** | 9745 | XPL | `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | `https://plasma.gateway.tenderly.co` | Live | Production |
| 19 | **Plume** | 98866 | PLUME | `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | `https://rpc.plume.org` | Live | Production |

### 5.2 Deterministic CREATE2 Deployment (`0x0000000004C9B572E8aB03C7A7377AaadEfd3502`)

All 19 networks share an identical contract address:
`0x0000000004C9B572E8aB03C7A7377AaadEfd3502`

- **9-Zero Vanity Prefix:** Mined via optimized CREATE2 salt derivation to save calldata gas during EIP-7702 authorization list inclusion.
- **Immutability:** Deployed via immutable factory bytecode using `solc 0.8.37`.
- **Global Parameter:** Standard across all EVM environments, preventing cross-chain address confusion.

### 5.3 Per-Chain RPC, Gas Pricing & Mempool Topologies

```
Network Typologies:
1. Sequencer L2s (Base, Optimism, Arbitrum, Linea, Ink, Unichain, World Chain, Plume):
   - Single sequencer; no public mempool competition.
   - Low tip multiplier (1x-2x), 3x base fee buffer for spike absorption.
2. Parallel EVM / High-Throughput (Monad, Sonic, MegaETH, Sei, Somnia):
   - Monad: Gas war ordering via tip bid (5x tip multiplier), bills on gasLimit. Minimum reserve: 10.5 MON.
3. Legacy / Public Mempool Chains (BNB Chain, Polygon):
   - Public transaction pool where bots observe unconfirmed transactions.
   - Requires 5x priority tip bids or private relay broadcast.
```

---

## 6. Supported Assets & Protocol Integrations

### 6.1 Token Standard Compatibility

- **Standard ERC-20:** Fully supported. Balances are queried and swept atomically.
- **Rebasing Tokens (e.g., stETH, AMPL):** Supported. The protocol queries `balanceOf(address(this))` immediately at execution time, accurately capturing the current balance.
- **Fee-on-Transfer Tokens:** Supported with safety buffers. RescueKit measures actual balance transferred rather than assuming nominal amounts.

### 6.2 NFT Standard Compatibility

- **ERC-721:** Fully supported across single transfers and batches. 0% protocol fee.
- **ERC-1155:** Fully supported for both single token IDs and batch transfers (`safeBatchTransferFrom`).
- **Soulbound / Non-Transferable Tokens:** **Not Supported.** Any attempt to execute a transfer on a non-transferable token (e.g., ENS Name Wrapper under specific fuses, soulbound badges) will cause the transaction to revert.

### 6.3 Integrated Lending Protocols

RescueKit integrates native liquidation and withdrawal adapters for major lending protocols:

| Protocol | Version | Supported Chains | Architecture |
|---|---|---|---|
| **Aave** | v3 | Ethereum, Base, Polygon, Arbitrum, Optimism, BSC, Monad, Sonic, Linea, Plasma | Pool / DataProvider Collateral Withdrawal |
| **Morpho** | Blue | Ethereum, Base, Monad | Market-based isolated lending pools |
| **Moonwell** | Core | Base, Optimism | Compound v2 style cToken redemption |
| **Curvance** | v1 | Monad | High-performance modular money market |

### 6.4 Flash Loan Providers & Routing Priority

When recovering collateral from active loans, the protocol dynamically routes flash loan requests based on liquidity and borrowing fees:

1. **Balancer v3:** 0% Fee (0 BPS). Primary provider on Ethereum, Base, Arbitrum, Optimism, Sonic.
2. **Morpho Blue:** 0% Fee (0 BPS). Secondary provider for single-asset flash loans.
3. **Uniswap v4:** 0% Flash Fee (In-lock flash swaps via PoolManager unlock callback).
4. **Aave v3:** 0.05% Fee (5 BPS). Fallback provider with deep multi-asset liquidity pools across 10 chains.

### 6.5 On-Chain DEX Swaps & Liquidity Routing

When rescued collateral differs from the flash-borrowed debt token, RescueKit executes an in-transaction swap to satisfy flash loan repayment:

- **Verified Routers:** Configured across 8 DEX chains with verified on-chain Uniswap v3/v4 Router, Quoter, and Factory contracts.
- **Exact Output Swapping:** Calls `swapExactTokensForTokens` with precise output targets matching flash loan principal plus accrued fee.
- **Slippage Bounds:** Hardcoded maximum slippage threshold (1.00% to 2.50%) to prevent sandwich attack MEV exploitation inside the block.

---

## 7. Protocol Economics & Mathematical Fee Specification

### 7.1 Protocol Fee Structure (1500 BPS / 15.00%)

RescueKit operates on a performance-based fee model deducted exclusively upon successful asset recovery:

- **Protocol Fee Rate:** `1500` basis points (`15.00%`).
- **User Net Recovery:** `8500` basis points (`85.00%`).
- **Fee Basis:** Calculated directly against the gross balance of recovered tokens and native assets.
- **Treasury Address:** `0x2735fAfD319155d8F7548d4FD68222cDbE808F39`

### 7.2 Affiliate Referral Split (600 BPS / 6.00%)

RescueKit incorporates an on-chain affiliate revenue-sharing mechanism:

- **Affiliate Commission:** `600` basis points (`6.00%` of gross recovered asset).
- **Treasury Net Cut:** `900` basis points (`9.00%` of gross recovered asset).
- **No Referrer Present:** If no valid referrer address is specified, 100% of the protocol fee (`1500` BPS / `15.00%`) routes to the protocol treasury.

```
Total Recovered Asset (100% / 10,000 BPS)
      │
      ├───> 85.00% (8,500 BPS) ──> Safe Destination Wallet (User)
      │
      └───> 15.00% (1,500 BPS) Total Protocol Fee
                 │
                 ├───> 6.00% (600 BPS) ──> Affiliate Referrer
                 └───> 9.00% (900 BPS) ──> Protocol Treasury (0x2735...)
```

### 7.3 Mathematical Implementation & Safe Division

To prevent integer overflow and rounding errors on high-precision or low-decimal tokens, `SponsorableBatchExecutor` executes a two-tier arithmetic routine:

```solidity
uint256 internal constant BPS_DENOMINATOR = 10000;

function _splitFee(uint256 amount) internal view returns (uint256 feeAmount, uint256 sendAmount) {
    uint256 currentFeeBps = _getFeeBps(); // 1500n
    if (currentFeeBps == 0) {
        return (0, amount);
    }
    // Safe multiplication check
    if (amount <= type(uint256).max / currentFeeBps) {
        feeAmount = (amount * currentFeeBps) / BPS_DENOMINATOR;
    } else {
        // High-magnitude overflow prevention
        feeAmount = (amount / BPS_DENOMINATOR) * currentFeeBps 
                  + ((amount % BPS_DENOMINATOR) * currentFeeBps) / BPS_DENOMINATOR;
    }
    sendAmount = amount - feeAmount;
}
```

- **Rounding Direction:** Integer division in Solidity truncates toward zero. This favors the user, guaranteeing that the deducted fee never exceeds `15.0000...%`.

### 7.4 Zero-Fee Exemptions & Asset Exclusions

- **NFTs (ERC-721 / ERC-1155):** `0%` Fee. Non-fungible tokens are forwarded completely intact.
- **Compromised Wallet Native Reserve:** On chains with mandatory state reserves (e.g., Monad's 10.5 MON reserve), the reserve is subtracted before calculating fees.

### 7.5 Asset Valuation Dynamics & Oracle Independence

RescueKit does not rely on external off-chain price oracles (Chainlink, Pyth) to compute token fees. 

- **Physical Settlement:** Fees are collected *in kind*. If 1,000 USDC is recovered, the contract transfers 150 USDC to the protocol/affiliate and 850 USDC to the safe destination.
- **Oracle Risk Elimination:** By taking fee cuts directly in the recovered asset, the protocol eliminates oracle downtime, price manipulation, stale feed errors, and flash-crash vulnerabilities.

---

## 8. Transaction Construction, Gas Dynamics & Type-4 Lifecycle

### 8.1 The Two-Component Gas Equation

Executing an asset rescue requires deriving two independent values:

$$\text{Total Transaction Cost} = \text{Gas Units} \times \text{Gwei Price}$$

| Component | Nature | Source | Dependency on RPC |
|---|---|---|---|
| **Gas Units** | Quantity (e.g., 350,000) | Analytical formula (`rescueGasFallback`) | **No (Deterministic Math)** |
| **Gwei Price** | Price per unit (e.g., 25 gwei) | Live network state (`estimateFeesPerGas`) | **Yes (Live Network Query)** |

### 8.2 Analytical Gas-Unit Fallback Formulas (`rescueGasFallback`)

Public RPC nodes frequently return invalid or failing `eth_estimateGas` simulations when evaluating EIP-7702 transactions on accounts that are not yet delegated on-chain. RescueKit solves this by employing worst-case analytical math:

#### Standard Networks (Base, Optimism, BSC, Arbitrum, Ethereum, Linea, Sonic, etc.)
$$\text{Gas}_{\text{Transfer}} = \max(100{,}000 + 65{,}000 \times N_{\text{calls}},\, 350{,}000)$$
$$\text{Gas}_{\text{Mint}} = \max(300{,}000 + 65{,}000 \times N_{\text{calls}},\, 450{,}000)$$
$$\text{Gas}_{\text{Claim}} = \max(250{,}000 + 65{,}000 \times N_{\text{calls}},\, 400{,}000)$$
$$\text{Gas}_{\text{Lending}} = \max(150{,}000 + 450{,}000 \times N_{\text{debt}} + 120{,}000 \times N_{\text{idle}},\, 500{,}000)$$

#### Polygon Override (Heavier Bytecode Access)
$$\text{Gas}_{\text{Transfer}} = \max(120{,}000 + 80{,}000 \times N_{\text{calls}},\, 400{,}000)$$
$$\text{Gas}_{\text{Lending}} = \max(200{,}000 + 500{,}000 \times N_{\text{debt}} + 150{,}000 \times N_{\text{idle}},\, 700{,}000)$$

#### Monad Override (Parallel Execution Overhead)
$$\text{Gas}_{\text{Lending}} = \begin{cases} 1{,}500{,}000 & \text{if } N_{\text{debt}} > 0 \\ \max(150{,}000 + 120{,}000 \times N_{\text{idle}},\, 500{,}000) & \text{if } N_{\text{debt}} = 0 \end{cases}$$

### 8.3 Live Fee Pricing Multipliers & Network Spikes

To guarantee inclusion ahead of competing mempool transactions, RescueKit applies aggressive fee multipliers over base fee and tip:

- **Base Fee Multiplier:** `3x` (buffers against rapid block base fee spikes).
- **Tip Multiplier:**
  - Standard Sequencer L2s: `1x - 2x`
  - High-Competition / Mempool Chains (BNB Chain, Monad): `5x`
- **Fee Guard:** If RPC returns `0` for all fee parameters, the transaction halts immediately to prevent broadcasting zero-tip transactions.

### 8.4 Receipt Polling, Confirmation Timeouts & Nonce Coordination

- **Receipt Wait Timeout:** 30 seconds on fast chains (Monad), 60 seconds on standard L2s, 90 seconds on Polygon.
- **Sponsor Nonce Management:** Queries `getTransactionCount(address, "pending")` to ensure subsequent rescues do not collide with unmined transactions.

---

## 9. Security Architecture, Threat Vectors & Trust Boundaries

### 9.1 Private Key Isolation & Client-Side Ephemeral Signing

```
┌─────────────────────────────────────────────────────────┐
│                      CLIENT BROWSER                     │
│  Compromised Key ──> Ephemeral Memory (viem Account)   │
│                             │                           │
│                      signAuthorization()                │
│                             │                           │
│                             ▼                           │
│                 EIP-7702 Signature Tuple                │
│                  { chainId, address, nonce, r, s, y }    │
└─────────────────────────────┬───────────────────────────┘
                              │ (Public Payload Only)
                              ▼
                  Broadcasted to Network RPC
          (Private Key NEVER Leaves Browser Memory)
```

- **Zero Exfiltration:** The compromised private key is never transmitted over HTTP, WebSockets, or remote logging services.
- **Memory Destruction:** Keys held in React component state are cleared upon tab closure or session reset.

### 9.2 Cryptographic Replay Protection & Chain ID Domain Separation

- **EIP-7702 Tuples:** Include explicit `chainId`. An authorization signed for Base (`8453`) cannot be replayed on Ethereum (`1`) or Monad (`143`).
- **Nonce Invalidation:** Every executed transaction increments the compromised account's nonce, permanently invalidating prior authorization tuples.

### 9.3 Frontrunning, Mempool Leakage & Private Relay Routing

- **Sequencer L2s:** Transactions submitted to Base, Arbitrum, Optimism, etc., route directly to the chain's private sequencer endpoint, bypassing public P2P mempools.
- **Public Mempool Networks (BSC, Polygon):** Autonomous monitoring engines broadcast via private transaction relays (e.g., Flashbots / bloXroute) to prevent MEV searchers from unbundling calldata.

### 9.4 What RescueKit Protects Against vs. What It Does Not Protect Against

#### What RescueKit Protects Against:
- Sweeper bots monitoring for inbound gas transfers.
- Re-delegation hijacking during the recovery block.
- Flash loan repayment failures (atomic rollback).
- Incomplete approvals or orphaned token approvals.

#### What RescueKit Does NOT Protect Against:
- **Pre-Existing Depletion:** Assets already transferred out by the attacker prior to RescueKit execution cannot be recovered.
- **Compromised Safe Destination:** If the user supplies an attacker-controlled address as the safe destination, assets are sent to the attacker.
- **Compromised Sponsor Wallet:** If the sponsor wallet's key is also leaked, sweeper bots will drain the sponsor wallet's native gas before broadcast.

---

## 10. Sponsor Wallet Architecture & Operational Mechanics

### 10.1 Sponsor Wallet Definition & Isolation

The Sponsor Wallet is an independent EOA created specifically to supply gas fees. 

- **Security Isolation:** The Sponsor Wallet must have **zero historical link** to the compromised private key.
- **Key Separation:** Compromised wallet signs the EIP-7702 authorization tuple; Sponsor wallet signs and broadcasts the Type-4 transaction.

### 10.2 Sponsor Funding Requirements & Calculations

The minimum native gas required in the Sponsor Wallet is determined by:

$$\text{Required Native Balance} = \text{Gas Units} \times (\text{Base Fee} + \text{Priority Fee})$$

If the Sponsor Wallet balance is below this threshold, the review modal disables execution and prompts the user with the exact missing native balance required.

### 10.3 Unused Gas Balance Reclamation

Gas limits configured on EVM transactions represent caps, not fixed charges (with the exception of Monad's gasLimit billing):
- Any unused gas units remain in the Sponsor Wallet.
- The user can immediately transfer remaining funds out of the Sponsor Wallet once the recovery confirms.

---

## 11. Referral & Affiliate Commission System

### 11.1 On-Chain Affiliate Attribution

RescueKit implements a permissionless referral system embedded directly into `SponsorableBatchExecutor.sol`:

- **Referral Code:** Encodes the affiliate's EVM address.
- **Execution Parameter:** Passed as the `referrer` address inside `executionData`.
- **Commission Split:** The affiliate receives 6.00% (600 BPS) of gross recovered assets directly from the protocol fee.

### 11.2 Self-Referral Prevention Logic

To prevent operators from gaming the protocol by referring their own recoveries:

```solidity
if (canPayReferral && target != referrer) {
    // Pay affiliate commission
}
```

If the `safeDestination` (`target`) matches the `referrer` address, referral attribution is disabled, and 100% of the 15% fee routes to the protocol treasury.

### 11.3 Gas-Stipended Transfer Isolation (50,000 Gas Guard)

To prevent a malicious or reverting smart-contract affiliate address from bricking the entire recovery transaction, outbound affiliate transfers are constrained:

```solidity
(bool affOk, ) = referrer.call{value: affiliateCut, gas: 50_000}("");
```

- **Gas Limit:** 50,000 units.
- **Reversion Handling:** If the affiliate call reverts or runs out of gas, the executor catches the failure, emits `ReferralFailed(...)`, and reroutes the affiliate cut to the protocol treasury. The user's recovery transaction **succeeds regardless**.

---

## 12. REST API Specification & Execution Models

The RescueKit backend provides three high-performance REST API endpoints for programmatic integration.

### 12.1 `/api/build` — Calldata & Batch Construction

- **Method:** `POST`
- **Headers:** `Content-Type: application/json`
- **Purpose:** Compiles recovery parameters into ERC-7821 compliant batch execution calldata and computes authorization hashes.

#### Request Schema:
```json
{
  "chainId": 8453,
  "action": "rescue",
  "compromised": "0x9f3...a21",
  "safeDestination": "0x4c8...7be",
  "sponsor": "0x1b2...9f0",
  "tokens": [
    { "address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", "amount": "1000000000" }
  ],
  "nftItems": [],
  "referrer": "0x2735fAfD319155d8F7548d4FD68222cDbE808F39"
}
```

#### Response Schema:
```json
{
  "success": true,
  "data": {
    "to": "0x9f3...a21",
    "erc7821Address": "0x0000000004C9B572E8aB03C7A7377AaadEfd3502",
    "executionData": "0x...",
    "mode": "0x0100000000007821000100000000000000000000000000000000000000000000",
    "digest": "0x..."
  }
}
```

### 12.2 `/api/execute` — Server-Side Execution Pipeline

- **Method:** `POST`
- **Purpose:** Full end-to-end execution for automated recovery agents.
- **Payload:** Accepts signed EIP-7702 authorization tuples along with sponsor credentials to broadcast transactions directly to the network.

### 12.3 `/api/broadcast` — Direct Raw Transaction Relay

- **Method:** `POST`
- **Purpose:** Relays pre-signed Type-4 transactions directly through configured private RPC nodes or Flashbots relays.

### 12.4 Rate Limiting & Denial-of-Service Mitigations

- **Limiter:** Sliding-window IP rate limiting (60 requests per minute).
- **Abuse Prevention:** Malformed payloads or invalid contract addresses return `HTTP 400 Bad Request` with structured error messages.

---

## 13. Smart Contracts & Technical Interface Reference

### 13.1 `SponsorableBatchExecutor` Contract Specification

- **Contract Name:** `SponsorableBatchExecutor`
- **Solidity Version:** `^0.8.20` (Deployed via `0.8.37`)
- **Canonical Address:** `0x0000000004C9B572E8aB03C7A7377AaadEfd3502`
- **Compiler Optimizations:** Enabled (200 runs)

### 13.2 Execution Modes & Bitmask Flags

```solidity
// Canonical Mode Constants
bytes32 constant MODE_SINGLE_BATCH         = 0x0100000000000000000000000000000000000000000000000000000000000000;
bytes32 constant MODE_BATCH_WITH_OPDATA    = 0x0100000000007821000100000000000000000000000000000000000000000000;
bytes32 constant MODE_NFT_SPONSOR_RESCUE   = 0x0100000000007821000300000000000000000000000000000000000000000000;
bytes32 constant MODE_FLASH_LOAN_BATCH     = 0x0100000000007821000400000000000000000000000000000000000000000000;
bytes32 constant MODE_CLAIM_BATCH          = 0x0100000000007821000600000000000000000000000000000000000000000000;
bytes32 constant MODE_MINT_721_BATCH       = 0x0100000000007821000700000000000000000000000000000000000000000000;
bytes32 constant MODE_MINT_1155_BATCH      = 0x0100000000007821000800000000000000000000000000000000000000000000;
bytes32 constant MODE_MULTI_CLAIM_BATCH    = 0x0100000000007821000900000000000000000000000000000000000000000000;
```

### 13.3 Contract Administrative Controls & Upgradability

- **Non-Upgradable:** Contract logic is immutable. There are no proxies or admin implementation pointers.
- **Fee Configuration:** Admin functions (`setFeeBps`, `setAffiliateCutBps`, `setFeeRecipient`) can only adjust fee basis points within hardcoded bounds:
  - Maximum fee limit: `2000` BPS (20.00%).
  - Enforced pause: Emergency pause mechanism protects against protocol-level zero-day vulnerabilities.

### 13.4 Interface Identifiers (`supportsInterface`)

The contract returns `true` for:
- `0x01ffc9a7`: ERC-165 Standard Interface Detection
- `0x150b7a02`: ERC-721 Token Receiver (`onERC721Received`)
- `0x4e2312e0`: ERC-1155 Token Receiver (`onERC1155Received` / `onERC1155BatchReceived`)

---

## 14. User Input Field Reference & Validation Matrix

| Field Name | Format | Required | Validation Rule | Invalid Behavior | Empty Behavior |
|---|---|---|---|---|---|
| **Compromised Key** | 64-char Hex (`0x...`) | Yes | Must derive valid secp256k1 public address | Rejects with "Invalid private key format" | Submit button disabled |
| **Safe Destination** | 40-char Hex Address | Yes | Must pass EIP-55 checksum validation | Rejects with "Invalid destination address" | Submit button disabled |
| **Sponsor Wallet** | Connected Web3 Provider | Yes | Must have native balance $\ge$ gas estimate | Surfaces "Insufficient sponsor balance" | Blocks review modal |
| **Token Address** | 40-char Hex Address | Optional | Must be deployed contract on active network | Displays "Contract not found on chain" | Ignored |
| **NFT Token ID** | Unsigned Integer (`uint256`) | Optional | Must be owned by compromised address | Excluded during scan | Ignored |
| **Claim Calldata** | Raw ABI Hex Bytes | Optional | Must decode against target distributor ABI | Reverts simulation: "Invalid claim payload" | Bypasses claim step |

---

## 15. Error Reference, Contract Reverts & Failure Resolutions

### 15.1 Smart Contract Custom Revert Errors

- **`UnsupportedExecutionMode()`**
  - *Cause:* Calldata specified an ERC-7821 execution mode not supported by the contract.
  - *Resolution:* Rebuild calldata using supported modes (1, 2, 3, 4, 6, 7, 8, 9).
- **`Unauthorized()`**
  - *Cause:* Direct call to internal execution routines without proper signature authorization context.
  - *Resolution:* Ensure transaction is signed via EIP-7702 authorization list tuple.
- **`SlippageExceeded()`**
  - *Cause:* In-lock swap yielded less than `amountOutMinimum` during flash loan debt settlement.
  - *Resolution:* Increase slippage tolerance or wait for pool liquidity to normalize.
- **`EnforcedPause()`**
  - *Cause:* Protocol administrative pause is active.
  - *Resolution:* Check official protocol status announcements.

### 15.2 Client-Side Validation & Simulation Rejections

- **`"Insufficient sponsor balance"`**
  - *Cause:* Sponsor wallet does not hold enough native gas tokens to cover the worst-case gas limit.
  - *Resolution:* Deposit additional native gas tokens into the Sponsor Wallet.
- **`"Compromised account nonce mismatch"`**
  - *Cause:* On-chain nonce changed between authorization signing and broadcast.
  - *Resolution:* Re-sign the authorization tuple with the updated nonce.

---

## 16. The "What Happens If..." Real-World Edge Case Directory

### 16.1 Asset & State Alterations

- **What if assets disappear before execution?**
  - If a sweeper bot moves an ERC-20 token before RescueKit's transaction is included, the `balanceOf` query inside the contract returns `0`. The contract skips the zero-balance transfer and continues sweeping remaining assets without reverting.
- **What if an airdrop claim expires?**
  - If the distributor contract reverts because the claim window closed, the atomic batch reverts. Mode 9 multi-claim batches can be configured with soft-failure flags to bypass reverted sub-claims.

### 16.2 Transaction Execution & Network Failures

- **What if the lending repayment fails?**
  - If collateral cannot satisfy debt repayment or flash loan repayment fails, the entire transaction reverts atomically. No collateral is lost, and the loan remains in its prior state.
- **What if the RPC endpoint goes offline during broadcast?**
  - The client automatically retries against configured fallback RPC endpoints (`fallbackRpcs`) without requiring re-signing.
- **What if I close or refresh the browser tab?**
  - If broadcast has already occurred, the transaction executes on-chain independently. If broadcast has not occurred, ephemeral memory is cleared and no transaction is sent.

### 16.3 Protocol & Financial Edge Cases

- **What if the wallet is rescued twice?**
  - The second rescue executes normally if new assets have arrived. If no assets exist, zero-value calls execute harmlessly, costing only sponsor gas.
- **What if the affiliate payment fails?**
  - Outbound affiliate transfer reverts are caught. The commission is routed to the treasury and the user's asset recovery completes without interruption.

---

## 17. Architectural Comparisons & Industry Alternatives

| Feature | RescueKit (EIP-7702) | Flashbots Bundles | Traditional Sweeping | Smart Contract Wallets (ERC-4337) |
|---|---|---|---|---|
| **Compromised Wallet Gas Funding** | **0 ETH (Zero)** | 0 ETH (Miner Tip) | **Requires Gas Funding** | 0 ETH (Paymaster) |
| **Sweeper Bot Exploitation Risk** | **Zero Funding Gap** | Low (Mempool private) | **Extreme (Immediate theft)** | Zero Funding Gap |
| **Supported Chains** | **19 EVM Mainnets** | Ethereum Only | Any | Network Dependent |
| **DeFi Debt Liquidation Support** | **Built-in Flash Loans** | Complex manual setup | Not supported | Requires custom paymaster |
| **Account Ownership Changes** | **None (Ephemeral)** | None | None | Requires account migration |
| **NFT Support** | **100% Free (0% Fee)** | Manual bundle cost | Manual transfer | Paymaster gas sponsor |

---

## 18. Comprehensive FAQ

#### Q: How does RescueKit bypass sweeper bots?
A: Sweeper bots can only steal assets if they have gas to transfer or if gas is deposited into the compromised account. RescueKit sponsors transactions externally via EIP-7702. Because the compromised account never receives gas, the bot has nothing to take.

#### Q: Is my private key uploaded to a server?
A: No. Private keys are used strictly inside your browser's local memory to sign an EIP-7702 authorization tuple. They are never transmitted over the internet.

#### Q: What chains are supported?
A: All 19 major EVM production mainnets listed in Section 5.1 are fully supported.

#### Q: What fee does RescueKit charge?
A: A 15% protocol fee is deducted on-chain exclusively from recovered fungible assets and native currency upon successful execution. NFTs are rescued completely free of protocol fees.

---

## 19. Protocol Limitations & Known Issues

1. **Non-Empty Calldata Native Call Fee Bypass:** In current contract builds, non-empty data native calls can theoretically bypass the fee split if constructed manually outside the official builder. The official builder always routes through the auto-split path.
2. **Public Mempool Frontrunning on BSC/Polygon:** On networks lacking sequencer privacy, high-priority public mempool transactions can theoretically be observed by sophisticated MEV bots capable of parsing 7702 calldata. Private relays are enforced for automated engines on these chains.
3. **RPC 7702 Simulation Inconsistencies:** Some third-party RPC providers do not yet support `eth_estimateGas` simulations with an `authorizationList`. The protocol bypasses this using deterministic fallback formulas.

---

## 20. Technical Glossary

- **EIP-7702:** Ethereum Improvement Proposal enabling EOAs to temporarily designate smart contract execution code for a single transaction.
- **ERC-7821:** Minimal batch execution interface standard defining `execute(bytes32 mode, bytes executionData)`.
- **Sponsor Wallet:** An uncompromised secondary account used exclusively to pay native network gas fees for the recovery transaction.
- **Sweeper Bot:** An automated script monitoring compromised accounts to instantly frontrun and steal deposited gas tokens.
- **Type-4 Transaction:** An EVM transaction carrying an EIP-7702 `authorizationList`.

---

## 21. Documentation Fact & Verification Ledger

| Fact / Assertion | Evidence Source | Technical Verification Path | Status |
|---|---|---|---|
| 19 Supported Mainnet Chains | `packages/chains/src/index.ts` & Live Bundle | `CHAINS.length == 19` confirmed on live Vercel build | **VERIFIED** |
| Deterministic Contract Address | `deploy_all_create2.mjs` | CREATE2 calculation `0x0000000004C9B572E8aB03C7A7377AaadEfd3502` | **VERIFIED** |
| 1500 BPS Protocol Fee | `SponsorableBatchExecutor.sol` | `_getFeeBps() == 1500n` | **VERIFIED** |
| 600 BPS Affiliate Cut | `SponsorableBatchExecutor.sol` | `_getAffiliateCutBps() == 600n` | **VERIFIED** |
| 0% Protocol Fee on NFTs | `SponsorableBatchExecutor.sol` | `_runMintBatch721` transfers full token without fee deduction | **VERIFIED** |
| Client-Side Key Handling | `packages/web/src/services/rescue.ts` | `privateKeyToAccount` invoked locally in browser context | **VERIFIED** |
| Flash Loan Providers | `packages/batch/src/lending/flashloans/` | Aave v3, Balancer v3, Morpho Blue, Uniswap v4 implementations | **VERIFIED** |

---

## 22. Version History & Protocol Changelog

- **v2.4.0 (Current):**
  - Added Plume Network mainnet deployment, expanding registry to 19 chains.
  - Deployed 9-zero `SponsorableBatchExecutor` vanity address across all networks.
  - Integrated Moonwell and Morpho isolated lending collateral recovery.
  - Integrated in-lock Uniswap v4 flash swaps.
- **v2.3.0:**
  - Standardized protocol fee at 15.00% (1500 BPS) with 6.00% (600 BPS) affiliate cut across all deployed chains.
  - Added Curvance money market recovery for Monad.
- **v2.2.0:**
  - Migrated chain architecture to self-contained per-chain strategy modules.
  - Implemented analytical gas-unit fallback formulas (`rescueGasFallback`).
