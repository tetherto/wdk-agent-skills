# integration-patterns — WDK + vaults.fyi Recipes

## End-to-End: Discover → Deposit with WDK

The most common integration pattern: find the best yield for a user's assets and execute a deposit using a WDK wallet.

```typescript
import { VaultsSdk } from '@vaultsfyi/sdk'
import WDK from '@tetherto/wdk'
import WalletManagerEvm from '@tetherto/wdk-wallet-evm'

// 1. Set up WDK wallet
const wdk = new WDK(seedPhrase)
  .registerWallet('ethereum', WalletManagerEvm, { provider: 'https://eth.drpc.org' })

const account = await wdk.getAccount('ethereum', 0)
const walletAddress = await account.getAddress()

// 2. Set up vaults.fyi SDK
const sdk = new VaultsSdk({ apiKey: process.env.VAULTS_FYI_API_KEY })

// 3. Find idle assets
const idle = await sdk.getIdleAssets({
  path: { userAddress: walletAddress },
  query: { allowedNetworks: ['mainnet', 'base', 'arbitrum', 'optimism', 'polygon'] },
})

// 4. Get best deposit option
const bestVault = await sdk.getBestVault({
  path: { userAddress: walletAddress },
})

// 5. Get transaction context
const context = await sdk.getTransactionsContext({
  path: { userAddress: walletAddress, network: bestVault.network, vaultId: bestVault.vaultId },
})

// 6. Build deposit transaction
const txData = await sdk.getActions({
  path: { action: 'deposit', userAddress: walletAddress, network: bestVault.network, vaultId: bestVault.vaultId },
  query: { assetAddress: bestVault.asset.address, amount: depositAmount },
})

// 7. Execute with WDK wallet (each action in order, wait for confirmation)
try {
  for (const action of txData.actions) {
    const tx = await account.sendTransaction({
      to: action.tx.to,
      value: BigInt(action.tx.value || '0'),
      data: action.tx.data,
    })
    await tx.wait() // wait for confirmation before sending the next action
    console.log(`${action.description}: ${tx.hash}`)
  }
} finally {
  account.dispose()
}
```

## Yield Comparison for WDK Lending Users

WDK's Aave module only covers Aave V3. vaults.fyi lets agents compare Aave against 80+ other protocols.

```typescript
// "Should I use Aave or is there something better for my USDC on Ethereum?"
const { data } = await sdk.getAllVaults({
  query: {
    allowedAssets: ['USDC'],
    allowedNetworks: ['mainnet'],
    sortBy: 'apy7day',
    sortOrder: 'desc',
    perPage: 5,
    onlyTransactional: true,
  },
})

// Present ranked options: protocol, APY, TVL, reputation score
// User can then choose Aave (via WDK lending module) or another protocol (via vaults.fyi transactions)
```

## Multi-Chain Yield Scan

Search across all supported networks:

```typescript
const networks = await sdk.getNetworks()
const networkSlugs = networks.map(n => n.name)

const result = await sdk.getAllVaults({
  query: {
    allowedAssets: ['USDC'],
    allowedNetworks: networkSlugs,
    sortBy: 'apy7day',
    sortOrder: 'desc',
    perPage: 10,
    minTvl: 1_000_000,
  },
})
```

## x402 Pay-Per-Request with WDK Wallet

WDK wallets holding USDT0 can pay for vaults.fyi API calls via x402. No API key needed.

```typescript
import { VaultsSdk } from '@vaultsfyi/sdk'
import WalletManagerEvm from '@tetherto/wdk-wallet-evm'
import { x402Client, wrapFetchWithPayment } from '@x402/fetch'
import { registerExactEvmScheme } from '@x402/evm/exact/client'

// WDK wallet on Base (for USDC x402 payments)
const account = await new WalletManagerEvm(process.env.SEED_PHRASE, {
  provider: 'https://mainnet.base.org'
}).getAccount()

const client = new x402Client()
registerExactEvmScheme(client, { signer: account })
const signer = wrapFetchWithPayment(fetch, client)

// SDK supports x402 via a signer client
const sdk = new VaultsSdk({ client: signer })
```

## Position Monitoring

Track existing positions and earned yield:

```typescript
// Get all positions
const positions = await sdk.getPositions({
  path: { userAddress: walletAddress },
  query: {
    allowedNetworks: ['mainnet', 'base', 'arbitrum', 'optimism', 'polygon'],
    sortBy: 'balanceUsd',
    sortOrder: 'desc',
  },
})

// Get returns for a specific position
const returns = await sdk.getUserVaultTotalReturns({
  path: { userAddress: walletAddress, network, vaultId },
})

// Compare position APY against benchmark
const benchmark = await sdk.getBenchmarks({
  path: { network },
  query: { code: 'usd' },
})
```

## Error Handling

```typescript
try {
  const result = await sdk.getAllVaults({ query: { allowedNetworks: ['invalid'] } })
} catch (error) {
  // SDK throws HttpResponseError with error.message
  // e.g. "Too Many Requests", "Not Found"
  console.error('vaults.fyi SDK error:', error.message)
}
```

Common errors:
- `"Bad Request"` — invalid parameters (check filter values, network slugs)
- `"Unauthorized"` — missing or invalid API key
- `"Not Found"` — vault or position not found (check vaultId and network)
- `"Too Many Requests"` — rate limited (back off and retry)
