# RescueKit — Lending Architecture & Integration Guide

This document is the authoritative architectural specification and developer guide for the **DeFi Lending Rescue System** in RescueKit.

It covers:
1. **The Separation of Concerns**: Lending Markets (where debt/collateral lives) vs. Flash Loan Providers (where capital is borrowed).
2. **The End-to-End Rescue Lifecycle**: Scan → Preview → Live Debt Multicall3 → Flash Execution.
3. **Dynamic Interest & Debt Resolution**: Why `repay(type(uint256).max)` + generic in-batch live re-fetch eliminates stale scan failures without guessing buffers.
4. **Smart Flash Loan Routing**: 0% fee prioritization (Morpho Blue, Balancer v2/v3) with Aave v3 fallback.
5. **Step-by-Step Guide: How to Add a New Lending Protocol**: Scanner + Batch Executor.
6. **Step-by-Step Guide: How to Add a New Flash Loan Provider**.

---

## 1. The Core Mental Model: Two Independent Layers

A common point of confusion is mixing up **where the user borrowed money** with **who provides the flash loan**. In RescueKit, these are completely decoupled:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      WHO PROVIDES THE FLASH CAPITAL?                        │
│                                                                             │
│   [Morpho Blue (0%)]       [Balancer v2/v3 (0%)]       [Aave v3 (0.05%)]    │
│   Single-token priority    Multi-token priority        Fallback pool        │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │ borrows capital
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       WHERE DOES THE MONEY GO?                              │
│                                                                             │
│   1. approve(lendingMarket, max)                                            │
│   2. lendingMarket.repay(debt)          <-- burns debt to zero              │
│   3. lendingMarket.withdraw(collateral) <-- releases 100% collateral        │
│   4. swapRouter.exactInputSingle(...)   <-- sells collateral to buy debt    │
│   5. repay flash loan provider                                              │
│   6. sweep remaining collateral + debt to safe address                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

- **Lending Protocols** (`packages/batch/src/lending/protocols/` and `packages/scanner/src/protocols/`):
  Where the compromised wallet actually holds positions (e.g., Aave v3, Compound v3 / Comet, Curvance, Morpho Blue Vaults).
- **Flash Loan Providers** (`packages/batch/src/lending/flashloans/`):
  External liquidity sources that lend capital for 1 block so the wallet can clear its debt before unlocking collateral.

---

## 2. Dynamic Interest & Live Debt Resolution

### The Stale Scan Problem
When a user opens the Lending page, the scanner queries their debt at block $N$. If the user spends 15 minutes reviewing quotes, or if interest accrues across multiple blocks:
- The on-chain debt grows by interest.
- If the batch only borrows the stale scan amount, Aave's `repay` call either:
  1. Repays only the stale amount, leaving debt dust behind $\rightarrow$ health factor stays $< \infty \rightarrow$ full collateral withdrawal fails.
  2. Or if `repay` attempts to clear 100% debt (`type(uint256).max`), the wallet balance is short by a few wei $\rightarrow$ `ERC20: transfer amount exceeds balance` revert.

### The Solution: 2-Stage Dynamic Resolution

```
User Scans (T = 0)
    │  Scanner records positions with preliminary debtRaw.
    ▼
User Clicks "Confirm Rescue" (T = 15m)
    │
    ├─► Step 1: In-Batch Live Multicall3 Re-Fetch (buildLendingBatch)
    │          Queries each protocol executor's `buildLiveDebtCall()`.
    │          Batch receives exact live debt at the current block.
    │
    ├─► Step 2: Flash Loan Borrows Live Amount (+ protocol buffer/fee)
    │          Morpho/Balancer: exact debt + 10 bps cushion (0% fee).
    │          Aave: exact debt + 5 bps protocol fee.
    │
    └─► Step 3: In-Tx Full Clearance
               Aave Pool runs `repay(asset, type(uint256).max, ...)`.
               Aave reads the live on-chain state directly inside the transaction,
               burning 100% of the debt.
               Any unused pennies from the cushion are swept directly to the safe
               destination at the end of the transaction. Zero fund loss.
```

