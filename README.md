# WDK Agent Skills

[Agent skills](https://agentskills.io) for writing and reviewing [Tether Wallet Development Kit (WDK)](https://wdk.tether.io/) modules.

This repository follows the open [Agent Skills](https://agentskills.io/specification) format, a simple standard for giving AI agents new capabilities and domain expertise.

## Available Skills

| Skill | Description |
|-------|-------------|
| `wdk` | Moved to [`tetherto/wdk-docs`](https://github.com/tetherto/wdk-docs/tree/main/skills/wdk) |
| `code-of-conduct` | Code of conduct that contributors and AI coding assistants follow when writing, modifying, or reviewing WDK module code |
| `wdk-review-jsdocs` | Review WDK source files for JSDoc format and type-annotation conventions |
| `wdk-review-dts` | Review and maintain WDK generated `.d.ts` type definitions |
| `wdk-review-tests` | Review WDK test suites for compliance with established testing conventions |
| `wdk-review-module` | Pre-review a wallet or protocol module against the [contribution guide](https://docs.wdk.tether.io/tools/create-wdk-module/) |

## Installation

```bash
npx skills add tetherto/wdk-agent-skills -g \
  --skill wdk-review-module \
  --skill wdk-review-jsdocs \
  --skill wdk-review-dts \
  --skill wdk-review-tests
```

## Usage

Run a review from your module's root. If your agent has no slash commands, ask for the skill by name.

```text
/wdk-review-module .
```

## Documentation

- **Contribution guide**: [docs.wdk.tether.io/tools/create-wdk-module](https://docs.wdk.tether.io/tools/create-wdk-module/)
- **Agent Skills Specification**: [agentskills.io](https://agentskills.io)

## Contributing

1. Create a new skill directory under `skills/`
2. Add a `SKILL.md` following the [Agent Skills specification](https://agentskills.io/specification)
3. Submit a pull request

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.
