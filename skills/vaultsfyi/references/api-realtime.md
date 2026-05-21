# api-realtime — Near Real-Time Onchain Data

## Base URL

```
https://api.vaults.fyi/v2
```

## NRT (Near Real-Time) Endpoints

These endpoints read directly from onchain state rather than hourly-indexed data. Use for deposit previews, live dashboards, or liquidation monitoring where sub-second freshness matters.

### Get All NRT Metrics
```
GET /v2/nrt/vault/{network}/{vaultId}
```
```bash
curl "https://api.vaults.fyi/v2/nrt/vault/mainnet/0x0B6C8ef0DE1Be5ed1B59E6e7a67fB9442FB9E49C" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```
Returns share price, total assets, total supply, and underlying asset price in a single call.

### Get NRT Share Price
```
GET /v2/nrt/vault/{network}/{vaultId}/sharePrice
```
```bash
curl "https://api.vaults.fyi/v2/nrt/vault/mainnet/0x0B6C8ef0DE1Be5ed1B59E6e7a67fB9442FB9E49C/sharePrice" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```
Current LP token value from onchain state.

### Get NRT Total Assets
```
GET /v2/nrt/vault/{network}/{vaultId}/totalAssets
```
```bash
curl "https://api.vaults.fyi/v2/nrt/vault/mainnet/0x0B6C8ef0DE1Be5ed1B59E6e7a67fB9442FB9E49C/totalAssets" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```
Current total assets held by the vault.

### Get NRT Total Supply
```
GET /v2/nrt/vault/{network}/{vaultId}/totalSupply
```
```bash
curl "https://api.vaults.fyi/v2/nrt/vault/mainnet/0x0B6C8ef0DE1Be5ed1B59E6e7a67fB9442FB9E49C/totalSupply" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```
Current total LP token supply.

### Get NRT Underlying Asset Price
```
GET /v2/nrt/vault/{network}/{vaultId}/underlyingAssetPrice
```
```bash
curl "https://api.vaults.fyi/v2/nrt/vault/mainnet/0x0B6C8ef0DE1Be5ed1B59E6e7a67fB9442FB9E49C/underlyingAssetPrice" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```
Current price of the deposit asset.

## When to Use NRT vs Standard Endpoints

| Use Case | Endpoint |
|----------|----------|
| Vault search / comparison | `/v2/detailed-vaults` (indexed, faster, cheaper) |
| Portfolio overview | `/v2/portfolio/positions` (indexed) |
| Deposit preview (exact shares) | `/v2/nrt/vault/.../sharePrice` (live onchain) |
| Liquidation monitoring | `/v2/nrt/vault/...` (live onchain) |
| Historical analysis | `/v2/historical/...` (indexed time-series) |
