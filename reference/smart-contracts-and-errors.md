# Smart Contract Specification & Error Directory

> Comprehensive technical reference for `SponsorableBatchExecutor.sol`, execution mode identifiers, cryptographic digests, and exhaustive error resolutions.

---

## 1. Smart Contract Architecture

The core execution contract for RescueKit is **`SponsorableBatchExecutor.sol`**. It implements the **ERC-7821** batch execution interface (`execute(bytes32 mode, bytes calldata executionData)`) and serves as the EIP-7702 delegation target for compromised accounts during recovery.

### Non-Custodial Architecture & EIP-7702 Execution Context

A central architectural guarantee of RescueKit is that **the smart contract never takes custody of user assets**:
- **`address(this)` IS the Compromised Wallet**: Under EIP-7702 ephemeral delegation, the bytecode of `SponsorableBatchExecutor` executes directly inside the account context of the compromised wallet.
- **Zero Intermediate Escrow**: There is **no escrow contract, protocol vault, intermediate pool, or deposit contract**. Assets are never held by or routed through an external protocol treasury contract before reaching safety.
- **Direct Atomic Transfers**: Rescued tokens, native currency, and NFTs are swept directly from the victim's wallet to `safeDestination` in the same atomic block.
- **Why Assets Cannot Be Stuck in the Contract**: Because the protocol takes zero custody, it is impossible for user funds to get "permanently stuck or locked in the contract". If a transfer to `safeDestination` reverts (e.g. if the user enters a misconfigured or unpayable contract address), the assets simply remain where they started: in `address(this)`—the victim's compromised wallet.
- **Sweeper Risk & The 91% Refund Safeguard**: Because funds that fail to leave `address(this)` remain in the compromised account, they are vulnerable to mempool sweeper bots. This is why:
  1. If a 6% referral payout fails and the subsequent redirect to the protocol treasury also fails, the contract automatically refunds the 6% cut into the user's sweep amount (`netSend += unpaidCut;`), returning **91.00%** net to `safeDestination` rather than leaving dust in the compromised address for attackers.
  2. Users must always ensure their `safeDestination` is a clean, receptive address (standard EOA or verified Safe multisig).

### Storage Layout & Protocol Constants
When a compromised account delegates to `SponsorableBatchExecutor` via EIP-7702, code execution runs in the context of the compromised wallet (`address(this)` is the compromised address). To access protocol-wide configuration without corrupting the victim's storage slots, immutable values and static logic contract references are utilized:

| Variable / Constant | Type | Value / Behavior | Description |
| :--- | :--- | :--- | :--- |
| `implementation` | `address` | Set at deployment | Address of the master canonical logic deployment. |
| `owner` | `address` | Admin multi-sig | Protocol governance address authorized to update parameters. |
| `feeRecipient` | `address` | Treasury address | Target address for protocol recovery fees. |
| `feeBps` | `uint256` | `1500` (15.00%) | Protocol recovery fee in basis points (max capped at 1,500 bps / 15%). |
| `affiliateCutBps` | `uint256` | `600` (6.00% gross) | Share of total rescued assets awarded to the affiliate (40% of the 15% fee). If both referrer transfer and treasury redirect fail, refunded to victim (yielding 91.00% net). |
| `referralPaid` | `mapping(address => bool)` | Persistent flag | Tracks whether a given compromised wallet has completed its initial referral payout. |
| `paused` | `bool` | `false` | Emergency circuit-breaker. When `true`, all execution reverts with `EnforcedPause()`. |

### Transient Storage Slots (EIP-1153)
To ensure optimal gas efficiency and rock-solid reentrancy protection during multi-step operations (such as flash loans and NFT mint callbacks), the contract utilizes ephemeral transient storage (`tstore` / `tload`):
- `_FLASH_ACTIVE_SLOT` (`0x466c6173682d...`): Set to `1` during active flash loan callback verification.
- `_FLASH_TARGET_SLOT` (`0x466c6173682d...`): Stores the authorized flash loan pool address to prevent spoofed callbacks.
- `_MINT_ACTIVE_SLOT` (`0x4d696e742d...`): Records expected NFT mint quantity for safe-transfer callback forwarding.
- `_MINT_COLLECTION_SLOT` (`0x4d696e742d...`): Stores authorized NFT collection address.
- `_MINT_DEST_SLOT` (`0x4d696e742d...`): Stores the target `safeDestination` for direct callback forwarding.
- `_MINT_FORWARDED_SLOT` (`0x4d696e742d...`): Tracks the counter of NFTs safely forwarded in real time.