---

## 3. Protocol Architecture & Interfaces

### 3.1 Scanner Interface (`packages/scanner/src/protocols/types.ts`)

Every lending protocol scanner implements `LendingProtocolScanner`:

```typescript
export interface ProtocolScanCall {
  address: Address;
  abi: readonly any[];
  functionName: string;
  args?: readonly any[];
}

export interface LendingProtocolScanner {
  id: string;
  name: string;
  /** Emits Multicall3 call definitions for this protocol on the target chain */
  buildCalls(user: Address, chain: RescueChain): ProtocolScanCall[];
  /** Parses Multicall3 results into normalized LendingPosition objects */
  parsePositions(
    results: readonly any[],
    startIndex: number,
    user: Address,
    chain: RescueChain
  ): LendingPosition[];
}
```

### 3.2 Batch Executor Interface (`packages/batch/src/lending/types.ts`)

Every lending protocol executor implements `LendingProtocolExecutor`:

```typescript
export interface MulticallContract {
  address: Address;
  abi: readonly any[];
  functionName: string;
  args?: readonly any[];
}

export interface LendingProtocolExecutor {
  protocolId: string;
  /** Encodes the repay call for the inner flash loan batch */
  encodeRepay(position: LendingPosition, compromisedAddress: Address): BatchCall;
  /** Encodes the collateral withdrawal call */
  encodeWithdraw(position: LendingPosition, compromisedAddress: Address): BatchCall;
  /**
   * Optional: Returns a Multicall3 call definition to fetch live debt on-chain
   * right before building the batch. Receives the full RescueChain config.
   */
  buildLiveDebtCall?(position: LendingPosition, user: Address, chain: RescueChain): MulticallContract;
  /**
   * Parses the raw result from buildLiveDebtCall into live debt wei.
   */
  parseLiveDebt?(result: any): bigint;
}
```

---

## 4. Smart Flash Loan Router (`packages/batch/src/lending/flashloans/router.ts`)

RescueKit evaluates flash loan providers dynamically based on **fees** and **live on-chain liquidity**:

### Priority Hierarchy
1. **Single Debt Token**:
   - **1st Choice**: **Morpho Blue** (0% Fee, Base / Mainnet)
   - **2nd Choice**: **Balancer v2** (0% Fee, Base / Mainnet / Polygon / Optimism / Arbitrum)
   - **3rd Choice**: **Balancer v3** (0% Fee)
   - **Fallback**: **Aave v3** (0.05% Fee / 5 bps)

2. **Multi Debt Tokens** (cross-collateral with multiple borrowings):
   - **1st Choice**: **Balancer v2 Multi** (0% Fee)
   - **2nd Choice**: **Balancer v3 Multi** (0% Fee)
   - **Fallback**: **Aave v3 Multi** (0.05% Fee / 5 bps)

### Live Liquidity Verification
Before selecting a 0% fee provider, the router calls `balanceOf(providerVault)` for each debt token. If the vault does not hold enough liquidity to fund the flash loan, it smoothly falls back to the next provider in the chain.

---

## 5. Adding Tokens, Markets, and Protocols

There is a fundamental difference between **adding a token or market to an existing protocol** versus **integrating a brand new protocol from scratch**:

| Task | What Needs to be Done | Files Touched | Code Changes? |
|---|---|---|---|
| **Add a Token / Market to an Existing Protocol** (e.g. Moonwell, Aave) | Add token or market address to the chain configuration | Only `packages/chains/src/<chain>/config.ts` | **No** (pure JSON/TS config) |
| **Add a Brand New Protocol** (e.g. Morpho Blue, Spark, Compound v2) | Write a scanner adapter + batch executor adapter | `packages/scanner/src/protocols/<name>.ts`<br>`packages/batch/src/lending/protocols/<name>.ts` | **Yes** (2 adapter files) |

---

