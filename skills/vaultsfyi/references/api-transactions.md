# api-transactions — Deposit, Withdraw & Rewards

## Base URL

```
https://api.vaults.fyi/v2
```

## Transaction Flow

Always follow this sequence. Don't skip the context step.

```
1. GET /transactions/context/...    → discover available actions and current state
2. Check currentDepositStep / currentRedeemStep and step arrays
3. GET /transactions/{action}/...   → get unsigned calldata
4. Sign and submit each action in order using currentActionIndex
5. Some vaults require multiple txs (approve → deposit, or request-redeem → wait → claim-redeem)
```

## Endpoints

### Get Transaction Context
```
GET /v2/transactions/context/{userAddress}/{network}/{vaultId}
```
```bash
curl "https://api.vaults.fyi/v2/transactions/context/0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045/mainnet/0x0B6C8ef0DE1Be5ed1B59E6e7a67fB9442FB9E49C" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Returns available actions, user balances, allowances, gas estimates, and claimable rewards. **Always call this before building a transaction.**

**Response includes:**
- `depositSteps` / `redeemSteps` — ordered step arrays
- `currentDepositStep` / `currentRedeemStep` — which step to execute next
- Available actions for this vault/user combination
- User's token balance and existing allowance

### Build Transaction
```
GET /v2/transactions/{action}/{userAddress}/{network}/{vaultId}
```
```bash
curl "https://api.vaults.fyi/v2/transactions/deposit/0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045/mainnet/0x0B6C8ef0DE1Be5ed1B59E6e7a67fB9442FB9E49C?assetAddress=0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48&amount=1000000" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

**Path parameter `action`** — one of:
- `deposit` — deposit tokens into vault
- `redeem` — withdraw tokens from vault
- `request-redeem` — initiate timelock withdrawal
- `claim-redeem` — claim after timelock expires
- `request-deposit` — initiate queued deposit
- `claim-deposit` — claim after queue processes
- `claim-rewards` — claim vault rewards
- `start-redeem-cooldown` — begin cooldown period

**Query parameters:**
- `amount` — token amount for deposit/redeem (in base units)

**Response:**
```json
{
  "actions": [
    {
      "tx": {
        "to": "0x...",
        "data": "0x...",
        "value": "0",
        "chainId": 1
      },
      "description": "Approve USDC spending"
    },
    {
      "tx": {
        "to": "0x...",
        "data": "0x...",
        "value": "0",
        "chainId": 1
      },
      "description": "Deposit USDC into vault"
    }
  ],
  "currentActionIndex": 0
}
```

Sign and submit each `actions[].tx` in order. The `currentActionIndex` indicates which action to execute next.

**With WDK wallet:**
```javascript
for (const action of txResponse.actions) {
  await evmAccount.sendTransaction({
    to: action.tx.to,
    value: BigInt(action.tx.value || '0'),
    data: action.tx.data
  })
}
```

### Get Rewards Context
```
GET /v2/transactions/rewards/context/{userAddress}
```
```bash
curl "https://api.vaults.fyi/v2/transactions/rewards/context/0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Returns claimable rewards across all networks and vaults for a wallet.

### Claim All Rewards
```
GET /v2/transactions/rewards/claim/{userAddress}
```
```bash
curl "https://api.vaults.fyi/v2/transactions/rewards/claim/0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Returns batched claim transaction calldata for all outstanding rewards (Merkl rewards supported).

### Get Transaction Suffix (Attribution)
```
GET /v2/transactions/suffix/{userAddress}/{vaultId}
```
```bash
curl "https://api.vaults.fyi/v2/transactions/suffix/0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045/0x0B6C8ef0DE1Be5ed1B59E6e7a67fB9442FB9E49C" \
  -H "x-api-key: $VAULTS_FYI_API_KEY"
```

Returns a hex suffix for integrators to append to calldata for TVL attribution tracking.

## Important Notes

- Transactions are fully non-custodial. Users sign directly with protocol contracts. No intermediary contracts.
- Some vaults require multiple sequential transactions. Always check the `actions` array length.
- The `amount` parameter uses base units (e.g., `1000000` for 1 USDC with 6 decimals).
- Transaction endpoints work with any API key (no PRO tier required).

## Security

- **Always present transaction details to the user before signing.** Show: vault name, protocol, network, action, amount, and estimated gas.
- **Never auto-execute deposits or withdrawals.** The agent must obtain explicit user confirmation.
- **Verify the context endpoint first.** It confirms the vault supports the intended action and the user has sufficient balance.
