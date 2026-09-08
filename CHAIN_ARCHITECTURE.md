# RescueKit — Chain Architecture & Integration Guide

This document covers everything about how chains are configured, how gas/fees are derived per chain, how the review modal and execution differ, and the exact, clean procedure for adding a new chain.

> **Read this before touching chains.** It is the single source of truth for the per-chain config system that was centralized in this refactor.

---

## 1. The mental model: one chain = one folder

Every chain lives in `packages/chains/src/<slug>/` and is self-contained. The rest of the app (web pages, review modals, execution, watcher) reads everything from the `CHAINS` registry, so **you never edit web code to add a chain.**

A chain folder has three files:

| File | Purpose |
|---|---|
| `config.ts` | The `RescueChain` object — every per-chain fact (slug, RPC, executor, feeRecipient, feeBps, Aave/swap config, reserve, etc.) |
| `rescue.ts` | The strategy — per-chain gas/fee behavior. Usually just `extends EthereumLikeRescueStrategy` with a fee config. |
| `tokens.ts` | Curated token list (symbol + address + decimals). |

### The two central abstractions

**`RescueChain`** (in `types.ts`) — pure data. It is the registry entry. A chain is fully described by this object. Key fields:

- `slug`, `displayName`, `viem` (the viem chain, incl. `rpcUrls` from `publicRpc`)
- `publicRpc` — public HTTP RPC URL. **Required, no default.**
- `alchemyNetwork` — used by the watcher/server for Alchemy RPC.
- `erc7821Address` — the executor contract address.
- `feeRecipient`, `feeBps` — protocol fee info (from the deployed executor).
- `nativeReserveBalance` — minimum native balance to leave (Monad 10.5 MON).
- `hasPublicMempool`, `headSubscription`, `logsSubscription` — watcher/detection config.
- `aave?` (pool, dataProvider, assets, cometMarkets), `swapRouter?`, `uniswap?` (quoter, factory), `aaveOracle?` — lending/quote config.

**`ChainRescueStrategy`** (in `types.ts`) — behavior. Implemented by each chain's `rescue.ts`. Methods:

- `rpcUrl(transport)` — for server/watcher contexts.
- `needsPrivateRelay()` / `privateRelayUrl()` — the autonomous watcher fires via relay if true.
- `adjustGasEstimate(baseEstimate)` — buffers a **live** gas estimate (execution only).
- `adjustFeeData(estimatedFees, selection)` — multiplies live gwei into `(maxFeePerGas, maxPriorityFeePerGas)`.
- `confirmationTimeout()` — receipt wait ms.
- `rescueGasFallback(input)` — the dynamic/worst-case gas-units formula (review modal + fallback when `estimateGas` fails).

---

## 2. The two gas quantities (this is the crux — read twice)

Every transaction needs **two independent numbers**, and they come from **two different places**:

| Quantity | What it is | Where it comes from | RPC? |
|---|---|---|---|
| **Gas units** | A count, e.g. `350000` | `rescueGasFallback(...)` — pure math | **No** |
| **Gwei price** | Per-unit price, e.g. `0.115 gwei` | `adjustFeeData(...)` on **live RPC gwei** | **Yes** |

They are multiplied at broadcast/sign time: `cost = gas_units × gwei_price`.

### Why this is important
- `rescueGasFallback` **does not need gwei** — it is only the count. It always works.
- `adjustFeeData` **needs the live gwei** from the RPC (`estimateFeesPerGas()` + `getGasPrice()`). If that RPC fails, **you cannot sign a tx** because a tx always needs `maxFeePerGas`.
- So "the math formula saves it if gwei fails" is **wrong** — the formula only supplies gas units. The price is irreplaceable-by-math.

### Fee floors are 0 (by design)
`minTipWei` and `minBaseFeeWei` are all `0n`. This means the fee is **purely `live_gwei × multiplier`**. There is no hardcoded floor, no explorer-maintained value, no overpaying a calm moment. The multiplier alone keeps the tx competitive *relative to the current network*.