### 5.1 Adding a Token or Market to an Existing Protocol (Pure Config)

`packages/chains` is the **single source of truth** for all chains, tokens, and market addresses.  
You **never** need to touch the scanner, batch builder, or UI to add tokens or markets.

#### A. Adding a New Collateral Token to Moonwell / Compound v3
To support a new collateral asset in an existing Moonwell market on Base (e.g., adding a new token to the USDC market):
1. Open `packages/chains/src/base/config.ts`.
2. Locate `cometMarkets` and find the target market (e.g. `USDC Lending Market`).
3. Add the token under `collateralAssets`:
```typescript
{
  comet: '0xb125E6687d4313864e53df431d5425969c15Eb2F',
  symbol: 'USDC',
  baseToken: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',
  baseDecimals: 6,
  name: 'USDC Lending Market',
  collateralAssets: [
    { address: '0x4200000000000000000000000000000000000006', symbol: 'WETH', decimals: 18 },
    // Add your new collateral token here:
    { address: '0xNEW_TOKEN_ADDRESS', symbol: 'NEW_SYMBOL', decimals: 18 },
  ],
}
```

#### B. Adding a New Moonwell Market
If Moonwell deploys a brand new Comet market (e.g., cbBTC lending market):
1. Add an entry to `cometMarkets` in `packages/chains/src/<chain>/config.ts`:
```typescript
{
  comet: '0xCOMET_PROXY_ADDRESS',
  symbol: 'cbBTC',
  baseToken: '0xTOKEN_ADDRESS',
  baseDecimals: 8,
  name: 'cbBTC Lending Market',
  collateralAssets: [
    { address: '0xCOLLATERAL_TOKEN_1', symbol: 'WETH', decimals: 18 },
    { address: '0xCOLLATERAL_TOKEN_2', symbol: 'USDC', decimals: 6 },
  ],
}
```

#### C. Adding a New Asset to Aave v3
To support scanning and rescuing a new asset in Aave v3:
1. Open `packages/chains/src/<chain>/config.ts`.
2. Add the token to `aave.assets`:
```typescript
aave: {
  pool: '0x...',
  dataProvider: '0x...',
  assets: [
    { address: '0xNEW_TOKEN', symbol: 'NEW', decimals: 18 },
  ],
}
```

---

### 5.2 Step-by-Step: Adding a Brand New Lending Protocol

Follow these 4 steps when adding an entirely new protocol that has distinct smart contract interfaces (e.g. Morpho Blue, Spark, Compound v2).

> [!IMPORTANT]
> **Modular Decoupling Rule for LendingPosition**:
> The scanner should always emit **separate positions** for collateral and debt:
> - Collateral position: `collateralRaw > 0n`, `debtRaw = 0n`, `decimals = collateralDecimals`, `symbol = collateralSymbol`.
> - Debt position: `debtRaw > 0n`, `collateralRaw = 0n`, `decimals = debtDecimals`, `symbol = debtSymbol`.
> Both positions share the same `marketAddress`.
> The UI's `getLoanGroups()` automatically pairs them by `marketAddress`, accurately preserves both tokens' independent decimals/symbols, and formats titles like `WETH / USDC (Protocol)`.

#### Step 1: Create the Scanner (`packages/scanner/src/protocols/<name>.ts`)

Implement `LendingProtocolScanner`:
```typescript
import { type Address } from "viem";
import type { RescueChain } from "@wallet-rescue/chains";
import type { LendingPosition } from "@wallet-rescue/types";
import { type LendingProtocolScanner, type ProtocolScanCall, isDust } from "./types.js";

export const myProtocolScanner: LendingProtocolScanner = {
  id: "myprotocol",
  name: "My Protocol",
  buildCalls(user: Address, chain: RescueChain): ProtocolScanCall[] {
    // Read markets from chain configuration (chain.myProtocol?.markets ?? [])
    return [
      {
        address: MY_PROTOCOL_ADDRESS,
        abi: myProtocolAbi,
        functionName: "getUserPosition",
        args: [user],
      },
    ];
  },
  parsePositions(results, startIndex, user, chain): LendingPosition[] {
    const res = results[startIndex]?.result;
    if (!res) return [];

    const positions: LendingPosition[] = [];
    // Emit decoupled collateral and debt LendingPosition objects:
    // - marketAddress: Identifies the pool/market (used by getLoanGroups to pair them)
    // - decimals: Use the exact decimals of THAT specific token (collateral vs debt)
    return positions;
  },
};
```