---

## 2. Execution Modes (ERC-7821)

`SponsorableBatchExecutor` interprets the 32-byte `mode` argument in `execute(bytes32 mode, bytes calldata executionData)` according to the following specification:

| Mode ID | Name | Mode Constant (bytes32) | Calldata Encoding Structure | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **1** | `SINGLE_BATCH` | `0x0100000000000000000000000000000000000000000000000000000000000000` | `abi.encode(Call[] calls)` | Self-invocation only (`msg.sender == address(this)`). Executes calls without sponsor opData. |
| **2** | `SINGLE_BATCH_WITH_OPDATA` | `0x0100000000007821000100000000000000000000000000000000000000000000` | `abi.encode(Call[] calls, bytes opData)` | Sponsored token & NFT recovery. `opData` contains inner signature and optional referrer. |
| **4** | `FLASH_LOAN_BATCH` | `0x0100000000007821000400000000000000000000000000000000000000000000` | `abi.encode(Call[] calls, bytes opData)` | Lending collateral rescue. `calls[0]` initiates the flash loan; inner operations unwind positions. |
| **6** | `CLAIM_BATCH` | `0x0100000000007821000600000000000000000000000000000000000000000000` | `abi.encode(Call[] calls, bytes opData)` | Single airdrop/token claim. `calls[0]` executes claim; tail calls sweep tokens and residual gas. |
| **7** | `MINT_BATCH` | `0x0100000000007821000700000000000000000000000000000000000000000000` | `abi.encode(Call[] calls, bytes opData, address safeDestination, address nftCollection)` | ERC-721 mint + auto-sweep. Intercepts `onERC721Received` or predicts ID and executes `transferFrom`. |
| **8** | `MINT_BATCH_1155` | `0x0100000000007821000800000000000000000000000000000000000000000000` | `abi.encode(Call[] calls, bytes opData, address safeDestination, address nftCollection)` | ERC-1155 edition mint + sweep. Intercepts `onERC1155Received` or decodes calldata to sweep edition. |
| **9** | `MULTI_CLAIM_BATCH` | `0x0100000000007821000900000000000000000000000000000000000000000000` | `abi.encode(Call[] claims, Call[] sweeps, bytes opData)` | Lenient multi-claim. Emits `ClaimFailed` on reverting claims without aborting sweep operations. |

---

## 3. Cryptographic Digests & Replay Protection

To prevent unauthorized execution, front-running, or cross-chain replay attacks, each mode defines an exact cryptographic digest. The digest is signed by the compromised account using standard **EIP-191 Personal Sign** format (`MessageHashUtils.toEthSignedMessageHash(digest)`).

### Mode 2, 4, 6 Digest
Used for standard token sweeps, flash loans, and single claims:
```solidity
bytes32 digest = keccak256(abi.encode(
    mode,
    keccak256(abi.encode(calls)),
    referrer,
    block.chainid,
    address(this)
));
```

### Mode 7 & 8 (NFT Mint) Digest
Explicitly pins the safe destination and NFT collection contract:
```solidity
bytes32 digest = keccak256(abi.encode(
    mode,
    keccak256(abi.encode(calls)),
    safeDestination,
    nftCollection,
    referrer,
    block.chainid,
    address(this)
));
```

### Mode 9 (Multi-Claim) Digest
Binds both claim calls and sweep calls independently:
```solidity
bytes32 digest = keccak256(abi.encode(
    mode,
    keccak256(abi.encode(claimCalls)),
    keccak256(abi.encode(sweepCalls)),
    referrer,
    block.chainid,
    address(this)
));
```

---

## 4. Complete Error Directory & Resolutions

### On-Chain Contract Reverts

