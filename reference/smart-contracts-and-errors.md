# Security Architecture & Smart Contract Reference

> Architectural security guarantees, non-custodial execution mechanics, and on-chain revert reference for `SponsorableBatchExecutor`.

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

## 2. On-Chain Contract Reverts

The table below catalogs every revert that can be thrown on-chain by `SponsorableBatchExecutor.sol`:

| Revert / Error | Trigger Condition | What Happened On-Chain | How to Resolve |
| :--- | :--- | :--- | :--- |
| `Unauthorized()` (`0x82b42900`) | Signature verification failed. | The recovered address from the batch authorization does not match `address(this)` (the compromised wallet). | 1) Verify that the private key corresponds to the compromised account; 2) When client-signing `batchDigest`, ensure you sign using EIP-191 personal sign (`account.signMessage({ message: { raw: digest } })`). |
| `"Flash loan request failed"` | A failure occurred during the flash loan operation. | The contract wraps the entire lending rescue (borrowing debt, repaying debt to the lending pool, withdrawing collateral, DEX swapping, and balance verification) inside a single callback. If **any** step fails—whether the pool lacks liquidity, debt repayment is rejected, collateral is locked, Uniswap slippage is exceeded, or residual balance is short—the contract catches the revert and emits this single error. | Because the contract aggregates all callback failures under this error, the exact root cause cannot be known from the revert string alone. Check the transaction or simulation trace manually (e.g. via Tenderly, Phalcon, or block explorer simulation) to see which internal step reverted. |
| `"Flash loan settlement failed"` | Insufficient debt token balance to repay the flash loan. | After repaying debt, withdrawing collateral, and swapping, the account balance was less than `loanAmount + premium`. | Check swap slippage or verify if enough collateral was sold to cover the borrowed debt plus fee. |
| `"Token claim failed"` | Single-claim transaction reverted. | The distributor or staking contract rejected the claim call (e.g. proof expired, already claimed, or ineligible). | Verify claim eligibility and proof data on the distributor contract. |
| `"NFT mint failed"` | NFT collection contract reverted during mint. | Mint preconditions were not met (e.g. allowlist proof invalid, public sale paused, or collection sold out). | Verify allowlist status, proof data, and sale phase on the collection's official mint interface. |
| `EnforcedPause()` | Protocol is temporarily paused. | Contract owner has paused protocol operations for emergency maintenance. | Check protocol announcements and retry once maintenance concludes. |