Register it in `packages/scanner/src/protocols/index.ts`:
```typescript
export const LENDING_PROTOCOL_SCANNERS: LendingProtocolScanner[] = [
  aaveScanner,
  cometScanner,
  curvanceScanner,
  myProtocolScanner, // <-- Add here
];
```

#### Step 2: Create the Batch Executor (`packages/batch/src/lending/protocols/<name>.ts`)

Implement `LendingProtocolExecutor` with `encodeRepay`, `encodeWithdraw`, and live debt resolution:

```typescript
import { encodeFunctionData, type Address } from "viem";
import type { RescueChain } from "@wallet-rescue/chains";
import type { BatchCall, LendingPosition } from "@wallet-rescue/types";
import type { LendingProtocolExecutor, MulticallContract } from "../types.js";

export const myProtocolExecutor: LendingProtocolExecutor = {
  protocolId: "myprotocol",

  encodeRepay(position: LendingPosition, compromisedAddress: Address): BatchCall {
    return {
      to: position.marketAddress,
      value: 0n,
      data: encodeFunctionData({
        abi: myProtocolAbi,
        functionName: "repay",
        // Pass type(uint256).max if protocol supports it, else position.debtRaw
        args: [position.debtAddress!, (1n << 256n) - 1n, compromisedAddress],
      }),
    };
  },

  encodeWithdraw(position: LendingPosition, compromisedAddress: Address): BatchCall {
    return {
      to: position.marketAddress,
      value: 0n,
      data: encodeFunctionData({
        abi: myProtocolAbi,
        functionName: "withdraw",
        args: [position.collateralAddress, (1n << 256n) - 1n, compromisedAddress],
      }),
    };
  },

  buildLiveDebtCall(position: LendingPosition, user: Address, chain: RescueChain): MulticallContract {
    return {
      address: position.marketAddress,
      abi: myProtocolAbi,
      functionName: "getDebt",
      args: [position.debtAddress!, user],
    };
  },

  parseLiveDebt(result: any): bigint {
    return BigInt(result);
  },
};
```

Register it in `packages/batch/src/lending/protocols/index.ts`:
```typescript
export const LENDING_PROTOCOL_EXECUTORS: Record<string, LendingProtocolExecutor> = {
  aave: aaveExecutor,
  comet: cometExecutor,
  curvance: curvanceExecutor,
  myprotocol: myProtocolExecutor, // <-- Add here
};
```

#### Step 3: Register Chain Configuration (if needed)

If the protocol uses specific registry or oracle addresses per chain, add them to `RescueChain` in `packages/chains/src/<chain>/config.ts`.

#### Step 4: Write Fork Tests

Add an Anvil fork test in `packages/batch/src/lending.fork.test.ts` verifying:
1. Collateral is deposited and debt is borrowed on the real protocol contract.
2. `buildLendingBatch` creates the rescue batch.
3. Batch executes cleanly through `SponsorableBatchExecutor`.
4. Debt is zeroed and collateral is safely transferred to `safeDestination`.

---

## 6. Step-by-Step: Adding a New Flash Loan Provider

If a new 0% fee or low-fee flash loan provider becomes available:

1. **Create Provider File** (`packages/batch/src/lending/flashloans/<provider>.ts`):
   - Implement `FlashLoanProvider` (`id`, `feeBps`, `isMultiToken`, `getPoolAddress`, `encodeFlashLoan`).
   - Pack the callback parameters with `encodedInnerParams` according to how the deployed `SponsorableBatchExecutor.sol` receives callbacks.
