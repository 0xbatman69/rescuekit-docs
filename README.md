# RescueKit Official Documentation

> Atomic, non-custodial asset recovery protocol for compromised EVM wallets powered by EIP-7702 ephemeral delegation and ERC-7821 batch execution.

RescueKit enables victims of private key compromises to atomically rescue trapped ERC-20 tokens, native assets, NFTs (ERC-721 and ERC-1155), airdrop/vesting claims, and DeFi lending collateral without ever funding the hacked wallet with native gas—completely bypassing sweeper bots.

---

## Documentation Index

### 1. Architecture & Core Mechanics
- [Core Architecture & Protocol Security](#core-architecture)
- [EIP-7702 Ephemeral Delegation & Sweeper Protection](#how-eip-7702-bypasses-sweeper-bots)
- [Protocol Fee & Settlement Mechanics](#protocol-fee--settlement-mechanics)
- [Supported Networks & Capabilities](#supported-networks)

### 2. User & Application Guides
- [Token & NFT Rescue Guide (`/transfer`)](./guides/token-and-nft-rescue.md)
  - Rescuing ERC-20 tokens, native ETH/gas, and NFTs
  - Dynamic scanning, custom token/NFT imports, and destination verification
- [NFT Mint & Save Guide (`/mint`)](./guides/nft-mint-and-save.md)
  - Allowlist, public, and paid minting with immediate atomic sweep
  - ERC-721 (Mode 7) and ERC-1155 (Mode 8) multi-edition support
- [Airdrop, Staking & Vesting Claims Guide (`/claim`)](./guides/airdrop-and-claims.md)
  - Single-claim (Mode 6) and multi-claim (Mode 9) workflows
  - Sweep token mapping, residual gas recovery, and token follow-ups
- [DeFi Lending Collateral Rescue Guide (`/lending`)](./guides/lending-collateral-rescue.md)
  - Flash-loan debt repayment and collateral extraction (Aave v3, Moonwell, Morpho)
  - Multi-collateral swaps, custom calldata, and human-readable function signatures
- [Sponsor Gas Wallet & Affiliate Referral Guide (`/refer`)](./guides/sponsor-wallet-and-referrals.md)
  - Burner sponsor wallet generation, funding, exporting, and custom key import
  - 6% on-chain native commission mechanics and referral link builder

### 3. Developer & Agent REST API
- [Headless REST API Reference](./api/rest-api-reference.md)
  - `POST /api/build`: 100% offline batch assembly, gas calculation, and digest building
  - `POST /api/broadcast`: Private relay transaction broadcast (Flashbots / 48 Club / Custom RPC)
  - `POST /api/execute`: Headless 1-shot execution with private keys
  - Rate limiting, anonymized SHA-256 client tokens, and RFC headers
  - Zero-key two-pass client signing tutorial (TypeScript / Viem)

### 4. Technical Reference & Error Troubleshooting
- [Security Model & Troubleshooting Directory](./reference/smart-contracts-and-errors.md)
  - Non-custodial architecture, EIP-7702 delegation, and fee-first settlement
  - Replay protection, private mempool execution, and anti-exploit guarantees
  - Complete on-chain and UI error directory: contract reverts, input validation, and RPC resolutions

---

## Core Architecture

### How EIP-7702 Bypasses Sweeper Bots

In traditional EVM transactions, executing a transfer from an account requires that account to pay gas in its native currency (e.g. ETH, BNB, POL). When a private key is leaked:
1. Malicious searcher scripts ("sweeper bots") monitor the mempool and balance of the compromised address.
2. The instant any native gas is deposited to pay for a transfer, the sweeper bot broadcasts a higher-priority transaction that siphons the gas before the victim can execute their recovery.

RescueKit eliminates this attack vector entirely using **EIP-7702**:
- The compromised wallet signs an **EIP-7702 Authorization tuple** designating RescueKit's batch executor contract (`SponsorableBatchExecutor`) as its ephemeral code implementation for the duration of the transaction.
- An independent, clean **Sponsor Gas Wallet** pays all network gas fees by broadcasting a Type-4 transaction (`0x04`) containing:
  - `authorizationList`: The compromised wallet's signed EIP-7702 delegation tuple.
  - `to`: The compromised wallet address.
  - `data`: The ERC-7821 batch execution payload.
- The transaction executes in the execution context of the compromised wallet (`address(this)` is the compromised address), transferring all designated assets directly to the user's `safeDestination`.
- **Zero native gas is ever deposited into the compromised address.** Sweeper bots remain completely inert.
- **100% Non-Custodial (No Escrow or Protocol Vault)**: The contract never takes custody of user assets. Assets transfer directly from the victim's account to `safeDestination` within the same atomic block. User funds can never be locked or stuck in a contract because no protocol vault or intermediate pool exists.

### Protocol Fee & Settlement Mechanics

- **Protocol Fee**: Fixed at **15.00%** of rescued asset value (`feeBps = 1500`, 1,500 / 10,000 basis points).
- **Affiliate Commission**: **40.00%** of the protocol fee (equivalent to **6.00%** gross of total rescued value) is automatically diverted and paid directly to the designated referrer address on-chain.
- **Protocol Treasury**: **9.00%** when referred; **15.00%** when unreferred, self-referred, or during repeat rescues.
- **Victim Net Recovery**: **85.00%** of gross asset value is transferred directly to `safeDestination`.
- **Double-Fallback Refund Safeguard (91.00% Net)**: If an affiliate payout fails and the subsequent redirect to the protocol treasury also fails, the unpayable 6.00% cut is automatically refunded into the user's sweep, delivering **91.00%** net recovery to `safeDestination` rather than leaving funds behind in the compromised wallet.
- **Fee-First Settlement Architecture (Anti-Exploit Protection)**: Protocol fees are processed on-chain *before* the remaining balance is dispatched to the safe destination. This design prevents malicious tokens, griefers, or bad actors from blacklisting the treasury address or engineering revert traps to siphon assets fee-free.
- **Clean Safe Wallet Requirement**: Because fees are settled prior to the final destination sweep, users must always supply a clean, unencumbered recovery wallet (a standard EOA or verified Safe multisig). If `safeDestination` rejects the transfer (e.g. an address blacklisted by centralized tokens like USDC/USDT, or a smart contract lacking a `receive()` function), the batch emits `CallFailed` without reverting—the protocol fee remains collected, and the remaining 85% (or 91%) stays behind in the compromised wallet.
- **Atomic Transaction Reverts**: If an entire recovery transaction fails and reverts on-chain (e.g. due to invalid signatures, flash loan conditions, or gas exhaustion), all state changes roll back atomically via standard EVM execution and zero fees are charged.

---

## Supported Networks

RescueKit is deployed and operational across 6 EVM networks:

| Chain Name | Chain ID | Native Gas Token | EIP-7702 Status | Private Relay Protection | Block Explorer |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Ethereum** | `1` | ETH | Active | Flashbots Protect (`https://rpc.flashbots.net`) | [Etherscan](https://etherscan.io) |
| **Base** | `8453` | ETH | Active | Standard Sequencer Pool | [Basescan](https://basescan.org) |
| **Optimism** | `10` | ETH | Active | Standard Sequencer Pool | [Optimistic Etherscan](https://optimistic.etherscan.io) |
| **BNB Smart Chain** | `56` | BNB | Active | 48 Club Private Relay (`https://rpc.48.club`) | [BscScan](https://bscscan.com) |
| **Polygon** | `137` | POL / MATIC | Active | Standard Failover Pool | [Polygonscan](https://polygonscan.com) |
| **Monad** | `143` | MON | Active | Standard Failover Pool | [MonadScan](https://monadscan.com) |
