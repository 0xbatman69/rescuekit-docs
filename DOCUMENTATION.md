# RescueKit Documentation

---

## Table of Contents

1. [Introduction](#1-introduction)
   - 1.1 How Sweeper Bots Work
   - 1.2 How EIP-7702 Solves This
2. [Quick Start](#2-quick-start)
   - 2.1 What You Need
   - 2.2 How to Rescue
3. [Token & NFT Rescue (`/transfer`)](#3-token--nft-rescue-transfer)
   - 3.1 How It Works
   - 3.2 Fees
4. [Airdrop & Claims Rescue (`/claim`)](#4-airdrop--claims-rescue-claim)
   - 4.1 How It Works
   - 4.2 Follow-Up Rescues
   - 4.3 Fees
5. [NFT Mint Rescue (`/mint`)](#5-nft-mint-rescue-mint)
   - 5.1 How It Works
   - 5.2 Finding Token IDs
   - 5.3 Fees
6. [DeFi Lending Rescue (`/lending`)](#6-defi-lending-rescue-lending)
   - 6.1 How It Works
   - 6.2 Debt Repayment, Swaps & Idle Deposits
   - 6.3 Fees
7. [Sponsor Wallet](#7-sponsor-wallet)
   - 7.1 How It Works
   - 7.2 Managing Keys & Funds
   - 7.3 Tips & Safety
8. [Referral Program (`/refer`)](#8-referral-program-refer)
   - 8.1 How It Works
   - 8.2 Commissions & Payouts
9. [Fees & How They Work](#9-fees--how-they-work)
   - 9.1 Fee Breakdown by Asset
   - 9.2 Referral Split
   - 9.3 Transfer Order & Safe Destination Requirements
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
12. [Smart Contracts & Technical Interface Reference](#12-smart-contracts--technical-interface-reference)
    - 12.1 `SponsorableBatchExecutor` Contract Specification
    - 12.2 Execution Modes & Bitmask Flags
    - 12.3 Contract Administrative Controls & Upgradability
    - 12.4 Interface Identifiers (`supportsInterface`)
13. [User Input Field Reference & Validation Matrix](#13-user-input-field-reference--validation-matrix)
    - 13.1 Compromised Wallet Address & Key
    - 13.2 Safe Destination Address
    - 13.3 Sponsor Wallet Address & Key
    - 13.4 Contract Addresses, Token IDs & Call Data
14. [Error Reference, Contract Reverts & Failure Resolutions](#14-error-reference-contract-reverts--failure-resolutions)
    - 14.1 Smart Contract Custom Revert Errors
    - 14.2 Client-Side Validation & Simulation Rejections
    - 14.3 EVM JSON-RPC & Broadcast Rejections
15. [The "What Happens If..." Real-World Edge Case Directory](#15-the-what-happens-if-real-world-edge-case-directory)
    - 15.1 Asset & State Alterations
    - 15.2 Transaction Execution & Network Failures
    - 15.3 Protocol & Financial Edge Cases
16. [Architectural Comparisons & Industry Alternatives](#16-architectural-comparisons--industry-alternatives)
    - 16.1 RescueKit vs. Flashbots Private Bundles
    - 16.2 RescueKit vs. ERC-4337 Account Abstraction
    - 16.3 RescueKit vs. Direct Account Gas Funding
17. [Comprehensive FAQ](#17-comprehensive-faq)
18. [Protocol Limitations & Known Issues](#18-protocol-limitations--known-issues)
19. [Technical Glossary](#19-technical-glossary)
20. [Documentation Fact & Verification Ledger](#20-documentation-fact--verification-ledger)
21. [Version History & Protocol Changelog](#21-version-history--protocol-changelog)
22. [Developer REST API Reference](./api/rest-api-reference.md)

---

## 1. Introduction

RescueKit is a mission-critical asset recovery protocol engineered specifically for compromised Ethereum Virtual Machine (EVM) accounts. By harnessing Ethereum Improvement Proposal 7702 (EIP-7702), RescueKit enables users to recover tokens, non-fungible tokens (NFTs), unclaimed airdrops, vesting releases, and collateral locked in decentralized finance (DeFi) lending markets without ever funding the compromised account with native gas.

### 1.1 How Sweeper Bots Work

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

### 1.2 How EIP-7702 Solves This

EIP-7702 introduces an ephemeral smart contract delegation standard for Externally Owned Accounts (EOAs). Instead of altering account ownership or deploying an expensive proxy contract, EIP-7702 permits an EOA to sign an authorization tuple designating an external smart contract implementation (`0x0000000004C9B572E8aB03C7A7377AaadEfd3502`) to execute code on behalf of the EOA for the duration of a single transaction.

Crucially, EIP-7702 transactions can be sponsored by an independent, uncompromised account (the "Sponsor Wallet"). The sponsor wallet pays 100% of the native gas required to broadcast and execute the transaction. The compromised account never receives, holds, or spends native gas tokens, completely starving the sweeper bot of an exploitation vector.


---

## 2. Quick Start

### 2.1 What You Need

- Compromised wallet address and its private key.
- A clean, uncompromised safe destination address.
- A sponsor wallet created on RescueKit, funded with enough gas for the network fee.

### 2.2 How to Rescue

1. Create a sponsor wallet in the app and deposit gas into it.
2. Enter your compromised wallet address and select your network.
3. Select or enter the items to recover.
4. Enter your clean safe destination address.
5. Click **Review** and enter your compromised private key.
6. Click **Rescue** to execute the rescue transaction.

---

## 3. Token & NFT Rescue (`/transfer`)

The Transfer page recovers ERC-20 tokens, native coins, and NFTs already sitting in your compromised wallet.

### 3.1 How It Works

1. Enter your compromised address and select your networks. The scanner checks for tokens, native coins, and NFTs across all selected chains at the same time.
2. You can rescue multiple tokens, multiple NFTs, or both combined at once. If you selected multiple networks, you can rescue across all of them at the same time with one click.
3. If a token or NFT does not appear automatically, you can manually enter the contract address (and token ID for NFTs) to include it.
4. Once confirmed via **Review** and **Rescue**, the assets are swept directly into your safe destination wallet.
5. If an individual token or NFT transfer fails (for example, non-transferrable token or nft), it simply skips that token and continues rescuing the rest of your assets without canceling the entire transaction.

### 3.2 Fees

- Tokens & Native Currency: A 15% recovery fee is deducted on-chain directly from the recovered amount during the transfer. You never pay upfront fees.
- NFTs (ERC-721 & ERC-1155): Rescued with 0% protocol fee. 100% of your NFTs go directly to your safe wallet.

---

## 4. Airdrop & Claims Rescue (`/claim`)

The Claims page recovers claimable tokens from contracts like airdrops, staking, vesting, and more.

### 4.1 How It Works

1. Enter your compromised address and select your network.
2. Enter the claim contract address. The app automatically checks the network to verify that the contract exists. If the contract supports common claim functions, it is detected automatically; otherwise, paste your claim calldata.
3. If the claim requires a native fee, enter the amount (like `0.01` or hex `0x...`) in the Value field. Your sponsor wallet pays this fee for you. If no fee is required, leave it blank.
4. The payout token is usually detected and filled in automatically, but always verify that the address is correct (or enter it manually if not detected). Native tokens are swept automatically by default, and you can also sweep existing tokens already sitting in your wallet by adding their contract addresses.
5. If a claim rewards multiple tokens at once, you can add extra token addresses to sweep all reward tokens together in one transaction.
6. You can add multiple claim boxes to execute different claims together at the same time.
7. Click **Review** to enter your compromised private key, then click **Rescue** to sweep your tokens directly into your safe wallet.

### 4.2 Follow-Up Rescues

If an individual claim fails (for example, if it expired), it simply skips that claim and continues rescuing your other claims without canceling the entire transaction.

Always enter the correct payout tokens so everything sweeps in the first transaction. If a contract transfers extra or unexpected tokens that you did not enter, an automatic follow-up rescue broadcasts immediately to recover them. However, because this requires a separate transaction, there is a brief on-chain window where tokens could be intercepted. While the follow-up rescue is fast, there is no guarantee in that window, so always double-check your token addresses.

### 4.3 Fees

- A 15% recovery fee is deducted on-chain directly from the claimed tokens upon successful sweep. You never pay upfront fees.

---

## 5. NFT Mint Rescue (`/mint`)

The Mint page lets you mint NFTs (ERC-721 or ERC-1155) from an eligible or allowlisted compromised wallet, either sweeping them straight to your safe wallet in the same transaction or executing the mint on its own.

### 5.1 How It Works

1. Enter your compromised wallet address, safe destination address, and select the network where the mint takes place.
2. Select your Mint Mode from the dropdown:
   - **Mint + Transfer**: Mints the NFT and immediately sweeps it directly to your safe wallet in the same transaction.
   - **Mint only**: Executes the mint function on the contract without sweeping the new NFT out of your wallet.
3. Enter the Mint Contract Address and paste the Mint Calldata (hex). The app automatically checks the network to verify that the contract exists.
4. If the mint has a mint fee in native currency, enter the amount (like `0.01` or hex `0x...`) in the Mint Price field. Your sponsor wallet pays this fee for you. For free mints, leave this blank.
5. In Mint + Transfer mode, if the NFT collection is the same contract as the mint contract, leave the NFT Contract Address blank. If the collection is a separate contract from the minting contract, enter the NFT contract address.
6. Native tokens in your compromised wallet are swept automatically by default. In Mint + Transfer mode, you can also select existing NFTs in your wallet to sweep them in the same transaction alongside your mint.
7. Click **Review** to enter your compromised private key, then click **Rescue** to execute the mint and sweep the NFTs directly into your safe wallet.

### 5.2 Finding Token IDs

Because an NFT's token ID cannot be known before minting, our contract discovers it on-chain during execution using three methods:
- Receiver hooks: Intercepts standard safe mint callbacks (`onERC721Received` and `onERC1155Received`) to capture and redirect the token ID as it is minted.
- Return data: Decodes the token ID directly from the mint function's return value.
- Supply queries: Checks `nextTokenId()` or `totalSupply()` on the collection contract to determine the new token ID.

If an NFT contract does not support any of these methods, the minted token ID is read from the transaction receipt and an automatic follow-up rescue broadcasts immediately to recover it. However, because this requires a separate transaction, there is a brief on-chain window before it confirms where the NFT could be intercepted, even though the follow-up broadcasts immediately.

### 5.3 Fees

- NFTs (ERC-721 & ERC-1155): Rescued with 0% protocol fee. 100% of your minted and rescued NFTs go directly to your safe wallet.
- Native Currency: The standard 15% recovery fee applies only if native tokens are swept from the wallet.

---

## 6. DeFi Lending Rescue (`/lending`)

The Lending page recovers collateral trapped in lending markets (such as Aave v3, Morpho Blue, Moonwell, and Curvance), including active positions with debt and idle deposits without debt.

### 6.1 How It Works

1. Enter your compromised wallet address and select your network(s). You can select multiple networks to scan positions across different chains at the same time.
2. The scanner automatically detects your lending positions, showing your deposited collateral and any outstanding debt.
3. Select the positions you want to recover. You can select multiple positions on the same network, or positions across different networks.
4. Enter your clean safe destination address.
5. Click **Review**. The app verifies that there is enough on-chain flash loan liquidity to borrow your debt tokens, calculates swap routes (if collateral differs from debt), and verifies sponsor gas.
6. If a swap is needed, you can adjust the slippage buffer (default is 1%). This buffer sets how much extra collateral is budgeted for the swap to guarantee the flash loan is fully repaid even if prices shift. Any leftover tokens from the swap are safely swept to your safe destination wallet.
7. Click **Review** to enter your compromised private key, then click **Rescue** to execute the recovery and sweep your net collateral directly into your safe wallet.

### 6.2 Debt Repayment, Swaps & Idle Deposits

- When a position has debt, an uncollateralized flash loan borrows the debt amount to repay the lending market and unlock your collateral in one atomic transaction, without needing to deposit funds into the compromised wallet.
- If your collateral is the same token as your borrowed debt, no swap takes place. The flash loan is repaid directly from the unlocked collateral.
- If your collateral differs from your borrowed debt (such as WETH collateral with USDC debt), an in-lock swap automatically converts just enough collateral to repay the flash loan.
- If a position requires a swap but no direct or multi-hop swap route is found, the review modal flags that no route was found, as the debt cannot be settled without an available swap route.
- If you have collateral deposited with zero debt (an idle position), no flash loan or swap is needed. It directly withdraws and sweeps your collateral to your safe wallet.
- If an individual debt token lacks on-chain flash loan liquidity, the review modal highlights that position so you can deselect it and continue rescuing your other positions.

### 6.3 Fees

- A 15% recovery fee is deducted on-chain directly from the net recovered collateral upon successful rescue. You never pay upfront fees.

---

## 7. Sponsor Wallet

The sponsor wallet is a clean burner wallet generated directly in your browser. It pays the gas fees for all your rescue transactions so your compromised wallet never needs to hold native gas.

### 7.1 How It Works

1. Click **Generate Sponsor Wallet** at the top of any rescue page to create your sponsor wallet in one click.
2. The wallet and its private key are generated client-side in your browser and encrypted locally using 256-bit AES-GCM.
3. Deposit a small amount of native gas (like ETH, POL, or BNB) into your sponsor wallet on the chain you want to rescue.
4. When you execute a rescue, the sponsor wallet broadcasts the transaction and pays the gas fees on behalf of your compromised wallet.

### 7.2 Managing Keys & Funds

- Click the three dots on the sponsor card and select **Export Phrase / Key** to view and copy your private key or 12-word seed phrase. You can also import this key into any external wallet app.
- You can withdraw any unused gas balance from your sponsor wallet back to any safe address directly from the app at any time.
- Click the three dots on the sponsor card and select **Reset Wallet** to clear and generate a fresh sponsor wallet whenever you want.

### 7.3 Tips & Safety

- Only deposit the gas needed to cover your planned rescues.
- Do not use the sponsor wallet as a primary wallet or to store personal savings. It is designed solely to pay gas fees for your rescues.

---

## 8. Referral Program (`/refer`)

The Referral page lets you generate a personal referral link to earn on-chain commissions by helping others recover their funds from compromised wallets.

### 8.1 How It Works

1. Go to the **Refer & Earn** page (`/refer`).
2. Enter your payout address (any clean EVM wallet where you want to receive commissions).
3. Copy your unique referral link or save the QR code to share.
4. When someone opens your link, your payout address is remembered in their browser for their rescues.
5. When they complete a successful rescue, 6% of the recovered assets are sent directly to your payout address in the exact same transaction.

### 8.2 Commissions & Payouts

- The standard recovery fee on tokens and native assets is 15%. When a rescue happens through a referral link, 6% goes to the referrer and 9% goes to the protocol. The recovering user always receives their full 85% net assets whether they use a referral link or not.
- NFTs are rescued with 0% protocol fee, so no referral fee applies to NFT rescues.
- Payouts are instant and on-chain. There are no claim portals, points, or withdrawal delays—commissions land in your wallet the moment the rescue transaction confirms.
- Commission applies to the referred wallet's first successful rescue on each supported network. For example, if a user rescues on Ethereum, you receive commission on Ethereum. If they also rescue on Monad, you receive commission on Monad. Any subsequent rescues by the same wallet on the same network do not pay a commission.
- Anti-self-referral checks prevent an account from earning commissions on its own rescues. The referrer address cannot be the compromised wallet, the sponsor wallet, or the safe destination address.
- If a referrer payout address is a smart contract that rejects the transfer, the commission routes to the protocol so the rescue transaction never fails. Always use a standard wallet address (EOA) so you never miss out on payouts.
- If redirecting that 6% cut to the protocol treasury also fails, the contract automatically adds the unpayable 6% back to the user's sweep amount, delivering 91% net recovery to the safe destination rather than leaving those tokens behind in the compromised wallet.

---

## 9. Fees & How They Work

RescueKit collects fees on-chain during execution. Understanding how fees are charged and the order in which transfers execute ensures your assets reach your safe wallet smoothly.

### 9.1 Fee Breakdown by Asset

- Tokens (ERC-20) and native gas assets incur a 15% protocol recovery fee. The recovering user receives 85% net assets delivered to their safe destination wallet.
- NFTs (ERC-721 and ERC-1155) have a 0% protocol fee. Rescued NFTs are delivered 100% intact to your safe destination wallet with no fee deduction.
- On active DeFi lending positions, the 15% fee applies only to the net collateral recovered after outstanding debt and flash loans are fully repaid.
- Fees are collected in kind directly in the recovered asset with no external price feeds or oracles. For example, recovering 1,000 USDC delivers 850 USDC to your safe wallet and 150 USDC to the protocol.
- Gas fees are separate from protocol fees and are paid by your sponsor wallet in native network currency directly to blockchain validators.

### 9.2 Referral Split

- When a rescue is executed through a referral link, the 15% fee is split on-chain: 6% goes to the referrer and 9% goes to the protocol treasury. The recovering user receives their full 85% net assets.
- If no referral link is used, or if anti-self-referral checks are triggered, the entire 15% fee goes to the protocol treasury.
- If the referrer payout address cannot receive funds (for example, a contract that rejects the transfer), the 6% cut redirects to the protocol treasury.
- If redirecting that 6% cut to the treasury also fails, the contract automatically adds the unpayable 6% back to the user's sweep amount, delivering 91% net recovery to the safe destination rather than leaving those tokens behind in the compromised wallet.

### 9.3 Transfer Order & Safe Destination Requirements

- Protocol fees are collected on-chain before the remaining balance is dispatched to your safe destination wallet. This fee-first order prevents malicious contracts from engineering revert traps on the protocol treasury to rescue assets for free.
- Always use a clean standard wallet address (EOA) or a verified Safe multisig as your safe destination. If a safe destination cannot receive transfers (such as a contract without a receive function for native currency, or an address blacklisted by centralized tokens like USDC or USDT), that transfer fails on-chain (`CallFailed`) while the fee remains collected.
- When a destination transfer fails, the remaining 85% stays behind in the compromised wallet where it remains vulnerable to sweeper bots. In that scenario, you will need to execute a new rescue with a clean, working safe wallet to recover the remaining funds, resulting in the 15% fee being charged once again on that remaining balance.
- If the entire transaction reverts on-chain, all state changes roll back completely via standard EVM execution and zero fees are charged.

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

## 8. Transaction Construction, Gas Dynamics & Type-4 Lifecycle

### 8.1 Gas Budgeting Architecture

Executing an EIP-7702 asset rescue requires coordinating two fundamental parameters:

1. **Gas Units (Limit):** The computational capacity budgeted to complete all batch steps.
2. **Gwei Price (Per-Unit Cost):** The live market fee (base fee + priority tip) paid to network validators/sequencers.

### 8.2 Analytical Gas Budgeting & RPC Resilience

Public RPC endpoints frequently fail to accurately simulate gas limits for EIP-7702 transactions when evaluating accounts that are not yet delegated on-chain. To eliminate transaction reverts caused by faulty RPC simulations, RescueKit implements an analytical gas estimation engine:

- **Operation-Specific Scaling:** Dynamically scales gas limits based on the operational complexity of the rescue—accounting for the exact number of transfer calls, claim proofs, or lending position repayments.
- **Complex Flow Allocations:** Heavy execution paths (such as multi-hop flash loan repayments and debt liquidations) receive conservative gas unit buffers to ensure safe execution under unexpected state conditions.
- **Deterministic Reliability:** Because this budget is computed analytically, transactions can be reliably signed and broadcast even when public RPC node estimators produce unreliable results.

### 8.3 Dynamic Fee Pricing & Priority Inclusion

To prevent MEV frontrunning and ensure immediate block inclusion:

- **Adaptive Base Buffering:** Incorporates dynamic base fee headrooms to absorb rapid block-to-block fee spikes without dropping from the block builder queue.
- **Competitive Priority Bidding:** Scales priority tips according to the network's specific mempool model—applying focused inclusion bids on sequencer L2s and competitive priority pricing on competitive public mempools.
- **Fail-Safe Price Guards:** If an RPC endpoint returns zero or invalid gas fee data, the client halts broadcast automatically to protect the user from broadcasting dead transactions.

### 8.4 Receipt Polling, Confirmation Timeouts & Nonce Coordination

- **Adaptive Receipt Polling:** Optimizes polling intervals based on network block times (from ultra-fast sub-second parallel chains to standard L2 block times).
- **Sponsor Nonce Tracking:** Synchronizes against pending transaction counts to allow rapid consecutive recoveries without nonce collisions.

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

## 12. Smart Contracts & Technical Interface Reference

### 12.1 `SponsorableBatchExecutor` Contract Specification

- **Contract Name:** `SponsorableBatchExecutor`
- **Solidity Version:** `^0.8.20` (Deployed via `0.8.37`)
- **Canonical Address:** `0x0000000004C9B572E8aB03C7A7377AaadEfd3502`
- **Compiler Optimizations:** Enabled (200 runs)

### 12.2 Execution Modes & Protocol Capabilities

RescueKit implements modular execution modes conforming to the ERC-7821 standard:

| Mode ID | Protocol Capability | Execution Description |
|---|---|---|
| **Mode 1** | Standard Single Batch | Executes sequential atomic calls without extra operational metadata. |
| **Mode 2** | Batch with Operational Data | Executes batch calls alongside contextual parameter decoding. |
| **Mode 3** | NFT Sponsor Rescue | Sweeps ERC-721 and ERC-1155 tokens directly to safety with 0% protocol fee. |
| **Mode 4** | Flash Loan Lending Batch | Manages flash loan borrowing, debt payoff, collateral redemption, and repayment. |
| **Mode 6** | Direct Claim Batch | Claims airdrops or vesting tokens and sweeps net balances in one step. |
| **Mode 7** | ERC-721 Mint & Forward | Intercepts NFT mint callbacks and redirects tokens to the safe destination. |
| **Mode 8** | ERC-1155 Mint & Forward | Intercepts semi-fungible mint callbacks and forwards assets atomically. |
| **Mode 9** | Multi-Claim Batch | Processes multi-protocol claim collections with granular error isolation. |

### 12.3 Contract Administrative Controls & Upgradability

- **Non-Upgradable:** Contract logic is immutable. There are no proxies or admin implementation pointers.
- **Fee Configuration:** Admin functions (`setFeeBps`, `setAffiliateCutBps`, `setFeeRecipient`) can only adjust fee basis points within hardcoded bounds:
  - Maximum fee limit: `1500` BPS (15.00%).
  - Enforced pause: Emergency pause mechanism protects against protocol-level zero-day vulnerabilities.

### 12.4 Interface Identifiers (`supportsInterface`)

The contract returns `true` for:
- `0x01ffc9a7`: ERC-165 Standard Interface Detection
- `0x150b7a02`: ERC-721 Token Receiver (`onERC721Received`)
- `0x4e2312e0`: ERC-1155 Token Receiver (`onERC1155Received` / `onERC1155BatchReceived`)

---

## 13. User Input Field Reference & Validation Matrix

| Field Name | Format | Required | Validation Rule | Invalid Behavior | Empty Behavior |
|---|---|---|---|---|---|
| **Compromised Key** | 64-char Hex (`0x...`) | Yes | Must derive valid secp256k1 public address | Rejects with "Invalid private key format" | Submit button disabled |
| **Safe Destination** | 40-char Hex Address | Yes | Must pass EIP-55 checksum validation | Rejects with "Invalid destination address" | Submit button disabled |
| **Sponsor Wallet** | Connected Web3 Provider | Yes | Must have native balance $\ge$ gas estimate | Surfaces "Insufficient sponsor balance" | Blocks review modal |
| **Token Address** | 40-char Hex Address | Optional | Must be deployed contract on active network | Displays "Contract not found on chain" | Ignored |
| **NFT Token ID** | Unsigned Integer (`uint256`) | Optional | Must be owned by compromised address | Excluded during scan | Ignored |
| **Claim Calldata** | Raw ABI Hex Bytes | Optional | Must decode against target distributor ABI | Reverts simulation: "Invalid claim payload" | Bypasses claim step |

---

## 14. Error Reference, Contract Reverts & Failure Resolutions

### 14.1 Smart Contract Custom Revert Errors

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

### 14.2 Client-Side Validation & Simulation Rejections

- **`"Insufficient sponsor balance"`**
  - *Cause:* Sponsor wallet does not hold enough native gas tokens to cover the worst-case gas limit.
  - *Resolution:* Deposit additional native gas tokens into the Sponsor Wallet.
- **`"Compromised account nonce mismatch"`**
  - *Cause:* On-chain nonce changed between authorization signing and broadcast.
  - *Resolution:* Re-sign the authorization tuple with the updated nonce.

---

## 15. The "What Happens If..." Real-World Edge Case Directory

### 15.1 Asset & State Alterations

- **What if assets disappear before execution?**
  - If a sweeper bot moves an ERC-20 token before RescueKit's transaction is included, the `balanceOf` query inside the contract returns `0`. The contract skips the zero-balance transfer and continues sweeping remaining assets without reverting.
- **What if an airdrop claim expires?**
  - If the distributor contract reverts because the claim window closed, the atomic batch reverts. Mode 9 multi-claim batches can be configured with soft-failure flags to bypass reverted sub-claims.

### 15.2 Transaction Execution & Network Failures

- **What if the lending repayment fails?**
  - If collateral cannot satisfy debt repayment or flash loan repayment fails, the entire transaction reverts atomically. No collateral is lost, and the loan remains in its prior state.
- **What if the RPC endpoint goes offline during broadcast?**
  - The client automatically retries against configured fallback RPC endpoints (`fallbackRpcs`) without requiring re-signing.
- **What if I close or refresh the browser tab?**
  - If broadcast has already occurred, the transaction executes on-chain independently. If broadcast has not occurred, ephemeral memory is cleared and no transaction is sent.

### 15.3 Protocol & Financial Edge Cases

- **What if the wallet is rescued twice?**
  - The second rescue executes normally if new assets have arrived. If no assets exist, zero-value calls execute harmlessly, costing only sponsor gas.
- **What if the affiliate payment fails?**
  - Outbound affiliate transfer reverts are caught. The commission is routed to the treasury and the user's asset recovery completes without interruption.

---

## 16. Architectural Comparisons & Industry Alternatives

| Feature | RescueKit (EIP-7702) | Flashbots Bundles | Traditional Sweeping | Smart Contract Wallets (ERC-4337) |
|---|---|---|---|---|
| **Compromised Wallet Gas Funding** | **0 ETH (Zero)** | 0 ETH (Miner Tip) | **Requires Gas Funding** | 0 ETH (Paymaster) |
| **Sweeper Bot Exploitation Risk** | **Zero Funding Gap** | Low (Mempool private) | **Extreme (Immediate theft)** | Zero Funding Gap |
| **Supported Chains** | **19 EVM Mainnets** | Ethereum Only | Any | Network Dependent |
| **DeFi Debt Liquidation Support** | **Built-in Flash Loans** | Complex manual setup | Not supported | Requires custom paymaster |
| **Account Ownership Changes** | **None (Ephemeral)** | None | None | Requires account migration |
| **NFT Support** | **100% Free (0% Fee)** | Manual bundle cost | Manual transfer | Paymaster gas sponsor |

---

## 17. Comprehensive FAQ

#### Q: How does RescueKit bypass sweeper bots?
A: Sweeper bots can only steal assets if they have gas to transfer or if gas is deposited into the compromised account. RescueKit sponsors transactions externally via EIP-7702. Because the compromised account never receives gas, the bot has nothing to take.

#### Q: Is my private key uploaded to a server?
A: No. Private keys are used strictly inside your browser's local memory to sign an EIP-7702 authorization tuple. They are never transmitted over the internet.

#### Q: What chains are supported?
A: All 19 major EVM production mainnets listed in Section 5.1 are fully supported.

#### Q: What fee does RescueKit charge?
A: A 15% protocol fee is deducted on-chain exclusively from recovered fungible assets and native currency upon successful execution. NFTs are rescued completely free of protocol fees.

---

## 18. Protocol Limitations & Known Issues

1. **Non-Empty Calldata Native Call Fee Bypass:** In current contract builds, non-empty data native calls can theoretically bypass the fee split if constructed manually outside the official builder. The official builder always routes through the auto-split path.
2. **Public Mempool Frontrunning on BSC/Polygon:** On networks lacking sequencer privacy, high-priority public mempool transactions can theoretically be observed by sophisticated MEV bots capable of parsing 7702 calldata. Private relays are enforced for automated engines on these chains.
3. **RPC 7702 Simulation Inconsistencies:** Some third-party RPC providers do not yet support `eth_estimateGas` simulations with an `authorizationList`. The protocol bypasses this using deterministic fallback formulas.

---

## 19. Technical Glossary

- **EIP-7702:** Ethereum Improvement Proposal enabling EOAs to temporarily designate smart contract execution code for a single transaction.
- **ERC-7821:** Minimal batch execution interface standard defining `execute(bytes32 mode, bytes executionData)`.
- **Sponsor Wallet:** An uncompromised secondary account used exclusively to pay native network gas fees for the recovery transaction.
- **Sweeper Bot:** An automated script monitoring compromised accounts to instantly frontrun and steal deposited gas tokens.
- **Type-4 Transaction:** An EVM transaction carrying an EIP-7702 `authorizationList`.

---

## 20. Documentation Fact & Verification Ledger

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

## 21. Version History & Protocol Changelog

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
