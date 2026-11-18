---
name: wdk-review-module
description: Pre-review a WDK wallet or protocol module against the contribution guide, checking every rule and checklist item and reporting pass, fail, not run, or n/a. Use before requesting a WDK source review.
user-invocable: true
argument-hint: "[module root directory]"
---

# WDK Module Pre-Review

Review the module at the provided path (or the current project if none given) against the [contribution guide](https://docs.wdk.tether.io/tools/create-wdk-module/). Read the guide first. If this skill and the guide disagree, the guide wins.

Give every item one result, with its evidence (the command and outcome, or the file and line):

- **pass**: you ran the command or read the code, and it holds.
- **fail**: it does not hold. Give the smallest fix.
- **not run**: you could not check it. Say what is missing.
- **n/a**: it does not apply to this module.

A claim in a README is not evidence that the code does it. Do not use real funds, real keys, or paid APIs unless the user says so. End with the count of each result, then the list of failures.

## Rules

### All modules

- **M1. Run on Bare.** `exports["."].bare` points to an entry that exists and loads `bare-node-runtime`.
- **M2. Resolve one copy of `@tetherto/wdk-wallet`.** `package.json` declares a version range, not an exact version.
- **M3. Keep failures distinct.** Invalid input, unsupported operations, missing configuration, and provider failures throw. They never become an empty or successful result, whether by `catch`, early return, or empty stub. WDK errors are reused where one fits, custom errors extend `WdkError`, and each thrown error has `@throws`.
- **M4. Never lose a transaction.** After a submission, every result and error path keeps the transaction identifier and partial-execution details. A failed submission is distinct from an unknown broadcast outcome.
- **M5. Add, don't redefine.** The module extends the base class for its type in the latest `@tetherto/wdk-wallet` release and implements every required method. Each override keeps the meaning and return shape of its method there.
- **M6. Export what callers need.** `wdk-review-dts` TD3 and TD4 hold. Thrown errors meet TD5, or their import path is documented.
- **M7. Leave secrets and logs to the host.** `grep -rnE "console\.[a-z]+\(" src index.js bare.js` finds nothing. Provider requests and telemetry are disclosed in the README.

### Protocol modules

- **M8. Sign with the host's account.** Signing and submission go only through the account given to the constructor. There is no seed or private key input, and no second signer. Chain module types are used where they specialize the base ones.
- **M9. Report status honestly.** An unrecognized status in a valid response maps to `pending` for swidge and SDA, or `in_progress` for fiat. Invalid identifiers, missing records, and failed requests throw.
- **M10. Honor fee caps.** Exceeding a fee cap the interface defines, such as `swapMaxFee`, throws `MaximumFeeExceededError`.

### Wallet modules

- **M11. Override both account factories.** A variant that changes the account class overrides `getAccount()` and `getAccountByPath()`.
- **M12. Clear their own state.** Extra keys or sessions are cleared in `dispose()`, after `super.dispose()`.

## Checklist

### Package

- **M13. Name.** `name` matches what the CLI generates: `wdk-wallet-<chain>[-<variant>]`, `wdk-protocol-<swap|bridge|lending>-<name>-<chain>`, or `wdk-protocol-<fiat|swidge|sda>-<name>`. `npm view <name> maintainers` returns 404, or list the maintainers for the user to confirm.
- **M14. Lint and types.** `npm run lint` passes, and `types/` matches the code (`wdk-review-jsdocs` R23). Do not run `build:types` (TD1).
- **M15. Dependencies.** Every entry in `dependencies` is used by `src/`, `types/`, `index.js`, or `bare.js`. Nothing they import is only in `devDependencies`.
- **M16. Packed install.** Pass when the install succeeds and `npm ls` shows one version:

  ```bash
  DIR="$(mktemp -d)"
  TARBALL="$DIR/$(npm pack --silent --pack-destination "$DIR")"
  (cd "$DIR" && npm init -y >/dev/null && npm install "$TARBALL" @tetherto/wdk && npm ls @tetherto/wdk-wallet)
  ```

### Behavior

- **M17. Tests.** `npm test` passes, `grep -rnE "(test|it)\.todo" tests` finds nothing, and every `@throws` in `src/` has a test.
- **M18. Main flow.** A test runs the main flow through `@tetherto/wdk` and a real wallet package, with no mocks or sender stubs, on a fork at a known block, or live if the provider can't run on a fork. Give the block height, or the network and transaction hash, for the review request.
- **M19. Bare runtime.** The same flow runs from the packed Bare entry with `bare <file>`, not through `npm test`, or the README documents an approved waiver. A test that only imports the entry does not count.
- **M20. Disposal.** If the module holds extra state, tests dispose through both the account and the manager and assert the state is gone.
- **M21. Companion skills.** Run `wdk-review-jsdocs`, `wdk-review-dts`, and `wdk-review-tests`. Report their violation counts.

### Documentation

- **M22. README.** It has install steps, a runnable example, configuration, supported chains and operations, errors with caller actions, fees, known limitations, supported WDK versions, migration steps for breaking changes, and support and security contacts. List what is missing.
- **M23. Caveats.** Bundler, Bare, credential, and real-funds caveats sit next to the run commands, not only in example comments.
- **M24. Credentials.** The repository holds no real credentials. `.env.example` lists every `process.env` variable that tests and examples read.
- **M25. Example assets.** Examples use Tether assets or neutral placeholders.
