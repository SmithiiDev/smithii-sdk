# CLAUDE.md — @smithii/sdk

Context for AI coding assistants integrating `@smithii/sdk` into a project.
This document mirrors the actual API surface — types, methods, and limits
are taken from the source.

## What this is

Framework-agnostic TypeScript SDK that exposes Smithii's on-chain tools
across Solana, EVM, and SUI. Runs in Node, Bun, browsers, and edge runtimes.
**Never imports React, Chakra, or any wallet-adapter-react.**

## Install

```bash
npm i @smithii/sdk
```

Peer deps (install only what you use):

```bash
# Solana
npm i @solana/web3.js @solana/spl-token @coral-xyz/anchor

# EVM
npm i viem

# SUI
npm i @mysten/sui
```

## Subpath exports

Always import from a subpath — the root barrel is large.

```typescript
import { PumpFunClient }      from '@smithii/sdk/pump'
import { PumpSwapClient }     from '@smithii/sdk/pumpswap'   // alias of PumpFunClient
import { LaunchpadClient,
         BonkClient,
         LaunchlabClient }    from '@smithii/sdk/launchpad'
import { MoonitClient }       from '@smithii/sdk/moonit'
import { AntiMEVClient }      from '@smithii/sdk/anti-mev'
import { TokenCreatorClient } from '@smithii/sdk/token-creator'
import { TokenManagerClient } from '@smithii/sdk/token-manager'
import { TokenVestingClient } from '@smithii/sdk/token-vesting'
import { TokenClaimClient }   from '@smithii/sdk/token-claim'
import { MultiSenderClient }  from '@smithii/sdk/multisender'
import { MarketMakerClient }  from '@smithii/sdk/market-maker'
import { MantisClient }       from '@smithii/sdk/mantis'
import { PaymentClient }      from '@smithii/sdk/payment'

import { EvmTokenCreatorClient } from '@smithii/sdk/evm/token-creator'
import { EvmMultisenderClient }  from '@smithii/sdk/evm/multisender'
import { EvmSnapshotClient }     from '@smithii/sdk/evm/snapshot'

import { SuiTokenCreatorClient } from '@smithii/sdk/sui/token-creator'
import { SuiSnapshotClient }     from '@smithii/sdk/sui/snapshot'
```

## Signer pattern (Solana)

Every Solana client takes a `Signer`:

```typescript
interface Signer {
  readonly publicKey: PublicKey
  signTransaction<T>(tx: T): Promise<T>
  signAllTransactions<T>(txs: T[]): Promise<T[]>
  signMessage?(message: Uint8Array): Promise<Uint8Array>
}
```