| Revert Error | Condition / Cause | Security Rationale | Resolution |
| :--- | :--- | :--- | :--- |
| `UnsupportedExecutionMode()` | The provided `mode` bytes32 does not match any recognized mode ID (1–9). | Protects against invalid calldata decoding. | Verify that the mode constant matches `ERC7821_MODES` in `@wallet-rescue/abi`. |
| `Unauthorized()` (`0x82b42900`) | The recovered signer from `ECDSA.recover(ethSignedDigest, signature)` does not equal `address(this)`. | Prevents unauthorized third parties from executing arbitrary calls on the victim's wallet. | 1) Ensure the private key belongs to the compromised address; 2) **Crucial Integration Detail**: The contract applies `MessageHashUtils.toEthSignedMessageHash(digest)`. In Viem/ethers, clients **must** use `account.signMessage({ message: { raw: digest } })` (EIP-191 personal sign). Using raw ECDSA `account.sign({ hash: digest })` double-hashes the digest and always reverts with `0x82b42900`. |
| `EnforcedPause()` | The contract owner has placed the contract in a paused state. | Emergency circuit breaker to halt protocol operations. | Check protocol status announcements; retry once maintenance concludes. |
| `"Flash loan request failed"` | The initial flash loan call (`calls[0]`) reverted on the pool. | Flash pool rejected loan request (e.g. pool out of liquidity). | Check pool liquidity and borrow limits for the target asset. |
| `"Flash loan settlement failed"` | Post-operation balance in the compromised account is less than `amount + premium`. | Pool would revert due to insufficient repayment. | Ensure sufficient collateral is liquidated/swapped to cover loan principal plus fee. |
| `"Token claim failed"` | Mode 6 claim interaction returned `false` or reverted. | Prevents silent failure where no tokens were claimed. | Verify claim eligibility and Merkle proof; switch to Mode 9 if submitting multiple claims. |
| `"NFT mint failed"` | NFT contract reverted during the mint execution call. | Halts execution if mint preconditions (allowlist, price, supply) are not met. | Confirm allowlist status, public sale timing, and ensure adequate native value is attached. |
| `FeeTooHigh()` | Attempted to set `feeBps` greater than `MAX_FEE_BPS` (`1500`). | Protocol governance safeguard preventing excessive fees. | Ensure governance proposal stays within maximum 15% boundary. |
| `ZeroFeeRecipient()` | Attempted to set `feeRecipient` to `address(0)`. | Prevents burning protocol fees into the null address. | Supply a valid treasury address. |

### Application & API Validation Errors

| Error String | Cause | Resolution |
| :--- | :--- | :--- |
| `"Destination address cannot be the same as the compromised address."` | `safeDestination.toLowerCase() === compromisedAddress.toLowerCase()`. | Provide an independent, clean recovery address. |
| `"Destination is a token contract. Trapped funds will be lost forever."` | On-chain bytecode inspection detected an ERC-20 or ERC-721 interface on the destination. | Provide a standard EOA wallet or Safe multisig address. |
| `"Insufficient sponsor gas. Please deposit at least X ETH..."` | Sponsor wallet balance is less than estimated gas execution cost. | Deposit native gas into the sponsor burner address shown in the sidebar. |
| `"Missing required execute fields: compromisedNonce, sponsorNonce, maxFeePerGas, maxPriorityFeePerGas, tokens"` | Required nonces, gas limits, or token list were omitted from `POST /api/execute`. | Include nonces, fee strings (in wei), and tokens array in the request body. |
| `"Invalid tokens list: tokens array is required and must not be empty"` | Request body provided an empty `tokens` array. | Include at least one ERC-20 token contract address or `"native"` in the `tokens` array. |
| `"Invalid private key: must be 32 hex bytes."` | Supplied private key is not 64 hexadecimal characters. | Double-check private key string; remove any extra whitespace or prefixes. |
| `"Too many requests. Please try again in X seconds."` | IP exceeded sliding window rate limit on `/build`, `/broadcast`, or `/execute`. | Wait for the `Retry-After` duration before making subsequent requests. |
| `"Missing required 'claims' array."` | Payload for claim batch build was missing claim definitions. | Provide valid contract address and calldata for the claim. |
| `"needsSwap.swapFrom and swapTo cannot be identical"` | Swap source and target addresses are the same token. | Set `swapFrom` to the collateral asset and `swapTo` to the borrowed debt asset. |

### RPC & Network Failures

| Issue | Cause | Automated Handling | Resolution |
| :--- | :--- | :--- | :--- |
| `Transaction confirmation timed out after 60000ms.` | Network congestion delayed block inclusion. | Client falls back to secondary RPC nodes to poll for receipt. | Check the transaction hash on the block explorer; tx may confirm shortly. |
| `Nonce too low` | Sponsor wallet submitted multiple transactions in rapid succession. | Local nonce tracker updates on next balance refresh. | Refresh sponsor wallet balance or wait 10 seconds for pending tx to confirm. |
| `Replacement transaction underpriced` | A replacement transaction did not increase gas fees by at least 10%. | Gas builder applies dynamic priority multipliers. | Select the **Rapid** gas priority preset before resubmitting. |
