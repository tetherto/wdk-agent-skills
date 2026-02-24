# WDK Agent Skills

[Agent skills](https://agentskills.io) for non-custodial multi-chain wallet operations using the [Tether Wallet Development Kit (WDK)](https://wdk.tether.io/).

This repository follows the open [Agent Skills](https://agentskills.io/specification) format, a simple standard for giving AI agents new capabilities and domain expertise.

## Available Skills

| Skill | Description |
|-------|-------------|
| `wdk` | Tether Wallet Development Kit — build and interact with non-custodial multi-chain wallets |

### WDK Capabilities

The `wdk` skill gives agents the knowledge to:

- **Wallet management** — Create and manage wallets on Bitcoin, EVM chains, Solana, Spark, TON, and TRON
- **Token transfers** — Send USDt and other tokens across supported chains
- **DEX swaps** — Execute token swaps via Velora (EVM)
- **Cross-chain bridging** — Bridge USDt0 across chains via LayerZero
- **DeFi lending** — Supply, withdraw, borrow, and repay through Aave V3
- **Fiat on/off ramps** — Buy and sell crypto via MoonPay

## Installation

```bash
npx skills add tetherto/wdk-agent-skills
```

## Usage

Skills are automatically available once installed. Example prompts:

- "Create a new Bitcoin wallet and show me the address"
- "Send 10 USDt on Ethereum to 0x..."
- "Swap 50 USDt for ETH on Arbitrum using Velora"
- "Bridge 100 USDt0 from Ethereum to Arbitrum"
- "Supply 500 USDt to Aave on Ethereum"

## Documentation

- **WDK Agent Skills Guide**: [docs.wdk.tether.io/ai/agent-skills](https://docs.wdk.tether.io/ai/agent-skills)
- **Agent Skills Specification**: [agentskills.io](https://agentskills.io)

## Contributing

1. Create a new skill directory under `skills/`
2. Add a `SKILL.md` following the [Agent Skills specification](https://agentskills.io/specification)
3. Submit a pull request

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.