**Safety guard:** `adjustFeeData` throws if the RPC returns no live fee at all (`!gasPrice && !maxFeePerGas && !maxPriorityFeePerGas`). This prevents a `0 × multiplier = 0 gwei` broadcast. The review modal surfaces the error; execution aborts before broadcast.

---

## 3. Per-chain fee config (current live values)

`EthereumLikeRescueStrategy` takes an `EthereumLikeConfig`. Here are the current per-chain values (all confirmed in `rescue.ts`):

| Chain | `baseFeeMult` | `tipMult` | `tipFloor` | `baseFloor` | `relay` | `timeout` | gas buffer |
|---|---|---|---|---|---|---|---|
| Base | 3 | 1 | 0 | 0 | false | 60s | 1.2x |
| Monad | 3 | 5 | 0 | 0 | false | 30s | 1.0x (as-is) |
| Optimism | 3 | 2 | 0 | 0 | false | 60s | 1.2x |
| BSC | 3 | 5 | 0 | 0 | **true** | 60s | 1.2x |
| Polygon | 3 | 2 | 0 | 0 | **true** | 90s | 1.2x |

### Why each chain differs (the "different flow" rationale)
- **Base** — sequencer L2, near-zero fees. Low tip (1x), 3x base buffer for spikes.
- **Monad** — no private relay possible, so it must **win the block ordering** via tip. 5x tip bid ("gas war"), gas estimate used as-is (bills on gasLimit, not gasUsed), 100 gwei base market.
- **Optimism** — **sequencer** chain (single sequencer, no public mempool race). Modest 2x tip, 3x base buffer. Cheap.
- **BSC** — legacy gasPrice market (no EIP-1559 base fee), public mempool. 5x tip (negligible cost) to guarantee inclusion, 3x base buffer on the derived base.
- **Polygon** — expensive (real ~253 gwei base), public mempool. 2x tip (moderate — 5x would be costly), 3x base buffer to survive spikes.

### The relay split
- **Manual rescue** → always broadcast via **public RPC** (the web `executionClients` uses `publicRpcUrl`, never Alchemy).
- **Autonomous watcher** → uses relay when `needsPrivateRelay === true` (BSC, Polygon). The same strategy tells the watcher this via `privateRelayUrl()`.

---

## 4. Gas-units fallback formula (`rescueGasFallback`)

This is the worst-case gas limit used in two places:
1. **Review modal** — always uses this (it never runs a live `estimateGas`).
2. **Execution fallback** — used when the live `estimateGas` throws.

### Default (`EthereumLikeRescueStrategy`) — used by Base, Optimism, BSC
```
transfer: max(100_000n + 65_000n × calls, 350_000n)
mint:     max(300_000n + 65_000n × calls, 450_000n)
claim:    max(250_000n + 65_000n × calls, 400_000n)
lending:  max(150_000n + 450_000n × debt + 120_000n × idle, 500_000n)
```

### Polygon override (heavier — bigger executor gas on Polygon)
```
transfer: max(120_000n + 80_000n × calls, 400_000n)
mint:     max(340_000n + 80_000n × calls, 500_000n)
claim:    max(280_000n + 80_000n × calls, 450_000n)
lending:  max(200_000n + 500_000n × debt + 150_000n × idle, 700_000n)
```

### Monad override
```
lending:  if debt > 0 → 1_500_000n (flat; Monad parallel execution is heavy)
          else max(150_000n + 120_000n × idle, 500_000n)
transfer/mint/claim: same as default
```

---

## 5. How the review modal computes cost (exact)

