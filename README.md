# @smithii/sdk

[![npm version](https://img.shields.io/npm/v/@smithii/sdk?color=blue)](https://www.npmjs.com/package/@smithii/sdk)
[![npm downloads](https://img.shields.io/npm/dm/@smithii/sdk?color=green)](https://www.npmjs.com/package/@smithii/sdk)
[![License](https://img.shields.io/badge/license-Proprietary-red)](./LICENSE)

**Framework-agnostic TypeScript SDK** for automating Smithii's on-chain tools — call them from a server, a bot, a CLI, or any custom UI.

> If you're looking for the no-code version, head to **[tools.smithii.io](https://tools.smithii.io)**.

---

## Let your Agent handle it

Paste this into Claude Code, Cursor, or any agent that supports the [agent skills standard](https://agentskills.io):

```
add smithii-sdk skill https://tools.smithii.io/skill/skill.md
```

Your agent will install the [Smithii SDK skill](https://github.com/SmithiiDev/smithii-sdk-skill) and immediately know how to build pump.fun bundlers, anti-MEV bots, token creators, multisenders, and the rest of the toolset — without hallucinating method signatures or limits.

---

## What you can build

- **Pump.fun bundler bots** — create a token and snipe it with up to 16 wallets atomically in the same block via Jito
- **PumpSwap bundle buy / sell** — multi-wallet trades on graduated pump.fun tokens
- **Launchlab & Bonk (LetsBonk.fun) bundlers** — Raydium LaunchLab token launch + snipe
- **Moonit bundler** — create and snipe on the Moonit protocol
- **Anti-MEV volume bots** — sandwichproof buy+sell bundles in the same block
- **Token creator** — deploy SPL tokens on Solana, EVM chains, and SUI
- **Multisender** — airdrop tokens to thousands of wallets in one transaction
- **Token manager** — revoke authorities, update metadata, snapshot holders
- **Market maker** — automated liquidity and volume management
- **Vesting & claims** — on-chain vesting schedules and claim flows
- **Mantis** — cross-chain bridging flows

## Supported chains

| Chain | Tools |
|-------|-------|
| **Solana** | pump · pumpswap · bonk · launchlab · moonit · token-creator · token-manager · token-vesting · token-claim · multisender · market-maker · anti-mev · mantis · payment |
| **EVM** | token-creator · multisender · snapshot (ETH, Base, BSC, Polygon, Arbitrum, Avalanche, Blast) |
| **SUI** | token-creator · wallet · snapshot |

---

## Install

```bash
npm i @smithii/sdk
```

Install only the peer deps for the chain(s) you target:

```bash
# Solana
npm i @solana/web3.js @solana/spl-token @coral-xyz/anchor

# EVM
npm i viem

# SUI
npm i @mysten/sui
```

---

## Quick start

### Solana — Pump.fun bundler (create token + snipe with multiple wallets)

```typescript
import { Connection, Keypair } from '@solana/web3.js'
import { PumpFunClient } from '@smithii/sdk/pump'

const client = new PumpFunClient({
  connection: new Connection(process.env.RPC_URL!),
  signer: yourSigner,
  jito: { uuid: process.env.JITO_UUID! },
  proxyUrl: process.env.PROXY_URL!,
})

const result = await client.createAndSnipeToken({
  name: 'My Token',
  symbol: 'MTK',
  description: 'My token description',
  image: imageFile,          // File or Blob
  devAmount: 0.5,            // SOL the dev wallet buys
  buyers: [
    { pk: 'base58-priv-key', amount: 0.1 },
    { pk: 'base58-priv-key', amount: 0.2 },
    // up to 16 wallets
  ],
})

console.log('Mint:', result.mint.toBase58())
console.log('Bundle IDs:', result.bundleIds)
```

### Solana — Anti-MEV volume bot

```typescript
import { AntiMEVClient } from '@smithii/sdk/anti-mev'

const client = new AntiMEVClient({
  connection,
  signer,
  backendUrl: process.env.BOTS_API_URL!,
})

// Sandwichproof buy+sell in same block
await client.runSingle({
  mint: 'TokenMintAddress',
  amount: 0.05,      // SOL per cycle
  cycles: 10,
  delayMs: 2000,
})
```

### Solana — Bundle buy existing token (up to 25 wallets)

```typescript
import { PumpFunClient } from '@smithii/sdk/pump'

await client.bundleBuy({
  mint: 'TokenMintAddress',
  buyers: [
    { pk: 'wallet-1-pk', amount: 0.1 },
    { pk: 'wallet-2-pk', amount: 0.15 },
  ],
})
```

### EVM — Token creator + multisender

```typescript
import { EvmTokenCreatorClient } from '@smithii/sdk/evm/token-creator'
import { EvmMultisenderClient } from '@smithii/sdk/evm/multisender'

const creator = new EvmTokenCreatorClient({ walletClient, chain: 'base' })
const { address } = await creator.deploy({ name: 'My Token', symbol: 'MTK', supply: 1_000_000n })

const sender = new EvmMultisenderClient({ walletClient, chain: 'base' })
await sender.send({ token: address, recipients: [{ address: '0x...', amount: 100n }] })
```

---

## How it works

Every tool follows the same pattern:

1. **Build** — the SDK constructs the on-chain instructions (Jito bundles for Solana, calldata for EVM)
2. **Pay** — a small fee is charged via Smithii's on-chain payment program **after** the main tx confirms
3. **Confirm** — the SDK polls for confirmation and returns signatures / bundle IDs

All sensitive config (RPC URLs, private keys, Jito UUID) is passed via constructor — nothing is hardcoded.

---

## Signers

| Chain | Signer type |
|-------|-------------|
| Solana | `useWallet()` from `@solana/wallet-adapter-react`, or a raw `Keypair` |
| EVM | `viem.WalletClient` or `privateKeyToAccount` |
| SUI | `useWallet()` from `@suiet/wallet-kit` |

---

## Links

- **Web app**: [tools.smithii.io](https://tools.smithii.io)
- **npm package**: [@smithii/sdk](https://www.npmjs.com/package/@smithii/sdk)
- **Issues & support**: [GitHub Issues](https://github.com/SmithiiDev/smithii-sdk/issues)
- **Twitter**: [@SmithiiTools](https://x.com/SmithiiTools)

---

## License

Proprietary — see [LICENSE](./LICENSE) for terms. Free to use with a valid Smithii plan.