2. **Export in Index** (`packages/batch/src/lending/flashloans/index.ts`).
3. **Register in Router** (`packages/batch/src/lending/flashloans/router.ts`):
   - Add pool lookup and liquidity check inside `selectBestFlashLoanProvider`.
   - Place it in the appropriate priority rank according to fee and token flexibility.

---

## 7. Critical Safety Checklist for Lending Code

- [ ] **Infinite Repay & Withdraw**: Always use `(1n << 256n) - 1n` (`type(uint256).max`) when calling Aave or protocols supporting full balance clearance to eliminate residual dust.
- [ ] **Deduplicate Collateral Withdrawals**: The builder must ensure a position whose collateral is being swapped is not withdrawn twice (handled by checking `collatPos !== pos && pos.collateralRaw === 0n`).
- [ ] **Keep Contract Frozen**: Never modify `SponsorableBatchExecutor.sol`. All changes are made in SDK builders and encoders.
- [ ] **Verify Gas Limits**: Lending rescues execute heavy inner loops (repay + withdraw + Uniswap swap + flash payback + sweep). Ensure the strategy's `rescueGasFallback` allocates sufficient gas (typically 1.2M - 1.5M units on Base/Monad/Polygon).

---

## 8. ⚠️ Critical Invariant Warning: Live Production Battle-Tested Verification

> [!CAUTION]
> **DO NOT MODIFY CORE LENDING BUILDER, PROTOCOL EXECUTORS, OR FLASH LOAN LOGIC WITHOUT EXTREME CARE.**
> Every component in the DeFi lending rescue subsystem has been battle-tested and proven live on mainnets (Base and Polygon) with real funds under production conditions. Any modification to swap encoding, token ordering, flash loan callback unpacking, or dynamic sweeping can cause reverts (`BAL#102`, slippage reverts, or stuck dust).

### Complete Mainnet Battle-Tested Audit Trail