Compatible inputs:
- `Keypair` from `@solana/web3.js` (wrap with the SDK's `keypairToSigner` if needed)
- `useWallet()` return value from `@solana/wallet-adapter-react`
- Any object that satisfies the interface

EVM clients take `viem.WalletClient`. SUI clients take `SuiSigner` (same shape from `@mysten/sui`).

---

## Solana clients

### PumpFunClient — `@smithii/sdk/pump`

Pump.fun bundler. Also exported as `PumpSwapClient` from `@smithii/sdk/pumpswap`
for graduated tokens.

```typescript
new PumpFunClient({
  connection: Connection,
  signer: Signer,
  jito: JitoConfig,
  paymentAddresses?: Partial<PaymentProgramAddresses>,
  proxyUrl?: string,
  http?: HttpClient,
})

// Methods
uploadMetadata(args): Promise<PumpMetadataResponse>
createAndSnipeToken(args: PumpCreateAndSnipeArgs): Promise<{
  createTxSignature: string
  buyerTxSignatures: string[]
  bundleIds: string[]
  paymentSignature: string
}>
bundleSellBuy(args: PumpBundleSellBuyArgs): Promise<{
  bundleIds: string[]
  txSignatures: string[]
  paymentSignature: string
}>
```

**Limits:**
- `createAndSnipeToken`: max **16 buyers** (split across Jito bundles, 5 txs each)
- `bundleSellBuy`: max **25 wallets**
- Bundle 1 must land before subsequent bundles (bonding curve requirement)
- Pre-generated (vanity) tokens incur double fee — validated via `isTokenPregenerated`
- Cashback coins (`isCashbackCoin: true`) use `TOKEN_2022_PROGRAM_ID`

### LaunchpadClient / BonkClient / LaunchlabClient — `@smithii/sdk/launchpad`

Raydium LaunchLab + LetsBonk.fun bundler.

```typescript
new LaunchpadClient({
  connection: Connection,
  signer: Signer,
  jito: JitoConfig,
  platformId?: PublicKey,                       // undefined = Raydium default
  paymentAddresses?: Partial<PaymentProgramAddresses>,
  http?: HttpClient,
})

// BonkClient = LaunchpadClient with BONK_PLATFORM_ID preset
// LaunchlabClient = LaunchpadClient with platformId undefined

createAndSnipe(args: LaunchpadCreateAndSnipeArgs): Promise<{
  mint: PublicKey
  bundleId: string
  createTxSignature: string
  buyerTxSignatures: string[]
  paymentSignature: string
  poolInfo: unknown
}>
```

**Limits:**
- Max **4 buyers** (Jito 5-tx limit: create + 4 buyers in one bundle)
- Pre-generated tokens incur double fee
- Payment is signed separately and only confirmed after bundle success

### MoonitClient — `@smithii/sdk/moonit`

```typescript
new MoonitClient({
  connection: Connection,
  signer: Signer,
  jito: JitoConfig,
  moonitRpcUrl?: string,
  environment?: Environment,                    // defaults to MAINNET
  paymentAddresses?: Partial<PaymentProgramAddresses>,
  http?: HttpClient,
})

get sdk: Moonit                                 // raw Moonit SDK instance

createAndSnipe(args: MoonitCreateAndSnipeArgs): Promise<{
  mint: PublicKey
  tokenId: string
  bundleId: string
  mintSignature: string
  buyerSignatures: string[]
  paymentSignature: string
}>
bundleSwap(args: MoonitBundleSwapArgs): Promise<{
  bundleIds: string[]
  txSignatures: string[]
  paymentSignature: string
}>
```

**Limits:**
- `createAndSnipe`: max **3 buyers** (mint + tip + 3 = 5 txs)
- `bundleSwap`: max **4 wallets** (4 swaps + tip = 5 txs)
- `privKeys.length` must equal `amounts.length`

### AntiMEVClient — `@smithii/sdk/anti-mev`

Sandwich-proof volume bot. Backend orchestrates the bundles.

```typescript
new AntiMEVClient({
  connection: Connection,
  signer: Signer,
  apiBaseUrl: string,
  variant?: 'pump' | 'pumpswap' | 'printr',     // defaults to 'pump'
  http?: HttpClient,
})

runSingle(config: AntiMEVSingleConfig): Promise<{ depositSignature: string }>
runMultiple(config: AntiMEVMultipleConfig): Promise<AntiMEVRunMultipleResponse>
getHistory(walletAddress?: string): Promise<AntiMEVHistory>
```

**Notes:**
- `runSingle`: backend builds deposit tx, user signs, SDK polls confirmation (60 attempts × 2s)
- `runMultiple`: pure backend orchestration with private keys — no client-side tx
- Validates: token address, antiMEVUses, amount/delay ranges before submission

### TokenCreatorClient — `@smithii/sdk/token-creator`

```typescript
new TokenCreatorClient({
  connection: Connection,
  signer: Signer,
  backend: BackendConfig,
  paymentAddresses?: Partial<PaymentProgramAddresses>,
  lookupTable?: PublicKey,
  tokenProgram?: PublicKey,                     // defaults TOKEN_PROGRAM_ID
})

createToken(args: CreateTokenArgs): Promise<{
  mint: PublicKey
  signature: string
  estimatedFeesSol: number
  metadataUri: string
}>
updateMetadata(args: UpdateMetadataArgs): Promise<{
  signature: string
  estimatedFeesSol: number
  metadataUri: string
}>
```

**Critical:**
- `isMutable` is **inverted on-chain**: SDK sends `!args.isMutable` because the
  front semantic is "true = locked". Pass `isMutable: true` if you want the
  metadata locked.
- Charges `RevokeMutability` fee when `isMutable === true` on create
- TX size validated to be ≤ 1232 bytes — large airdrops will fail; use
  MultiSenderClient for that
- Supports both `TOKEN_PROGRAM_ID` and `TOKEN-2022`
- DexTools links pushed after on-chain confirmation

### TokenManagerClient — `@smithii/sdk/token-manager`

```typescript
new TokenManagerClient({
  connection: Connection,
  signer: Signer,
  umiRpcUrl?: string,
  paymentAddresses?: Partial<PaymentProgramAddresses>,
  http?: HttpClient,
  priorityFeeMicroLamports?: number,            // static fallback
})

revokeAuthority(args: RevokeAuthorityArgs): Promise<{ signature: string }>
increaseSupply(args: IncreaseSupplyArgs): Promise<{ signature: string }>
makeImmutable(args: MakeImmutableArgs): Promise<{ signature: string }>
snapshotTokenHolders(args): Promise<{ signature: string, holders: TokenHolder[] }>
snapshotNftCollectionHolders(args): Promise<{ signature: string, ...NftCollectionHolderResult }>
```

**Notes:**
- `RevokeAuthorityArgs.type`: `'mint' | 'freeze'`
- All paid actions atomically bundle Smithii payment ix + action ixs
- NFT snapshot needs `creatorAddress` and a DAS-enabled `rpcUrl` (Helius)

### TokenVestingClient — `@smithii/sdk/token-vesting`

```typescript
new TokenVestingClient({
  connection: Connection,
  signer: Signer,
  paymentWallet?: PublicKey,
})

initVesting(args): Promise<{ signature: string, vestingPda: PublicKey }>
claimVesting(args): Promise<{ signature: string, claimPda: PublicKey }>
getVestingState(vestingAddress: PublicKey, decimals: number): Promise<VestingState | null>
getClaimedPercent(claimAddress: PublicKey): Promise<number>
findVestingPda(mint: PublicKey, authority: PublicKey): { vesting: PublicKey, bump: number }
findClaimPda(vesting: PublicKey, signer: PublicKey): PublicKey
```

**Notes:** `receivers: string[]` empty array = public vesting (any wallet can claim).

### TokenClaimClient — `@smithii/sdk/token-claim`

```typescript
new TokenClaimClient({
  connection: Connection,
  signer: Signer,
  paymentWallet?: PublicKey,
})

initialize(args): Promise<{ signature: string, stakePda: PublicKey }>
claim(args): Promise<{ signature: string, claimPda: PublicKey }>
withdrawRemaining(args): Promise<string>
getState(stakeAddress: PublicKey, decimals: number): Promise<TokenClaimState | null>
getClaimedPercent(claimAddress: PublicKey): Promise<number>
findStakePda(mint, authority): { stake: PublicKey, bump: number }
findTokenClaimPda(stake, signer): PublicKey
```

### MultiSenderClient — `@smithii/sdk/multisender`

```typescript
new MultiSenderClient({
  connection: Connection,
  signer: Signer,
  backend: BackendConfig,
  paymentAddresses?: Partial<PaymentProgramAddresses>,
  lookupTable?: PublicKey,
})

sendAirdrop(args): Promise<{ ...SignAndSendBatchesResult, totalRecipients: number }>
scheduleAirdrop(args): Promise<{ signature: string, airdropId: string }>
getHistory(wallet: PublicKey): Promise<AirdropResponse[]>
```

**Notes:**
- `amountMode: 'same' | 'different'`
- `wallets.length` must be > 0 and ≤ `AIRDROPPER_LIMIT`
- Scheduled airdrops require an SPL token (native SOL not supported)
- Lookup table is cached after first fetch

### MarketMakerClient — `@smithii/sdk/market-maker`

```typescript
new MarketMakerClient({
  connection: Connection,
  signer: Signer,
  paymentAddresses?: Partial<PaymentProgramAddresses>,
  vault?: PublicKey,
  http?: HttpClient,
})

isTokenTradable(mint: string): Promise<boolean>
deposit(args: MarketMakerDepositArgs): Promise<{ signature: string }>
```

### MantisClient — `@smithii/sdk/mantis`

```typescript
new MantisClient({
  connection: Connection,
  signer: Signer,
  paymentWallet?: PublicKey,
  usdcMint?: PublicKey,
  usdtMint?: PublicKey,
})

initialize(args): Promise<{ signature: string, launchPda: PublicKey }>
edit(args): Promise<string>                                    // signature only
buy(args): Promise<{ signature: string, userLaunchPda: PublicKey }>
claim(args): Promise<{ signature: string, userLaunchPda: PublicKey }>
withdraw(args): Promise<{ signature: string, withdrawalObserverPda: PublicKey }>
getLaunchState(launchAddress: PublicKey): Promise<MantisLaunchState | null>
getUserLaunchState(mint, launchAuthority, user?): Promise<MantisUserLaunchState>
getWithdrawalState(launchAddress, authority): Promise<MantisWithdrawalState | null>
findLaunchPda(mint, authority): { launch: PublicKey, bump: number }
findUserLaunchPda(mint, launch, user): { userLaunchAccount: PublicKey, bump: number }
findWithdrawalObserverPda(launch, authority): PublicKey
```

**Notes:** `BuyArgs.paymentMethod: 'SOL' | 'USDC' | 'USDT'`.

### PaymentClient — `@smithii/sdk/payment`

Read-only. No signer needed for state queries.

```typescript
new PaymentClient({
  connection: Connection,
  addresses?: Partial<PaymentProgramAddresses>,
})

getTools(): Promise<{ prices: number[], accumulatedFees: number[], count: number }>
getReferral(user: PublicKey): Promise<ReferralAccount>
getPlan(user: PublicKey): Promise<PlanAccount>
```

**Notes:** `PlanEnum: 'DegenPass' | 'TrenchesPass' | 'BasedPass' | 'OGPass'`.
On error, `getReferral`/`getPlan` return `EMPTY_*` constants instead of throwing.

---

## EVM clients

### EvmTokenCreatorClient — `@smithii/sdk/evm/token-creator`

```typescript
new EvmTokenCreatorClient({
  walletClient: WalletClient,
  registry?: EvmRegistry,
  contractAddress?: Address,
})

deployErc20(args: DeployErc20Args): Promise<{ tokenAddress: Address, hash: Hex }>
```

**Notes:**
- Validates: name ≤ 32, symbol ≤ 8, supply > 0, tax ∈ [0, 20]
- Token address is computed locally via `getCreate2Address()` — deterministic

### EvmMultisenderClient — `@smithii/sdk/evm/multisender`

```typescript
new EvmMultisenderClient({
  walletClient: WalletClient,
  publicClient?: PublicClient,
  registry?: EvmRegistry,
  contractAddress?: Address,
})

airdrop(args: EvmAirdropConfig): Promise<{ hash: Hex, totalAmount: number, totalFee: number }>
```

**Notes:** SDK adds `feePerWallet * wallets.length` as native value automatically.

### EvmSnapshotClient — `@smithii/sdk/evm/snapshot`

```typescript
new EvmSnapshotClient({
  walletClient: WalletClient,
  publicClient: PublicClient,
  registry?: EvmRegistry,
  paymentsAddressOverride?: Address,
})

snapshotErc20Holders(args): Promise<{ paymentHash: Hex, ...FetchHoldersResult }>
snapshotErc721Holders(args): Promise<{ paymentHash: Hex, ...FetchHoldersResult }>
```

**Notes:** Pays first via private payment step, then scans block ranges
(default chunk: 2500 for ERC20, 30000 for ERC721).

---

## SUI clients

### SuiTokenCreatorClient — `@smithii/sdk/sui/token-creator`

```typescript
new SuiTokenCreatorClient({
  signer: SuiSigner,
  network: SuiNetwork,                          // preset or raw RPC URL
  paymentAddress?: string,
  suiClient?: SuiClient,
})

deploy(args: DeployTokenArgs): Promise<{
  digest: string
  explorer: string
  treasuryAddress: string
  treasuryObjectType: string
  coinType: string
  coinAddress: string
  paymentDigest: string
}>
configure(args): Promise<{ digest, explorer } | null>
mint(params: SuiMintParams): Promise<{ digest, explorer }>
mergeOwnedCoins(coinType: string): Promise<{ digest, explorer } | null>
```

**Notes:**
- Fee (7.5 SUI default) is paid before deploy. `skipPayment: true` is internal/dev only.
- `configure` returns null if no configuration is needed
- `mergeOwnedCoins` returns null if fewer than 2 coins exist

### SuiSnapshotClient — `@smithii/sdk/sui/snapshot`

```typescript
new SuiSnapshotClient({
  apiBaseUrl: string,
  http?: HttpClient,
})

getTokenHolders(coinType: string, name: string): Promise<string[]>
getTokenInfo(coinType: string): Promise<unknown>
saveTokenSnapshot(args): Promise<SuiTokenSnapshotRecord>
getNftCollectionHolders(slug: string): Promise<string[]>
getNftInfo(slug: string): Promise<unknown>
saveNftSnapshot(args): Promise<SuiNftSnapshotRecord>
getPreviousUserSnapshots(accountAddress: string): Promise<SuiTokenSnapshotRecord[]>
```

**Notes:** Backend wrapper only — caller pays SUI fee separately and threads
the digest into the save call.

---

## Shared types

```typescript
interface JitoConfig {
  uuid: string
  proxyUrl?: string
  bundleEndpoint?: string
}

interface BackendConfig {
  baseUrl: string
  http?: HttpClient
}

type PriorityFeesLevel = 'fast' | 'turbo' | 'ultra'

interface HttpClient {
  get<T = unknown>(url: string, options?: HttpRequestOptions): Promise<T>
  post<T = unknown>(url: string, body?: unknown, options?: HttpRequestOptions): Promise<T>
}
```

## Error handling

All clients throw typed errors that extend `SmithiiError`:

| Class | Thrown when |
|-------|-------------|
| `ConfigError` | Missing/invalid constructor config |
| `ValidationError` | Invalid argument at API boundary |
| `HttpError` | Backend HTTP failure (has `status`, `url`, `body`) |
| `RpcError` | Solana RPC failure |
| `BundleError` | Jito bundle failed (has `bundleId?`) |
| `TransactionFailedError` | On-chain tx failed (has `signature`, `onChainError`) |
| `TransactionTimeoutError` | Confirmation timeout (has `signature`, `attempts`) |

```typescript
import { SmithiiError, BundleError } from '@smithii/sdk/core'

try {
  const result = await client.createAndSnipeToken({ ... })
} catch (err) {
  if (err instanceof BundleError) {
    console.error('Bundle failed', err.bundleId, err.message)
  } else if (err instanceof SmithiiError) {
    console.error('SDK error', err.message)
  } else {
    throw err
  }
}
```

## On-chain invariants — never work around these

These are protocol-level constants. Changing or bypassing any of them
produces silently incorrect transactions.

- **Payment is charged AFTER the main tx confirms.** Never reorder.
- **Pump.fun token math:** `tokenAmount = solAmount * 35_707_140 * 1e6`,
  `maxSolCost = solAmount * 1.1 * 1e9`.
- **Slippage values are deliberate:** Pump buys 10%, Pump sells 0% min-out,
  Moonit first buy 25%, Raydium launchpad creation 99.99%.
- **Jito limits:** max 5 txs per bundle, 500_000 lamports tip on Pump,
  compute price 200_000 microLamports.
- **Wallet limits per call:** Pump 16/25, Launchpad 4, Moonit 3/4
  (see each client section).
- **Pump.fun upgrade:** after `PUMP_UPGRADE_TIMESTAMP_MS`, BUY/SELL ixs
  must include the extra fee recipient. The SDK handles this automatically.
- **TokenCreator `isMutable` is inverted on-chain.** The SDK sends `!args.isMutable`.

## Common mistakes to avoid

- ❌ `import { ... } from '@smithii/sdk'` — use a subpath, the root barrel
  pulls in every chain's deps.
- ❌ Passing a private key string as `signer`. Wrap with
  `Keypair.fromSecretKey(bs58.decode(...))` first.
- ❌ Manually calling a payment instruction. Every client charges payment
  internally; never duplicate it.
- ❌ Retrying a failed bundler call without changing parameters. If the
  first transaction in a Pump bundle didn't land, the bonding curve state
  may have changed — re-fetch state first.
- ❌ Treating `bundleSellBuy` like a swap. It's atomic across all wallets;
  if one fails, all fail.
- ❌ Using `TokenCreatorClient` for an airdrop > ~30 wallets. Use
  `MultiSenderClient` instead — TX size is enforced at 1232 bytes.

## Environment variables (caller's responsibility)

The SDK never reads `process.env`. All config goes through the constructor.
A typical app sets:

```bash
RPC_URL          # Solana RPC endpoint
JITO_UUID        # Jito Block Engine auth token (for bundlers)
PROXY_URL        # Smithii proxy URL (required for bundlers)
BOTS_API_URL     # Smithii backend API (anti-mev, market-maker, multisender, token-creator)
SUI_RPC_URL      # SUI RPC endpoint (SUI clients)
```

## Links

- Web app: https://tools.smithii.io
- npm: https://www.npmjs.com/package/@smithii/sdk
- Issues: https://github.com/SmithiiDev/smithii-sdk/issues
- Discord: https://discord.gg/3AFfGDfmk7
