# Security Model & Troubleshooting Directory

> Architectural security guarantees, non-custodial execution mechanics, and an on-chain troubleshooting directory for RescueKit users.

---

## 1. Security & Non-Custodial Architecture

RescueKit is engineered from the ground up to recover assets safely without ever taking custody of user funds or requiring gas to be sent to a compromised wallet.

### 100% Non-Custodial Execution (EIP-7702)
- **`address(this)` is the Victim's Account**: Under EIP-7702 ephemeral delegation, the execution bytecode (`SponsorableBatchExecutor`) runs directly within the context of the compromised wallet.
- **Zero Protocol Custody**: There is **no escrow contract, protocol vault, intermediate pool, or deposit contract**. Assets are never held by or routed through an external protocol contract before reaching safety.
- **Direct Atomic Transfers**: Rescued tokens, native currency, and NFTs transfer directly from the victim's account to `safeDestination` in the same atomic block.
- **Why Funds Cannot Be "Stuck in a Contract"**: Because no contract holds custody, user funds can never become frozen, locked, or stuck in a protocol contract. If a sweep transfer to `safeDestination` reverts (e.g. if the recipient address is invalid or unpayable), the assets simply remain where they started: in the victim's compromised wallet.

### Fee-First Settlement & The Safe Wallet Requirement
- **Anti-Exploit Protection**: Protocol fees (15%) are settled on-chain *before* the remaining balance is swept to `safeDestination`. This design prevents bad actors or malicious token contracts from blacklisting the treasury or constructing revert traps to siphon assets fee-free.
- **Clean Safe Wallet Requirement**: Because fees are settled prior to the final destination sweep, users must always supply a clean, unencumbered recovery wallet (a standard EOA or verified Safe multisig). If `safeDestination` rejects incoming transfers (e.g. an address blacklisted by centralized tokens like USDC/USDT, or a smart contract lacking a `receive()` function), the transaction confirms, the protocol fee remains collected, and the remaining 85% stays behind in the compromised wallet.
- **Double-Fallback Refund Safeguard (91.00% Net)**: If an affiliate payout fails (e.g. unoptimized contract receiver) and the subsequent redirect to the protocol treasury also fails, the unpayable 6.00% cut is automatically refunded to `safeDestination`. The victim receives **91.00%** net recovery rather than leaving residual value behind for attackers.
- **Atomic Rollback on Transaction Reverts**: If an entire recovery transaction fails and reverts on-chain (due to gas exhaustion, invalid signature, or flash loan failure), standard EVM execution rolls back all state changes atomically and zero fees are charged.

### Replay & Front-Running Protection
- **Chain & Account Binding**: Every recovery authorization is cryptographically bound to the current `block.chainid` and the compromised wallet's address. It is mathematically impossible to replay an authorization on another blockchain or against another wallet.
- **Mempool Protection**: The independent sponsor gas wallet pays all transaction fees via private RPC relays (Flashbots on Ethereum, 48 Club on BSC, standard sequencer pools on L2s), keeping the transaction hidden from public mempool sweeper bots until it is safely included in a block.

---

## 2. Complete Error Directory & Troubleshooting

### On-Chain Contract Reverts

| Error | Cause | What Happened | How to Resolve |
| :--- | :--- | :--- | :--- |
| `Unauthorized()` (`0x82b42900`) | Signature verification failed. | The recovered address from the batch authorization does not match the compromised wallet. | 1) Ensure you are entering the private key that matches the compromised address; 2) If using client libraries or the REST API, ensure you sign `batchDigest` using EIP-191 personal sign (`account.signMessage({ message: { raw: digest } })`). |
| `"Flash loan request failed"` | A failure occurred during the flash loan operation. | The contract wraps the entire lending rescue (borrowing debt, repaying debt to the lending pool, withdrawing collateral, swapping collateral on a DEX, and approving repayment) inside a single callback. If **any** step fails—whether the pool lacks liquidity, debt repayment is rejected, collateral is locked, Uniswap slippage is exceeded, or residual balance is short—the contract catches the revert and emits this single error. | Because the contract aggregates all callback failures under this error, the exact root cause cannot be known from the error string alone. Check the transaction or simulation trace manually (e.g. via Tenderly, Phalcon, or block explorer simulation) to see which internal step reverted. |
| `"Flash loan settlement failed"` | Insufficient balance to repay the flash loan. | After repaying debt, withdrawing collateral, and swapping, the account balance was less than `loanAmount + premium`. | Check swap slippage or verify if enough collateral was sold to cover the borrowed debt plus fee. |
| `"Token claim failed"` | Single-claim transaction reverted. | The airdrop or staking contract rejected the claim call (e.g. proof expired, already claimed, or ineligible). | Verify claim eligibility and proof data. If claiming multiple rewards, select multiple claims in the UI so a single failing claim does not revert the entire rescue. |
| `"NFT mint failed"` | NFT contract reverted during mint. | Mint preconditions were not met (e.g. allowlist proof invalid, public sale paused, or sold out). | Verify allowlist status, proof data, and sale phase on the collection's official mint interface. |
| `EnforcedPause()` | Protocol is temporarily paused. | Emergency maintenance is active. | Check protocol announcements and retry once maintenance concludes. |

### Application & Input Validation Errors

| Error String | Cause | How to Resolve |
| :--- | :--- | :--- |
| `"Destination address cannot be the same as the compromised address."` | Entered the compromised address as the safe recovery destination. | Enter an uncompromised, separate wallet address. |
| `"Destination is a token contract. Trapped funds will be lost forever."` | Entered an ERC-20 or ERC-721 token contract address as the recovery destination. | Supply a standard personal wallet (EOA) or Safe multisig address. |
| `"Insufficient sponsor gas. Please deposit at least X ETH..."` | The local sponsor burner wallet lacks enough native gas to broadcast the transaction. | Copy the sponsor address shown in the sidebar and deposit a small amount of native gas (e.g. 0.005 ETH/BNB/POL). |
| `"Invalid private key: must be 32 hex bytes."` | The private key string is not formatted as 64 hexadecimal characters. | Verify the private key string; remove any extra spaces or invalid characters. |
| `"Private key does not match compromised address."` | The derived address from the private key does not match the entered compromised wallet address. | Ensure you are pasting the private key belonging to the compromised wallet being rescued. |
| `"No balance found to rescue for the selected assets."` | Selected tokens or native currency have zero balance on-chain. | Check the selected chain and verify whether the assets were already moved or drained. |

### Network & RPC Failures

| Issue | Cause | How to Resolve |
| :--- | :--- | :--- |
| `Transaction confirmation timed out after 60000ms.` | Network congestion delayed block inclusion. | Check the transaction hash on the block explorer; the transaction will often confirm shortly once network congestion eases. |
| `Nonce too low` | The sponsor wallet broadcasted multiple transactions in rapid succession before the previous confirmed. | Wait 10–15 seconds for pending transactions to confirm, or click **Refresh Balances** in the sponsor card. |
| `Replacement transaction underpriced` | A replacement transaction did not increase gas fees by at least 10%. | Select the **Rapid** gas preset or increase `maxPriorityFeePerGas` before resubmitting. |
