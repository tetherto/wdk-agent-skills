# api-discovery — Vault Search & Reference Data

## Base URL

```
https://api.vaults.fyi/v2
```

## Vault Discovery

### List Detailed Vaults
```
GET /v2/detailed-vaults
```

```bash
curl "https://api.vaults.fyi/v2/detailed-vaults?allowedAssets=USDC&sortBy=apy7day&sortOrder=desc&perPage=5" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

The primary search endpoint. Returns vaults with full analytics: APY, TVL, Reputation Score, protocol, curator, and risk data.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `allowedAssets` | string | all | Comma-separated asset symbols (e.g., `USDC,USDT,DAI`) |
| `allowedNetworks` | string | `base,mainnet,arbitrum,optimism` | Comma-separated network slugs. **Must be set explicitly to include other networks.** |
| `disallowedNetworks` | string | none | Networks to exclude |
| `allowedProtocols` | string | all | Filter by protocol name |
| `allowedCurators` | string | all | Filter by curator |
| `allowedTags` | string | all | Filter by tag (e.g., `top-curator`, `lst`, `rwa`) |
| `minTvl` | number | 100000 | Minimum TVL in USD. **Set to 0 to include small vaults.** |
| `maxTvl` | number | none | Maximum TVL in USD |
| `minApy` | number | none | Minimum APY (raw decimal, e.g., `0.05` for 5%) |
| `maxApy` | number | none | Maximum APY |
| `onlyTransactional` | boolean | false | Only return vaults with deposit/withdraw support |
| `sortBy` | string | `tvl` | One of: `tvl`, `apy1day`, `apy7day`, `apy30day` (all lowercase) |
| `sortOrder` | string | `asc` | `asc` or `desc`. **Use `desc` for highest-first.** |
| `perPage` | number | 50 | Results per page (max 5000 on `/v2/vaults`) |
| `page` | number | 0 | Zero-indexed page number |

**Response shape:**
```json
{
  "data": [
    {
      "vaultId": "0x...",
      "address": "0x...",
      "network": "mainnet",
      "protocol": { "name": "Aave", "product": "V3", "version": "3.0" },
      "asset": { "symbol": "USDC", "address": "0x...", "decimals": 6 },
      "tvl": { "usd": 125000000, "token": "125000000.00" },
      "apy": {
        "1day": { "base": 0.0312, "reward": 0.0021, "total": 0.0333 },
        "7day": { "base": 0.0298, "reward": 0.0019, "total": 0.0317 },
        "30day": { "base": 0.0285, "reward": 0.0018, "total": 0.0303 }
      },
      "apyComposite": { "totalApy": 0.0317 },
      "reputationScore": 92,
      "curator": { "name": "Gauntlet" },
      "tags": ["top-curator"],
      "isTransactional": true
    }
  ],
  "nextPage": 1,
  "itemsOnPage": 50
}
```

**APY notes:**
- All APY values are raw decimals. `0.0543` = 5.43%.
- `apy.*.total` = `base` + `reward`. Reward APYs from `rewards[]` are already rolled into `apy.*.reward`. Don't double-count.
- `apyComposite.totalApy` is the correct APY for nested vaults (vaults that deposit into other vaults).
- Recommended display: `apy.7day.total` or `apy.30day.total`. 1-day and 1-hour windows are noisy.

### Get Single Vault
```
GET /v2/detailed-vaults/{network}/{vaultId}
```

```bash
curl "https://api.vaults.fyi/v2/detailed-vaults/mainnet/0x0B6C8ef0DE1Be5ed1B59E6e7a67fB9442FB9E49C" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Returns full detail for one vault. Same response shape as a single item from the list endpoint.

### Get Vault APY
```
GET /v2/detailed-vaults/{network}/{vaultId}/apy
```

```bash
curl "https://api.vaults.fyi/v2/detailed-vaults/mainnet/0x0B6C8ef0DE1Be5ed1B59E6e7a67fB9442FB9E49C/apy" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Returns APY breakdown only (base, reward, composite) across all timeframes.

### Get Vault TVL
```
GET /v2/detailed-vaults/{network}/{vaultId}/tvl
```

```bash
curl "https://api.vaults.fyi/v2/detailed-vaults/mainnet/0x0B6C8ef0DE1Be5ed1B59E6e7a67fB9442FB9E49C/tvl" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Returns current TVL in USD and native token units.

## Reference Data

### List Networks
```
GET /v2/networks
```
**SDK:** `await sdk.getNetworks()`

```bash
curl "https://api.vaults.fyi/v2/networks" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Returns all supported blockchain networks with name, chain ID, and CAIP-2 identifier. 20+ EVM networks supported.

Networks can be referenced by slug (`mainnet`, `base`, `arbitrum`) or CAIP-2 (`eip155:1`, `eip155:8453`, `eip155:42161`).

### List Assets
```
GET /v2/assets
```

```bash
curl "https://api.vaults.fyi/v2/assets?perPage=10" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Returns supported token metadata: symbol, name, address, decimals, network.

### List Protocols
```
GET /v2/protocols
```

```bash
curl "https://api.vaults.fyi/v2/protocols" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Returns 80+ DeFi protocols with names, products, and descriptions.

### List Curators
```
GET /v2/curators
```

```bash
curl "https://api.vaults.fyi/v2/curators" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Returns 79+ vault curators for filtering.

### List Tags
```
GET /v2/tags
```

```bash
curl "https://api.vaults.fyi/v2/tags" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Returns categorization tags: `top-curator`, `lst`, `rwa`, etc.

## Pagination

All list endpoints use offset pagination:
- `page` (zero-indexed) + `perPage` (default 50)
- Response includes `nextPage` (null when done) and `itemsOnPage`
