# api-portfolio — Positions, Idle Assets & Recommendations

## Base URL

```
https://api.vaults.fyi/v2
```

## Portfolio Positions

### List All Positions
```
GET /v2/portfolio/positions/{userAddress}
```
```bash
curl "https://api.vaults.fyi/v2/portfolio/positions/0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045?sortBy=balanceUsd&sortOrder=desc" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Returns all vault positions for a wallet across all networks and protocols.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `allowedNetworks` | string | `base,mainnet,arbitrum,optimism` | Must set explicitly for full coverage |
| `sortBy` | string | `balanceUsd` | One of: `balanceUsd`, `tvl`, `apy1Day`, `apy7Day`, `apy30Day` (**camelCase**, unlike `/detailed-vaults`) |
| `sortOrder` | string | `asc` | `asc` or `desc` |
| `perPage` | number | 50 | Results per page |
| `page` | number | 0 | Zero-indexed |

**Response:** Array of position objects with token balances, current value in USD, vault details, and unclaimed rewards.

### Get Single Position
```
GET /v2/portfolio/positions/{userAddress}/{network}/{vaultId}
```
```bash
curl "https://api.vaults.fyi/v2/portfolio/positions/0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045/mainnet/0x0B6C8ef0DE1Be5ed1B59E6e7a67fB9442FB9E49C" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Detailed view of one position with cost basis, current value, and P&L.

### Get Total Returns
```
GET /v2/portfolio/total-returns/{userAddress}/{network}/{vaultId}
```
```bash
curl "https://api.vaults.fyi/v2/portfolio/total-returns/0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045/mainnet/0x0B6C8ef0DE1Be5ed1B59E6e7a67fB9442FB9E49C" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Calculates earned yield for a specific position. Returns native and USD amounts.

### List Position Events
```
GET /v2/portfolio/events/{userAddress}/{network}/{vaultId}
```
```bash
curl "https://api.vaults.fyi/v2/portfolio/events/0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045/mainnet/0x0B6C8ef0DE1Be5ed1B59E6e7a67fB9442FB9E49C" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Chronological deposit, withdraw, and claim history with transaction metadata.

## Idle Assets

### List Idle Assets
```
GET /v2/portfolio/idle-assets/{userAddress}
```
```bash
curl "https://api.vaults.fyi/v2/portfolio/idle-assets/0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045?allowedNetworks=mainnet,base,arbitrum,optimism" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Returns token balances not currently earning yield across all networks.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `allowedNetworks` | string | `base,mainnet,arbitrum,optimism` | Must set explicitly for full coverage |

Use this to identify deposit opportunities. Pair with `/best-deposit-options` or `/best-vault`.

## Recommendations

### Get Best Deposit Options
```
GET /v2/portfolio/best-deposit-options/{userAddress}
```
```bash
curl "https://api.vaults.fyi/v2/portfolio/best-deposit-options/0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045?onlyTransactional=true&maxVaultsPerAsset=3" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Analyzes wallet balances and recommends optimal vaults for each idle asset.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `onlyTransactional` | boolean | false | Only recommend vaults with deposit support |
| `maxVaultsPerAsset` | number | 3 | Max recommendations per token |
| `minimumTvl` | number | 100000 | Minimum vault TVL |
| `protocols` | string | all | Filter by protocol |
| `networks` | string | default 4 | Filter by network |

**Response:** Ranked list of vaults matching the user's idle token holdings, grouped by asset.

### Get Best Single Vault
```
GET /v2/portfolio/best-vault/{userAddress}
```
```bash
curl "https://api.vaults.fyi/v2/portfolio/best-vault/0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045?onlyTransactional=true" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Returns the single best vault opportunity for the wallet. Ideal for "one-tap earn" flows.

**Parameters:** Same as `best-deposit-options`.

**Response:** One vault recommendation with full detail (APY, TVL, protocol, risk).

## Important Notes

- `sortBy` casing differs from `/detailed-vaults`: use `apy7Day` (camelCase) here, not `apy7day` (lowercase). Wrong casing silently falls back to default sort.
- `allowedNetworks` defaults to 4 networks on all portfolio endpoints. Pass the full list to avoid missing positions on other chains.
- All APY values are raw decimals (multiply by 100 for percentage display).
