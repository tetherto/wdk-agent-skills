---
name: vaultsfyi
description: vaults.fyi yield discovery, portfolio tracking, and non-custodial transaction building for DeFi vaults. Use when searching for the best yield, comparing vault APYs across protocols and chains, checking portfolio positions, finding idle assets, or building deposit/withdraw transactions. Covers 1000+ vaults across 80+ protocols and 20+ EVM networks. Works standalone or alongside WDK wallet skills for end-to-end yield management.
license: Apache-2.0
compatibility: Requires @vaultsfyi/sdk package and network access to api.vaults.fyi. Authentication via API key or x402 pay-per-request (USDC on Base).
metadata:
  author: vaultsfyi
  version: "1.0"
---

# vaults.fyi

Yield intelligence API for DeFi. Discover vaults, compare APYs, track positions, and build non-custodial deposit/withdraw transactions across 80+ protocols and 20+ EVM networks.

## Two Integration Paths

Choose based on your agent's capabilities:

| Agent type | Integration | When to use |
|------------|-------------|-------------|
| **MCP-native** (Claude Code, WDK MCP Toolkit, any MCP client) | Connect to the vaults.fyi MCP server | Agent supports Model Context Protocol natively |
| **File-based** (Cursor, Windsurf, Claude project knowledge) | Use the TypeScript SDK patterns in this skill | Agent loads SKILL.md as instructions and generates code |

### MCP Server (Preferred for MCP-native agents)

38 tools covering discovery, portfolio, transactions, and historical data. Connect directly:

```
Endpoint: https://mcp.vaults.fyi/mcp
Transport: Remote HTTP (Streamable HTTP)
Auth: Authorization: Bearer <VAULTS_API_KEY>
```

**Claude Code setup:**
```bash
claude mcp add --transport http vaults-fyi https://mcp.vaults.fyi/mcp \
  --header "Authorization: Bearer YOUR_VAULTS_API_KEY"
```

**WDK MCP Toolkit:** Register the vaults.fyi MCP server alongside WDK tools. The agent gets yield discovery and wallet operations in a single tool namespace.

**x402 (keyless):** Agents with a USDC balance on Base can access the MCP server without an API key via the x402 payment protocol.

MCP tool reference: https://docs.vaults.fyi/ai-agents/mcp-server

#### MCP Tool Summary (38 tools)

| Surface | Tools |
|---------|-------|
| Discovery & filtering | `vaults_search`, `vaults_list`, `networks`, `protocols`, `assets`, `curators`, `tags` |
| Vault detail & risk | `vault_details`, `vault_apy`, `vault_tvl` |
| Historical analysis | `vault_apy_history`, `vault_tvl_history`, `vault_returns`, `vault_share_price_history`, `benchmark_apy`, `benchmark_apy_history`, `asset_price_history`, `vault_history` |
| Recommendations | `best_vault`, `best_deposit_options` |
| Positions & portfolio | `positions`, `position_details`, `wallet_balances`, `user_events`, `rewards`, `claim_all_rewards` |
| Transactions & tokens | `build_vault_tx`, `build_claim_rewards`, `transaction_context`, `approve_erc20`, `transfer_erc20`, `transfer_native`, `wrap_native`, `unwrap_native`, `token_balance`, `submit_tx_hash`, `get_transaction_status` |

### TypeScript SDK (For file-based agents)

For agents that generate code from instructions rather than calling tools natively. Use the `@vaultsfyi/sdk` package for type-safe access to the full API surface. The code examples in this skill use the SDK.

### Raw REST API Reference

For direct endpoint usage (e.g., from other programming languages, `curl`, or when not using the SDK), the reference files document the raw REST API with endpoint paths, parameters, and response shapes.

| File | Content |
|------|---------|
| `references/api-discovery.md` | Vault search, filtering, network/protocol/asset lists |
| `references/api-portfolio.md` | Positions, idle assets, best deposit options, returns, events |
| `references/api-transactions.md` | Transaction context, deposit/withdraw calldata, rewards claiming |
| `references/api-historical.md` | APY/TVL/share-price time-series, benchmarks, asset prices |
| `references/api-realtime.md` | Near real-time onchain reads (share price, total assets, supply) |
| `references/integration-patterns.md` | WDK integration recipes, x402 payment setup, common workflows |

When a task involves a specific API surface, read the relevant reference file(s) before writing code.

## Documentation

**API Docs**: https://docs.vaults.fyi
**LLM Reference**: https://docs.vaults.fyi/llms-full.txt
**OpenAPI Spec (v2)**: https://api.vaults.fyi/v2/documentation/json
**MCP Server Docs**: https://docs.vaults.fyi/ai-agents/mcp-server
**Developer Portal**: https://portal.vaults.fyi


## Authentication

