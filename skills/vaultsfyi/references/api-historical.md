# api-historical — Time-Series Data & Benchmarks

## Base URL

```
https://api.vaults.fyi/v2
```

## Historical Vault Data

All historical endpoints require an API key (not available via x402).

### Get Combined History
```
GET /v2/historical/{network}/{vaultId}
```
```bash
curl "https://api.vaults.fyi/v2/historical/mainnet/0x0B6C8ef0DE1Be5ed1B59E6e7a67fB9442FB9E49C?fromTimestamp=1745247600&toTimestamp=1747839600&granularity=1day&perPage=30" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```
Returns combined APY, TVL, and share-price time-series.

### Get Historical APY
```
GET /v2/historical/{network}/{vaultId}/apy
```
```bash
curl "https://api.vaults.fyi/v2/historical/mainnet/0x0B6C8ef0DE1Be5ed1B59E6e7a67fB9442FB9E49C/apy?fromTimestamp=1747234800&toTimestamp=1747839600&granularity=1day" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```
APY time-series with base yield and rewards components.

### Get Historical TVL
```
GET /v2/historical/{network}/{vaultId}/tvl
```
```bash
curl "https://api.vaults.fyi/v2/historical/mainnet/0x0B6C8ef0DE1Be5ed1B59E6e7a67fB9442FB9E49C/tvl?fromTimestamp=1747234800&toTimestamp=1747839600&granularity=1day" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```
TVL snapshots over a configurable time range.

### Get Historical Share Price
```
GET /v2/historical/{network}/{vaultId}/sharePrice
```
```bash
curl "https://api.vaults.fyi/v2/historical/mainnet/0x0B6C8ef0DE1Be5ed1B59E6e7a67fB9442FB9E49C/sharePrice?fromTimestamp=1747234800&toTimestamp=1747839600&granularity=1day" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```
LP token share price evolution.

### Get Historical Asset Prices
```
GET /v2/historical/asset-prices/{network}/{assetAddress}
```
```bash
curl "https://api.vaults.fyi/v2/historical/asset-prices/mainnet/0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48?fromTimestamp=1747234800&toTimestamp=1747839600&granularity=1day" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```
USD price history for a specific token.

**Common parameters for all historical endpoints:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `fromTimestamp` | number | Unix seconds, start of range |
| `toTimestamp` | number | Unix seconds, end of range |
| `granularity` | string | `1hour`, `1day`, or `1week` |
| `page` | number | Zero-indexed |
| `perPage` | number | Results per page |

## Benchmarks

### Get Current Benchmark Rate
```
GET /v2/benchmarks/{network}
```
```bash
curl "https://api.vaults.fyi/v2/benchmarks/mainnet?code=usd" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Returns the current baseline DeFi yield for a network. Two benchmark codes:
- `usd` — USD stablecoin lending/saving benchmark
- `eth` — ETH staking/lending benchmark

Use benchmarks to contextualize vault yields: "This vault's 8.3% APY is 2.1x the USD benchmark rate."

### Get Historical Benchmarks
```
GET /v2/historical-benchmarks/{network}
```
```bash
curl "https://api.vaults.fyi/v2/historical-benchmarks/mainnet?code=usd&fromTimestamp=1745247600&toTimestamp=1747839600&perPage=30" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Benchmark rate time-series. Same pagination parameters as other historical endpoints.

## APY Methodology

For questions about how APY is calculated:
- Trailing windows (1h, 1d, 7d, 30d) based on share price change
- TVL-weighted across vault layers for composite vaults
- Reward APYs include token incentive programs
- Full methodology: https://docs.vaults.fyi/methodology/calculating-apy.md