1. **Single-Debt Aggregation (Morpho Blue at 0% Fee)**
   - **Network**: Base Mainnet
   - **Tx Hash**: [`0x1ff5de92c8a992c59c716ff34d819d0cbfb61d49dd686d7fef91ce633006bcbc`](https://basescan.org/tx/0x1ff5de92c8a992c59c716ff34d819d0cbfb61d49dd686d7fef91ce633006bcbc)
   - **Verification**: Atomic aggregation of Aave v3 + Moonwell loans into 1 single 0% flash loan via Morpho Blue (`0.40 USDC`). Zeroed both debts, withdrew collaterals, swapped, swept to safe. Gas: 760k.

2. **Pure Idle Collateral Rescue (Mode 2 Direct Withdrawal)**
   - **Network**: Base Mainnet
   - **Tx Hash**: [`0x277b31e530077c3da0c03c2ca46df8bf5607f62d2a3f7246da484e2cc1e1a085`](https://basescan.org/tx/0x277b31e530077c3da0c03c2ca46df8bf5607f62d2a3f7246da484e2cc1e1a085)
   - **Verification**: Direct withdrawal and sweep of pure idle supplied collateral without flash loans or swap overhead. Gas: 132k.

3. **Mixed Batch Rescue (Active Loans + Pure Idle Collateral Atomic in 1 Tx)**
   - **Network**: Base Mainnet
   - **Tx Hash**: [`0xd013acb248fd473c49b16e65279c2a6b2821840fd96979eee21a53c64dd4424e`](https://basescan.org/tx/0xd013acb248fd473c49b16e65279c2a6b2821840fd96979eee21a53c64dd4424e)
   - **Verification**: Morpho 0% flash loan (`0.35 USDC`) repaid Moonwell & Aave debt, while simultaneously executing pure idle collateral withdrawal (`0.00020 WETH` from Moonwell WETH market inside `flashInner`), swapped, swept `1.395 USDC` + `0.000187 WETH` to safe. Gas: 783k.

4. **Multi-Token Debt Batch (Balancer v2 Multi at 0% Fee)**
   - **Network**: Base Mainnet
   - **Tx Hash**: [`0x6f55fac1bc0e8623a1f21520d729a027d73270728febe3f5bd48ff0d1ae9c3ad`](https://basescan.org/tx/0x6f55fac1bc0e8623a1f21520d729a027d73270728febe3f5bd48ff0d1ae9c3ad)
   - **Verification**: Balancer v2 Vault lent `0.00005 WETH` + `0.15 USDC` at 0% fee (enforced numerical ascending token sorting to satisfy `BAL#102`), repaid Moonwell & Aave, swapped collaterals, repaid Balancer, swept `0.825 USDC` + `0.000303 WETH` to safe. Gas: 808k.

5. **Multi-Token Debt Batch (Balancer v3 Multi at 0% Fee + Affiliate Referral)**
   - **Network**: Base Mainnet
   - **Tx Hash**: [`0x3df823590321a3502d3a3836b4ed96a1e0ea865c6e6dcd6b19f1bd97528895c6`](https://basescan.org/tx/0x3df823590321a3502d3a3836b4ed96a1e0ea865c6e6dcd6b19f1bd97528895c6)
   - **Verification**: Balancer v3 Vault (`0xbA13...`) lent `0.00005 WETH` + `0.15 USDC` at 0% fee via `onBalancerV3Unlock` callback, repaid loans, swapped, repaid Balancer v3 at 0% fee, awarded valid first-time affiliate commission to referrer, swept `0.340 USDC` + `0.000363 WETH` to safe. Gas: 894k.

6. **Multi-Token Debt Fallback (Aave v3 Multi + One-Time Referral Lock)**
   - **Network**: Base Mainnet
   - **Tx Hash**: [`0x0cdbdf6abf029b6ed1a88f900c24a185201a760e80eb40007f5cd62c82c4d8e7`](https://basescan.org/tx/0x0cdbdf6abf029b6ed1a88f900c24a185201a760e80eb40007f5cd62c82c4d8e7)
   - **Verification**: Aave v3 pool lent `0.100051 USDC` + `0.000030015 WETH` at 0.05% premium, repaid Moonwell & Aave, swapped, repaid Aave with fee, verified one-time referral protection ($0.00 duplicate referral paid), swept `0.396 USDC` + `0.000266 WETH` to safe. Gas: 846k.

7. **Simultaneous Multichain Lending Rescue (Base + Polygon in 1 Parallel Batch)**
   - **Base Tx**: [`0x2586e64fbb7a2aba91d4c95f7fe74d1486b40d6cd48edc9d15ff1e0635a582e1`](https://basescan.org/tx/0x2586e64fbb7a2aba91d4c95f7fe74d1486b40d6cd48edc9d15ff1e0635a582e1) — Moonwell loan rescued via Morpho Blue 0% flash loan (`0.10 USDC`), swapped WETH to USDC, swept `0.6555 USDC` + `0.0000625 WETH` to safe. Gas: 430k.
   - **Polygon Tx**: [`0xb53bf8900cbf3daa7b0470ca33a14b263401ed6374a3254b16c830707c8d5f02`](https://polygonscan.com/tx/0xb53bf8900cbf3daa7b0470ca33a14b263401ed6374a3254b16c830707c8d5f02) — Polygon Aave v3 loan rescued: Smart router detected Morpho is not on Polygon and seamlessly routed to Balancer v2 Vault at 0% fee borrowing `0.050001 USDC`, repaid Polygon Aave v3 debt, swapped 2 WPOL to USDC on Uniswap v3, paid first-time affiliate commission on Polygon to referrer `0x640E...` (`0.01136 USDC` + WPOL dust carved out of protocol fee), swept `0.1609 USDC` to safe. Gas: 1,013k.
   - **Verification**: Executed simultaneously across 2 independent blockchains in 1 user submission; proved cross-chain first-time referral reward mechanics.