`ProgressPanel.tsx` (transfer/mint), `ClaimsPage` (claim), `LendingPage` (lending) all follow the same pattern. There is **no `estimateGas`** in the review modal — it deliberately uses the formula (per the doc's finding that public RPCs return unreliable `estimateGas` for 7702-delegated accounts).

```
gasForCheck = strategy.rescueGasFallback({ kind, callCount, debtCount, idleCount })   // gas UNITS (math)
feeData     = estimateFeesPerGas()      // live gwei (RPC, no catch — throws on failure)
gasPrice    = getGasPrice()             // live gwei (RPC, no catch)
adjusted    = strategy.adjustFeeData({ maxFeePerGas, maxPriorityFeePerGas, gasPrice })

baseFee = adjusted.maxFeePerGas > adjusted.maxPriorityFeePerGas
            ? adjusted.maxFeePerGas - adjusted.maxPriorityFeePerGas
            : adjusted.maxFeePerGas / 2
tip     = adjusted.maxPriorityFeePerGas

cost = gasForCheck × (baseFee + tip)     // units × price
ok   = sponsorBalance >= cost
```

This is **worst-case**: it budgets at `maxFeePerGas` (the cap) × worst-case gas units. So it's conservative — if the sponsor has this much, the tx will not run out of gas.

**On RPC failure:** the review modal's fee fetch has **no `.catch()`** now — it throws, is caught by the surrounding `try/catch`, and surfaces an error. **No fake/hardcoded gwei is substituted.**

---

## 6. How execution computes gas + fees (exact)

`executeRescue` (and the claim/mint/lending equivalents) follow this order:

1. **Validate executor** — `chain.erc7821Address` must exist.
2. **Build batch** — `buildRescueBatch(...)`.
3. **Parallel pre-fetch** (`Promise.all`, no catch on fees):
   ```
   compromisedNonce = getTransactionCount
   bytecode         = getBytecode
   feeData          = estimateFeesPerGas()   // live gwei, NO catch → abort if fails
   gasPrice         = getGasPrice()          // live gwei, NO catch → abort if fails
   sponsorNonce     = getTransactionCount(pending)
   ```
   **If the fee RPC fails → the whole `Promise.all` rejects → execution aborts before broadcast.**
4. **Adjust fees** — `adjustFeeData(...)` multiplies live gwei. Throws if no live fee (the guard).
5. **Set fallback gas** — `gasLimit = rescueGasFallback(...)` (math).
6. **Build auth** — `authList` unless already delegated.
7. **Live gas estimate** (`estimateGas` with `authorizationList`) — wrapped in its own `try/catch`:
   ```
   try {
     gasLimit = adjustGasEstimate(await estimateGas({ ... authorizationList }))
   } catch {
     // keep the rescueGasFallback value from step 5
   }
   ```
   **If `estimateGas` fails (the unreliable 7702 case) → keeps the formula gas → broadcasts anyway.** This is the doc's finding: public RPCs mis-estimate 7702-delegated txs, so the formula is the safety net **for gas units**.
8. **Sign + broadcast** via public RPC (or relay for watcher).

### Failure matrix
| What fails | Result |
|---|---|
| **Gwei fetch** (step 3) | `Promise.all` rejects → **abort before broadcast**. `rescueGasFallback` never runs. |
| **Gas `estimateGas`** (step 7, the 7702 case) | Empty catch → **falls back to formula**, tx proceeds. |
| **`rescueGasFallback` itself** | Never fails — it is pure math. |

So the dynamic formula is the safety net for **gas units** when `estimateGas` (7702) fails. It is **not** a safety net for **gwei** (price) — if gwei fails, you can't sign, so you abort. That is correct.

---

## 7. How to add a new chain (clean procedure)

1. **Create the folder** `packages/chains/src/<slug>/` with:
   - `config.ts` — build the `RescueChain` object. **Must** include: `slug`, `displayName`, `publicRpc`, `viem` (via `ethereumLike(..., { publicRpc })`), `erc7821Address`, `feeRecipient`, `feeBps`, `alchemyNetwork`, `nativeGasToken`, `rescuePath`, `readiness7702`, `note`, `commonTokens`, plus lending config (`aave`, `swapRouter`, `uniswap`, `aaveOracle`) if the chain supports lending.
   - `rescue.ts` — the strategy. Typically:
     ```ts
     export class MyChainRescueStrategy extends EthereumLikeRescueStrategy {
       readonly slug = 'myChain';
       getChain() { return myChain; }
       constructor() {
         super({
           minTipWei: 0n,
           minBaseFeeWei: 0n,
           baseFeeMultiplier: 3n,
           tipMultiplier: /* per-chain */,
           gasEstimateBufferPct: 120n,
           needsPrivateRelay: /* true if watcher uses relay */,
           confirmationTimeout: 60_000,
         });
       }
     }
     ```
     Override `rescueGasFallback` only if the chain's gas profile genuinely differs (like Polygon/Monad).
   - `tokens.ts` — curated tokens.

2. **Register in `packages/chains/src/index.ts`:**
   - Import + re-export the chain, strategy, and tokens.
   - Add to `CHAINS` array.
   - Add a `case` in `getRescueStrategy()`.

3. **Update the test** `packages/chains/src/index.test.ts` — the hardcoded chain-id list (`[10, 56, 137, 143, 8453]`) must include the new chain id.

4. **Add the icon** (optional — falls back to a letter avatar):
   - `components/ui/index.tsx` — add a `MyChainLogo` component.
   - `hooks/useChainOptions.tsx` — add `myChain: () => <MyChainLogo />` to `CHAIN_ICONS`.

5. **Lending/quote support (only if the chain has it):**
   - Populate `aave`, `swapRouter`, `uniswap`, `aaveOracle` on the chain config, or lending silently returns empty.

### What you do NOT touch
- **Web pages / review modals / execution** — they all read `CHAINS` via `useChainOptions()`, `findChain()`, `publicRpcUrl()`, and `getRescueStrategy()`. Adding to `CHAINS` makes the chain appear everywhere automatically.
- **`shared/utils.ts`** — the RPC is now per-chain config (`publicRpc`), **not** a global map. Do NOT add to a `publicRpc(chainId)` map (that is gone). The only per-chain exception is the `ALCHEMY_UNSUPPORTED_SLUGS` list (`['bsc']`) — add a slug there only if Alchemy CORS-blocks that chain's browser requests (BSC is the known case).

### The completeness guarantee
- `publicRpc` is a **required** field on `RescueChain`. If missing, the type/`ethereumLike` throws — you cannot silently ship a chain with no RPC.
- `feeRecipient` + `feeBps` are required, so a chain without fee config fails at build.
- A chain with no strategy in `getRescueStrategy()` returns `null` and execution throws immediately (no silent no-op).
- Every strategy must implement all `ChainRescueStrategy` methods. If you extend `EthereumLikeRescueStrategy` you get them for free; only override what genuinely differs.

---

## 8. Known gaps / honest notes

- **The contract fee-enforcement vulnerability is still open.** The builder now routes native sweeps through the executor's auto-split path, but a hand-crafted non-empty-data native call can still bypass the on-chain fee. This requires a contract change + redeploy + fork test — **not yet done.**
- **Review modal vs. execution gas can differ.** Review modal uses `rescueGasFallback` (formula) for units; execution overwrites with the live `estimateGas` when it succeeds. Since the live estimate may differ from the formula, the two can disagree — but the review modal is conservative (budgets at worst-case), so this is safe.
- **`feeBps` is currently `1500n` (15%)** across all live chains. This is a config value; the deployed executor must match it. If you change it in the contract, update `feeBps` in every chain config to match. The header comments in `base/config.ts` and `monad/config.ts` have been updated to match this 15% — keep them in sync.
- **The public RPC choice** per chain (e.g. `polygon.drpc.org`, `bsc-dataseed.binance.org`) is a public endpoint. If it rate-limits or goes down, rescues on that chain can fail — consider a more reliable provider or a fallback list in the future.
- **`broadcastNftFollowUp` (mode-3 NFT following) is the safety net for minted NFTs the in-tx sweep missed** (e.g. mint quantity > 1, unpredictable tokenIds, or ERC-1155 batches). It builds one mode-3 call per minted token and uses `strategy.rescueGasFallback({ kind: 'transfer', callCount: calls.length })` as its gas floor (scales with the number of tokens) — not a hardcoded literal. Note: it is imported but **not currently wired into the UI flow**; the common path is handled in-tx by the mode-7 `mint-forward` callback.
- **Lending chains must declare Aave/swap config.** Base and Monad have full `aave`/`swapRouter`/`uniswap`/`aaveOracle`. A chain that supports lending but omits these will silently scan to zero positions. Add them when enabling lending on a chain.
