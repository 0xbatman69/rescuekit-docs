# Documentation

---

## Table of Contents

1. [Introduction](#1-introduction)
   - 1.1 How Sweeper Bots Work
   - 1.2 How EIP-7702 Solves This
2. [Quick Start](#2-quick-start)
   - 2.1 What You Need
   - 2.2 How to Rescue
3. [Sponsor Wallet](#3-sponsor-wallet)
   - 3.1 How It Works
   - 3.2 Managing Keys & Funds
   - 3.3 Tips & Safety
4. [Token & NFT Rescue (`/transfer`)](#4-token--nft-rescue-transfer)
   - 4.1 How It Works
   - 4.2 Fees
5. [NFT Mint Rescue (`/mint`)](#5-nft-mint-rescue-mint)
   - 5.1 How It Works
   - 5.2 Finding Token IDs
   - 5.3 Fees
6. [Airdrop & Claims Rescue (`/claim`)](#6-airdrop--claims-rescue-claim)
   - 6.1 How It Works
   - 6.2 Follow-Up Rescues
   - 6.3 Fees
7. [DeFi Lending Rescue (`/lending`)](#7-defi-lending-rescue-lending)
   - 7.1 How It Works
   - 7.2 Debt Repayment & Idle Deposits
   - 7.3 Swaps & Routing
   - 7.4 Fees
8. [Referral Program (`/refer`)](#8-referral-program-refer)
   - 8.1 How It Works
   - 8.2 Commissions & Payouts
9. [Fees & How They Work](#9-fees--how-they-work)
   - 9.1 Fee Breakdown by Asset
   - 9.2 Referral Split
   - 9.3 Transfer Order & Safe Destination Requirements
10. [Custom RPC & Transaction Broadcasting](#10-custom-rpc--transaction-broadcasting)
    - 10.1 Custom RPC
    - 10.2 Transaction Broadcasting
11. [Supported Networks](#11-supported-networks)
12. [What RescueKit Can & Cannot Do](#12-what-rescuekit-can--cannot-do)
    - 12.1 What RescueKit Can Do
    - 12.2 What RescueKit Cannot Do

---

## 1. Introduction

RescueKit helps you recover trapped funds from hacked or compromised EVM wallets. By using EIP-7702, a separate clean wallet pays the gas fees so automated sweeper bots never get a chance to steal your gas. You can rescue tokens, NFTs, unclaimed airdrops, and DeFi lending positions directly into a safe wallet in a single transaction.

### 1.1 How Sweeper Bots Work

When an account's private key or seed phrase leaks, automated MEV sweeper bots monitor the address across public transaction mempools and block builders. Sweeper bots maintain persistent RPC subscriptions listening for inbound transfers.

1. User sends gas to the compromised wallet.
2. Sweeper bot detects the incoming transfer.
3. Bot drains the gas to the attacker wallet within milliseconds.
4. The user never gets a chance to broadcast a transaction to move their funds.
5. Trapped assets remain stuck.

Under this hostile condition, traditional transactions (`eth_sendRawTransaction`) fail because the account owner cannot fund the account with the native gas required to broadcast any transaction. Any gas sent to the address is stolen within milliseconds by the bot.

### 1.2 How EIP-7702 Solves This

EIP-7702 allows an ordinary wallet (EOA) to temporarily delegate its execution to a smart contract without changing account ownership. By signing an authorization designating the RescueKit contract (`0x0000000004C9B572E8aB03C7A7377AaadEfd3502`), the wallet gains the ability to execute batch rescue operations.

Crucially, EIP-7702 transactions can be sponsored by a separate, clean account (the "Sponsor Wallet"). The sponsor wallet pays 100% of the gas needed to broadcast and execute the rescue. Because the compromised account never receives or holds native gas, sweeper bots never get a chance to trigger.


---

## 2. Quick Start

### 2.1 What You Need

- Compromised wallet address and its private key.
- A clean, uncompromised safe destination address.
- A sponsor wallet created on RescueKit, funded with enough gas for the network fee.

### 2.2 How to Rescue

1. Create a sponsor wallet in the app and deposit gas into it.
2. Enter your compromised wallet address and select your network.
3. Select or enter the details to rescue.
4. Enter your clean safe destination address.
5. Click **Review** and enter your compromised private key.
6. Click **Rescue** to execute the rescue transaction.

---

## 3. Sponsor Wallet

The sponsor wallet is a clean burner wallet generated directly in your browser. It pays the gas fees for all your rescue transactions so your compromised wallet never needs to hold native gas.

### 3.1 How It Works

1. Click **Generate Sponsor Wallet** at the top of any rescue page to create your sponsor wallet in one click.
2. The wallet and its private key are generated client-side in your browser and encrypted locally using 256-bit AES-GCM.
3. Deposit a small amount of native gas (like ETH, POL, or BNB) into your sponsor wallet on the chain you want to rescue.
4. When you execute a rescue, the sponsor wallet broadcasts the transaction and pays the gas fees on behalf of your compromised wallet.

### 3.2 Managing Keys & Funds

- Click the three dots on the sponsor card and select **Export Phrase / Key** to view and copy your private key or 12-word seed phrase. You can also import this key into any external wallet app.
- You can withdraw any unused gas balance from your sponsor wallet back to any safe address directly from the app at any time.
- Click the three dots on the sponsor card and select **Reset Wallet** to clear and generate a fresh sponsor wallet whenever you want.

### 3.3 Tips & Safety

- Only deposit the gas needed to cover your planned rescues.
- Do not use the sponsor wallet as a primary wallet or to store personal savings. It is designed solely to pay gas fees for your rescues.

---

## 4. Token & NFT Rescue (`/transfer`)

The Transfer page recovers ERC-20 tokens, native coins, and NFTs already sitting in your compromised wallet.

### 4.1 How It Works

1. Enter your compromised address and select your networks. The scanner checks for tokens, native coins, and NFTs across all selected chains at the same time.
2. You can rescue multiple tokens, multiple NFTs, or both combined at once. If you selected multiple networks, you can rescue across all of them at the same time with one click.
3. If a token or NFT does not appear automatically, you can manually enter the contract address (and token ID for NFTs) to include it.
4. Once confirmed via **Review** and **Rescue**, the assets are swept directly into your safe destination wallet.
5. If an individual token or NFT transfer fails (for example, non-transferrable token or nft), it simply skips that token and continues rescuing the rest of your assets without canceling the entire transaction.

### 4.2 Fees

- For tokens and native currency, a 15% recovery fee is deducted on-chain directly from the recovered amount during the transfer.
- NFTs (ERC-721 and ERC-1155) are rescued with a 0% protocol fee. 100% of your NFTs go directly to your safe wallet.

---

## 5. NFT Mint Rescue (`/mint`)

The Mint page lets you mint NFTs (ERC-721 or ERC-1155) from an eligible or allowlisted compromised wallet, either sweeping them straight to your safe wallet in the same transaction or executing the mint on its own.

### 5.1 How It Works

1. Enter your compromised wallet address, safe destination address, and select the network where the mint takes place.
2. Select your Mint Mode from the dropdown.
   - Mint only: executes the mint function on the contract without sweeping the new NFT out of your wallet.
   - Mint + Transfer: mints the NFT and immediately sweeps it directly to your safe wallet in the same transaction.
3. Enter the Mint Contract Address and paste the Mint Calldata (hex). The app automatically checks the network to verify that the contract exists.
4. If the mint has a mint fee in native currency, enter the amount (like `0.01` or hex `0x...`) in the Mint Price field. Your sponsor wallet pays this fee for you. For free mints, leave this blank.
5. In Mint + Transfer mode, if the NFT collection is the same contract as the mint contract, leave the NFT Contract Address blank. If the collection is a separate contract from the minting contract, enter the NFT contract address.
6. Native tokens in your compromised wallet are swept automatically by default. In Mint + Transfer mode, you can also select existing NFTs in your wallet to sweep them in the same transaction alongside your mint.
7. Click **Review** to enter your compromised private key, then click **Rescue** to execute the mint and sweep the NFTs directly into your safe wallet.

### 5.2 Finding Token IDs

Because an NFT's token ID cannot be known before minting, our contract discovers it on-chain during execution using three methods.
- Receiver hooks intercept standard safe mint callbacks (`onERC721Received` and `onERC1155Received`) to capture and redirect the token ID as it is minted.
- Return data decodes the token ID directly from the mint function's return value.
- Supply queries check `nextTokenId()` or `totalSupply()` on the collection contract to determine the new token ID.

If an NFT contract does not support any of these methods, the minted token ID is read from the transaction receipt and an automatic follow-up rescue broadcasts immediately to recover it. However, because this requires a separate transaction, there is a brief on-chain window before it confirms where the NFT could be intercepted, even though the follow-up broadcasts immediately.

### 5.3 Fees

- NFTs (ERC-721 and ERC-1155) are rescued with a 0% protocol fee. 100% of your minted and rescued NFTs go directly to your safe wallet.
- The standard 15% recovery fee applies only if native currency is swept from the wallet.

---

## 6. Airdrop & Claims Rescue (`/claim`)

The Claims page recovers claimable tokens from contracts like airdrops, staking, vesting, and more.

### 6.1 How It Works

1. Enter your compromised address and select your network.
2. Enter the claim contract address. The app automatically checks the network to verify that the contract exists. If the contract supports common claim functions, it is detected automatically, or you can paste your claim calldata.
3. If the claim requires a native fee, enter the amount (like `0.01` or hex `0x...`) in the Value field. Your sponsor wallet pays this fee for you. If no fee is required, leave it blank.
4. The payout token is usually detected and filled in automatically, but always verify that the address is correct (or enter it manually if not detected). Native tokens are swept automatically by default, and you can also sweep existing tokens already sitting in your wallet by adding their contract addresses.
5. If a claim rewards multiple tokens at once, you can add extra token addresses to sweep all reward tokens together in one transaction.
6. You can add multiple claim boxes to execute different claims together at the same time.
7. Click **Review** to enter your compromised private key, then click **Rescue** to sweep your tokens directly into your safe wallet.

### 6.2 Follow-Up Rescues

If an individual claim fails (for example, if it expired), it simply skips that claim and continues rescuing your other claims without canceling the entire transaction.

Always enter the correct payout tokens so everything sweeps in the first transaction. If a contract transfers extra or unexpected tokens that you did not enter, an automatic follow-up rescue broadcasts immediately to recover them. However, because this requires a separate transaction, there is a brief on-chain window where tokens could be intercepted. While the follow-up rescue is fast, there is no guarantee in that window, so always double-check your token addresses.

### 6.3 Fees

- A 15% recovery fee is deducted on-chain directly from the claimed tokens during the rescue transaction.

---

## 7. DeFi Lending Rescue (`/lending`)

The Lending page recovers collateral trapped in lending markets (such as Aave v3, Morpho Blue, Moonwell, and Curvance), including active positions with debt and idle deposits without debt.

### 7.1 How It Works

1. Enter your compromised wallet address and select your network(s). You can select multiple networks to scan positions across different chains at the same time.
2. The scanner automatically detects your lending positions, showing your deposited collateral and any outstanding debt.
3. Select the positions you want to recover. You can select multiple positions on the same network, or positions across different networks.
4. Enter your clean safe destination address.
5. Click **Review**. The app verifies that there is enough on-chain flash loan liquidity to borrow your debt tokens, calculates swap routes (if collateral differs from debt), and verifies sponsor gas.
6. If a swap is needed, the review modal displays **Est. Output (Debt)** for the tokens needed to cover your debt at current DEX prices. You can adjust the slippage buffer (default is 1%). Rather than reducing your received tokens, this buffer budgets extra collateral for the swap to guarantee the flash loan is fully repaid even if prices shift, with any leftover tokens safely swept to your safe destination wallet.
7. Click **Review** to enter your compromised private key, then click **Rescue** to execute the recovery and sweep your net collateral directly into your safe wallet.

### 7.2 Debt Repayment & Idle Deposits

- When a position has debt, an uncollateralized flash loan borrows the debt amount to repay the lending market and unlock your collateral in one atomic transaction, without needing to deposit funds into the compromised wallet.
- If your collateral is the same token as your borrowed debt, no swap takes place. The flash loan is repaid directly from the unlocked collateral.
- If you have collateral deposited with zero debt (an idle position), no flash loan or swap is needed. It directly withdraws and sweeps your collateral to your safe wallet.
- If an individual debt token lacks on-chain flash loan liquidity, the review modal highlights that position so you can deselect it and continue rescuing your other positions.

### 7.3 Swaps & Routing

- Swaps route through major DEXes on each network (such as Uniswap and PancakeSwap).
- If your collateral differs from your borrowed debt, direct single-hop and 2-hop routes across fee tiers are compared automatically, selecting whichever route yields the highest output and lowest price impact. If only one route exists, it uses that route. If no route is found, the review modal flags that the position cannot be settled.
- On networks supporting Uniswap v4, swaps strictly route through canonical, hookless pools (`hooks == address(0)`). Pools with custom or third-party hooks are never used.
- Only the minimum slice of collateral needed to clear the loan is sold, all remaining collateral is swept directly to safety. If price slippage prevents the swap from fully covering the debt down to the last wei, the entire transaction atomically reverts on-chain so zero collateral is ever lost.

### 7.4 Supported Protocols by Chain

Supported lending protocols across each network today. This list is kept updated as new protocols are added.

| Network | Supported Protocols |
|---|---|
| Ethereum | Aave v3, Morpho Blue, Compound v3, Moonwell |
| Base | Aave v3, Morpho Blue, Compound v3, Moonwell |
| BNB Chain | Venus Protocol, Aave v3 |
| Arbitrum One | Aave v3, Morpho Blue, Compound v3 |
| Arc | Morpho Blue |
| Polygon | Aave v3, Morpho Blue, Compound v3 |
| Optimism | Aave v3, Morpho Blue, Compound v3, Moonwell |
| Monad | Aave v3, Morpho Blue, Curvance |
| Sonic | Aave v3 |
| Robinhood | Morpho Blue |
| Berachain | Morpho Blue (Bend) |
| MegaETH | Aave v3 |
| Linea | ZeroLend, Compound v3 |
| Ink | Aave v3 |
| Unichain | Compound v3, Morpho Blue |
| Sei | — |
| World Chain | Morpho Blue |
| Somnia | — |
| Plasma | Aave v3 |
| Plume | — |

### 7.5 Fees

- A 15% recovery fee is deducted on-chain directly from the net recovered collateral during the rescue transaction.

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
- Payouts are instant and on-chain. There are no claim portals, points, or withdrawal delays, and commissions land in your wallet the moment the rescue transaction confirms.
- Commission applies to the referred wallet's first successful rescue on each supported network. For example, if a user rescues on Ethereum, you receive commission on Ethereum. If they also rescue on Monad, you receive commission on Monad. Any subsequent rescues by the same wallet on the same network do not pay a commission.
- Anti-self-referral checks prevent an account from earning commissions on its own rescues. The referrer address cannot be the compromised wallet, the sponsor wallet, or the safe destination address.
- If a referrer payout address is a smart contract that rejects the transfer, the commission routes to the protocol so the rescue transaction never fails. Always use a standard wallet address (EOA) so you never miss out on payouts.

---

## 9. Fees & How They Work

RescueKit collects fees on-chain during execution. Understanding how fees are charged and the order in which transfers execute ensures your assets reach your safe wallet smoothly.

### 9.1 Fee Breakdown by Asset

- Tokens (ERC-20) and native gas assets incur a 15% protocol recovery fee. The recovering user receives 85% net assets delivered to their safe destination wallet.
- NFTs (ERC-721 and ERC-1155) have a 0% protocol fee. Rescued NFTs are delivered 100% intact to your safe destination wallet with no fee deduction.
- On DeFi lending positions, the 15% fee applies only to the net collateral recovered (after any outstanding debt is repaid).
- Fees are collected in kind directly in the recovered asset with no external price feeds or oracles. For example, recovering 1,000 USDC delivers 850 USDC to your safe wallet and 150 USDC to the protocol.
- Gas fees are separate from protocol fees and are paid by your sponsor wallet in native network currency directly to blockchain validators.

### 9.2 Referral Split

- When a rescue is executed through a referral link, the 15% fee is split on-chain, where 6% goes to the referrer and 9% goes to the protocol treasury. The recovering user receives their full 85% net assets.
- If no referral link is used, or if anti-self-referral checks are triggered, the entire 15% fee goes to the protocol treasury.
- If the referrer payout address cannot receive funds (for example, a contract that rejects the transfer), the 6% cut redirects to the protocol treasury.
- If redirecting that 6% cut to the treasury also fails, the contract automatically adds the unpayable 6% back to the user's sweep amount, delivering 91% net recovery to the safe destination rather than leaving those tokens behind in the compromised wallet.

### 9.3 Transfer Order & Safe Destination Requirements

- Protocol fees are collected on-chain before the remaining balance is dispatched to your safe destination wallet. This fee-first order prevents malicious contracts from engineering revert traps on the protocol treasury to rescue assets for free.
- Always use a clean standard EOA wallet address or a verified Gnosis Safe as your safe destination. If a safe destination cannot receive transfers (such as a contract without a receive function for native currency, or an address blacklisted by centralized tokens like USDC or USDT), that transfer fails on-chain (`CallFailed`) while the fee remains collected.
- When a destination transfer fails, the remaining 85% stays behind in the compromised wallet where it remains vulnerable to sweeper bots. In that scenario, you will need to execute a new rescue with a clean, working safe wallet to recover the remaining funds, resulting in the 15% fee being charged once again on that remaining balance.
- To prevent this risk, the app verifies your safe destination address before execution:
  - **Standard EOA Wallets**. Recommended. Standard private key wallets have no custom code and cannot reject incoming transfers.
  - **Gnosis Safe**. Supported if your Safe is already deployed on the rescue network and accepts native coin. If your Safe is not deployed on the target chain, the rescue is blocked to protect your funds. You must deploy your Safe on that network first or use a standard EOA wallet instead.
  - **Other Contracts Blocked**. Custom smart contracts, smart accounts, and unverified addresses are blocked to prevent your funds from getting trapped.
- If the entire transaction reverts on-chain, all state changes roll back completely via standard EVM execution and zero protocol fees are charged.

---

## 10. Custom RPC & Transaction Broadcasting

### 10.1 Custom RPC

- You can configure custom RPC endpoints for any supported network by clicking the RPC settings icon next to the network selector.
- When you set a custom RPC for a network, every feature that makes RPC calls on that network uses your custom endpoint.
- Custom RPCs are especially useful for fast receipt polling during follow-up rescues. When an [**NFT Mint**](#5-nft-mint-rescue-mint) or [**Claim**](#6-airdrop--claims-rescue-claim) transaction confirms, the receipt logs are checked to detect minted token IDs or unswept tokens and broadcast the follow-up sweep as fast as possible.
- Public RPCs often have higher latency when polling receipts and logs compared to private custom endpoints.
- Custom endpoints also help avoid occasional public RPC rate limits.
- Before saving, the endpoint is tested for connectivity, latency, and chain ID match to ensure it belongs to the selected network.
- Custom URLs are stored locally in your browser, and you can reset any network back to its default public RPC at any time with a single click.

### 10.2 Transaction Broadcasting

- On Ethereum, transactions broadcast through MEV Blocker first, immediately falling back to Flashbots if needed.
- On BSC, transactions broadcast through 48 Club.
- If private relay broadcast fails on Ethereum or BSC, the rescue stops with an error instead of using public RPCs to keep transactions out of the public mempool.
- If you set a custom RPC for Ethereum or BSC, this private relay broadcast is not overwritten.
- On all other chains, transactions broadcast through your custom RPC, immediately falling back to default backup RPCs if it fails or times out.

---

## 11. Supported Networks

RescueKit is deployed and verified across 20 EVM mainnets. All deployments share the identical contract address `0x0000000004C9B572E8aB03C7A7377AaadEfd3502`.

| Network | ID | Native |
|---|---|---|
| Ethereum | 1 | ETH |
| Base | 8453 | ETH |
| BNB Chain | 56 | BNB |
| Arbitrum One | 42161 | ETH |
| Arc | 5042 | USDC |
| Polygon | 137 | POL |
| Optimism | 10 | ETH |
| Monad | 143 | MON |
| Sonic | 146 | S |
| Robinhood | 4663 | ETH |
| Berachain | 80094 | BERA |
| MegaETH | 4326 | ETH |
| Linea | 59144 | ETH |
| Ink | 57073 | ETH |
| Unichain | 130 | ETH |
| Sei | 1329 | SEI |
| World Chain | 480 | ETH |
| Somnia | 5031 | SOMI |
| Plasma | 9745 | XPL |
| Plume | 98866 | PLUME |

---

## 12. What RescueKit Can & Cannot Do

### 12.1 What RescueKit Can Do

- Rescue without funding your compromised wallet with gas (your sponsor wallet pays the gas).
- Rescue single or multiple ERC-20 tokens, native coins, and NFTs on any chain in a single transaction, or execute across multiple chains at the same time.
- Mint new NFTs and sweep existing NFTs to your safe wallet in a single transaction.
- Rescue single or multiple claims on any chain in a single transaction, or execute across multiple chains at the same time.
- Rescue trapped collateral from single or multiple DeFi lending positions (which includes idle positions and positions with debt, both) on any chain in a single transaction, or execute across multiple chains at the same time.

### 12.2 What RescueKit Cannot Do

- Recover funds that were already stolen or transferred out before your rescue.
- Recover funds if you enter an attacker-controlled or compromised safe destination address.
- Rescue non-transferable or soulbound tokens and NFTs that cannot be moved on-chain.
- Protect your sponsor wallet if you leak or compromise the sponsor wallet's own private key or seed phrase.
- Reverse transactions once confirmed on the blockchain.
- Guarantee newly minted NFTs are swept in the same transaction if the NFT contract does not support standard discovery methods.
