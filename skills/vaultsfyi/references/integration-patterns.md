# integration-patterns — WDK + vaults.fyi Recipes

## End-to-End: Discover → Deposit with WDK

The most common integration pattern: find the best yield for a user's assets and execute a deposit using a WDK wallet.

```javascript
import WDK from '@tetherto/wdk'
import WalletManagerEvm from '@tetherto/wdk-wallet-evm'

// 1. Set up WDK wallet
const wdk = new WDK(seedPhrase)
  .registerWallet('ethereum', WalletManagerEvm, { provider: 'https://eth.drpc.org' })

const account = await wdk.getAccount('ethereum', 0)
const walletAddress = await account.getAddress()

const headers = { 'x-api-key': process.env.VAULTSFYI_API_KEY }

// 2. Find idle assets
const idleRes = await fetch(
  `https://api.vaults.fyi/v2/portfolio/idle-assets/${walletAddress}?` +
  new URLSearchParams({ allowedNetworks: 'mainnet,base,arbitrum,optimism,polygon' }),
  { headers }
)
const idle = await idleRes.json()

// 3. Get best deposit option
const bestRes = await fetch(
  `https://api.vaults.fyi/v2/portfolio/best-vault/${walletAddress}`,
  { headers }
)
const bestVault = await bestRes.json()

// 4. Present to user and get confirmation
// AGENT MUST SHOW: vault name, protocol, network, APY, TVL, amount
// AGENT MUST WAIT for explicit user approval

// 5. Get transaction context
const ctxRes = await fetch(
  `https://api.vaults.fyi/v2/transactions/context/${walletAddress}/${bestVault.network}/${bestVault.vaultId}`,
  { headers }
)
const ctx = await ctxRes.json()

// 6. Build deposit transaction
const txRes = await fetch(
  `https://api.vaults.fyi/v2/transactions/deposit/${walletAddress}/${bestVault.network}/${bestVault.vaultId}?amount=${depositAmount}`,
  { headers }
)
const txData = await txRes.json()

// 7. Execute with WDK wallet (each action in order)
try {
  for (const action of txData.actions) {
    const result = await account.sendTransaction({
      to: action.tx.to,
      value: BigInt(action.tx.value || '0'),
      data: action.tx.data
    })
    console.log(`${action.description}: ${result.hash}`)
  }
} finally {
  account.dispose()
}
```

## Yield Comparison for WDK Lending Users

WDK's Aave module only covers Aave V3. vaults.fyi lets agents compare Aave against 80+ other protocols.

```javascript
// "Should I use Aave or is there something better for my USDC on Ethereum?"
const vaults = await fetch(
  'https://api.vaults.fyi/v2/detailed-vaults?' + new URLSearchParams({
    allowedAssets: 'USDC',
    allowedNetworks: 'mainnet',
    sortBy: 'apy7day',
    sortOrder: 'desc',
    perPage: '5',
    onlyTransactional: 'true'
  }),
  { headers: { 'x-api-key': process.env.VAULTSFYI_API_KEY } }
)
const { data } = await vaults.json()

// Present ranked options: protocol, APY, TVL, reputation score
// User can then choose Aave (via WDK lending module) or another protocol (via vaults.fyi transactions)
```

## Multi-Chain Yield Scan

Search across all supported networks:

```javascript
const allNetworksRes = await fetch(
  'https://api.vaults.fyi/v2/networks',
  { headers: { 'x-api-key': process.env.VAULTSFYI_API_KEY } }
)
const networks = await allNetworksRes.json()
const networkSlugs = networks.map(n => n.name).join(',')

const vaults = await fetch(
  'https://api.vaults.fyi/v2/detailed-vaults?' + new URLSearchParams({
    allowedAssets: 'USDC',
    allowedNetworks: networkSlugs,
    sortBy: 'apy7day',
    sortOrder: 'desc',
    perPage: '10',
    minTvl: '1000000'
  }),
  { headers: { 'x-api-key': process.env.VAULTSFYI_API_KEY } }
)
```

## x402 Pay-Per-Request with WDK Wallet

WDK wallets holding USDT0 can pay for vaults.fyi API calls via x402. No API key needed.

```javascript
import WalletManagerEvm from '@tetherto/wdk-wallet-evm'
import { x402Client, wrapFetchWithPayment } from '@x402/fetch'
import { registerExactEvmScheme } from '@x402/evm/exact/client'

// WDK wallet on Base (for USDC x402 payments)
const account = await new WalletManagerEvm(process.env.SEED_PHRASE, {
  provider: 'https://mainnet.base.org'
}).getAccount()

const client = new x402Client()
registerExactEvmScheme(client, { signer: account })
const paidFetch = wrapFetchWithPayment(fetch, client)

// Now use paidFetch instead of fetch — 402 payments handled automatically
const vaults = await paidFetch(
  'https://api.vaults.fyi/v2/detailed-vaults?allowedAssets=USDC&sortBy=apy7day&sortOrder=desc'
)
```

## Position Monitoring

Track existing positions and earned yield:

```javascript
// Get all positions
const positions = await fetch(
  `https://api.vaults.fyi/v2/portfolio/positions/${walletAddress}?` +
  new URLSearchParams({
    allowedNetworks: 'mainnet,base,arbitrum,optimism,polygon',
    sortBy: 'balanceUsd',
    sortOrder: 'desc'
  }),
  { headers }
)

// Get returns for a specific position
const returns = await fetch(
  `https://api.vaults.fyi/v2/portfolio/total-returns/${walletAddress}/${network}/${vaultId}`,
  { headers }
)

// Compare position APY against benchmark
const benchmark = await fetch(
  `https://api.vaults.fyi/v2/benchmarks/${network}`,
  { headers }
)
```

## Error Handling

```javascript
const response = await fetch(url, { headers })
if (!response.ok) {
  const error = await response.json()
  // error.error — error type
  // error.message — human-readable description
  // error.errorId — for support reference
  throw new Error(`vaults.fyi API error: ${error.message}`)
}
```

Common error codes:
- `400` — invalid parameters (check filter values, network slugs)
- `401` — missing or invalid API key
- `404` — vault or position not found (check vaultId and network)
- `429` — rate limited (back off and retry)