Two methods, both fully supported:

### API Key (Standard)
```typescript
import { VaultsSdk } from '@vaultsfyi/sdk'
const sdk = new VaultsSdk({ apiKey: process.env.VAULTS_FYI_API_KEY })
```
Obtain an API key at https://portal.vaults.fyi. Rate limits vary by plan.

### x402 Pay-Per-Request
No API key needed. Pay per request in USDC on Base. Ideal for AI agents and WDK wallets that already hold stablecoins. Supported via `new VaultsSdk({ client: signer })`.

See [integration patterns](references/integration-patterns.md) for x402 setup with WDK wallets.


## Core Capabilities

### 1. Yield Discovery
Find the best vaults across all protocols and chains.

```typescript
import { VaultsSdk } from '@vaultsfyi/sdk'
const sdk = new VaultsSdk({ apiKey: process.env.VAULTS_FYI_API_KEY })

// Search for top USDC vaults by 7-day APY
const result = await sdk.getAllVaults({
  query: {
    allowedAssets: ['USDC'],
    sortBy: 'apy7day',
    sortOrder: 'desc',
    perPage: 10,
  },
})
// result.data[0].apy['7day'].total = 0.0832 means 8.32% APY
```

### 2. Portfolio Intelligence
See what a wallet holds, what's earning yield, and what's sitting idle.

```typescript
// Get all positions for a wallet
const positions = await sdk.getPositions({
  path: { userAddress: walletAddress },
})

// Find idle assets not earning yield
const idle = await sdk.getIdleAssets({
  path: { userAddress: walletAddress },
  query: { allowedNetworks: ['mainnet', 'base', 'arbitrum', 'optimism'] },
})

// Get the single best deposit option for this wallet
const best = await sdk.getBestVault({
  path: { userAddress: walletAddress },
})
```

### 3. Transaction Building
Build non-custodial deposit/withdraw transactions. Users sign directly with protocol contracts.

```typescript
// Step 1: Get transaction context (always do this first)
const context = await sdk.getTransactionsContext({
  path: { userAddress: walletAddress, network, vaultId },
})

// Step 2: Build the deposit transaction
const actions = await sdk.getActions({
  path: { action: 'deposit', userAddress: walletAddress, network, vaultId },
  query: { assetAddress, amount: '1000000' }, // 1 USDC
})
// actions.actions[] contains tx.to, tx.data, tx.value, tx.chainId
// Sign and submit each action in order using currentActionIndex
```

### 4. Benchmark Comparison
Compare vault yields against the market baseline.

```typescript
const benchmark = await sdk.getBenchmarks({
  path: { network: 'mainnet' },
  query: { code: 'usd' },
})
// Returns USD benchmark rate for the network
```


## Critical Notes

### APY values are raw decimals
All APY fields return as raw decimals, not percentages. `0.0543` means 5.43%. Multiply by 100 for display.

### Default filters silently exclude data
- `allowedNetworks` defaults to `["base", "mainnet", "arbitrum", "optimism"]`. Pass explicitly to include other networks.
- `minTvl` defaults to 100,000 USD. Set `minTvl=0` to include smaller vaults.
- `sortOrder` defaults to ascending. Use `sortOrder=desc` for highest-APY-first.

### Transaction flow is multi-step
Always call the context endpoint before building transactions. Some vaults require multiple sequential transactions (approve, then deposit). Check `currentDepositStep` / `depositSteps` in the context response.

### vaultId vs address
Use the `vaultId` field from API responses for subsequent API calls, not the contract `address`. They can diverge for some vaults.


## WDK Integration

vaults.fyi works alongside WDK wallet modules. The typical flow:

1. **Discover**: Use vaults.fyi to find the best yield for the user's assets
2. **Decide**: Present options with APY, TVL, reputation score, and risk data
3. **Execute**: Build transactions via vaults.fyi, sign with a WDK wallet account

For deposit/withdraw transactions, vaults.fyi returns standard EVM calldata that any WDK EVM wallet can sign and submit via `sendTransaction({ to, value, data })`.

See [integration patterns](references/integration-patterns.md) for complete WDK + vaults.fyi recipes.


## Security

### Write Operations Require Human Confirmation
Transaction endpoints return unsigned calldata. The agent MUST present transaction details to the user and obtain explicit confirmation before signing with any wallet.

### Pre-Transaction Checklist
Before executing any deposit or withdraw:
- [ ] User explicitly requested the action
- [ ] Vault details (protocol, network, asset, APY) were presented and confirmed
- [ ] Transaction context was fetched to verify available actions
- [ ] Amount is explicitly specified and reasonable
- [ ] Gas estimates were reviewed

### Read Operations Are Safe
All discovery, portfolio, and historical endpoints are read-only. They query indexed data and never interact with user funds.
