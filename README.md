# WDK Agent Skills

Agent skills for non-custodial multi-chain wallet operations using the [Tether Wallet Development Kit (WDK)](https://wallet.tether.io/).

These skills enable AI agents to build and interact with wallets across multiple blockchains, perform token transfers, execute DEX swaps, bridge assets cross-chain, and access DeFi lending protocols.

## Available Skills

| Skill | Capability |
|-------|------------|
| `wallet-btc` | Create and manage Bitcoin wallets, send BTC/USDT via BIP-84, Electrum, and PSBT |
| `wallet-evm` | EVM wallets with EIP-1559 transactions, ERC20 transfers, and ERC-4337 account abstraction |
| `wallet-solana` | Solana wallets with Ed25519 keypairs and SPL token transfers |
| `wallet-spark` | Spark (Lightning) wallets for instant Bitcoin payments, deposits, and withdrawals |
| `wallet-ton` | TON wallets with Jetton support and gasless transactions via paymaster |
| `wallet-tron` | TRON wallets with TRC20 tokens and gasfree transaction support |
| `protocol-swap` | Token swaps via Velora (EVM) and StonFi (TON) DEX protocols |
| `protocol-bridge` | Cross-chain USDT0 bridging via LayerZero omnichain protocol |
| `protocol-lending` | DeFi lending and borrowing through Aave V3 (supply, withdraw, borrow, repay) |
| `protocol-fiat` | Fiat on/off ramps via MoonPay integration |

## Installation

```bash
npx skills add tetherto/wdk-agent-skills
```

## Usage

Skills are automatically available once installed. Example prompts:

- "Create a new Bitcoin wallet and show me the address"
- "Send 10 USDT on Ethereum to 0x..."
- "Swap 50 USDT for ETH on Arbitrum using Velora"
- "Bridge 100 USDT0 from Ethereum to Arbitrum"
- "Supply 500 USDT to Aave on Ethereum"

## Documentation

- **WDK Docs**: [docs.wallet.tether.io](https://docs.wallet.tether.io)
- **WDK Core**: [github.com/tetherto/wdk-core](https://github.com/tetherto/wdk-core)
- **Agent Skills Guide**: [docs.wallet.tether.io/ai/agent-skills](https://docs.wallet.tether.io/ai/agent-skills)

## Contributing

1. Create a new skill directory under `skills/`
2. Add a `SKILL.md` with frontmatter (name, description) and reference files
3. Submit a pull request

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.
