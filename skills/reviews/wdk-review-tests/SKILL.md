---
name: wdk-review-tests
description: Review WDK test suites for compliance with established testing conventions (assertions, mock boundaries, isolation, coverage, naming, structure).
user-invocable: true
argument-hint: "[file or directory path to test files]"
---

# WDK Test Suite Review

Review the test files at the provided path (or the current project's test directory if none given). Apply all rules below.

For each violation found, report:
1. **Rule violated** (e.g., R1, R2, etc.)
2. **File and line**
3. **The offending code**
4. **Suggested fix** with corrected code

Group findings by file. After all files are reviewed, provide a summary with total violation counts per rule.

---

# WDK Testing Standards

You are reviewing test files for a WDK (Wallet Development Kit) module. Apply the rules below to identify violations and suggest fixes.

The rules are repository-agnostic. No WDK module is a "golden reference" — apply every rule to every module equally. If existing code conflicts with a rule, the code is the candidate for change, not the rule.

---

## Rules

### R1. Assert concrete values, never properties or types

Every assertion must compare against a known, pre-computed concrete value. Flag any assertion that checks a type, pattern, range, or mere existence.

**Violation:**
```javascript
expect(typeof result).toBe('string')
expect(fee).toBeGreaterThan(0)
expect(result).toBeDefined()
expect(result).not.toBe(null)
expect(result).toMatch(/^0x[a-f0-9]+$/)
```

**Correct:**
```javascript
expect(result).toBe(EXPECTED_HASH)
expect(fee).toBe(202_000n)
```

If the test mocks all external dependencies, the return value is deterministic — compute and hardcode it.

**Exception:** `toBeUndefined()` is acceptable — it asserts a concrete value (`undefined`) and is functionally equivalent to `toBe(undefined)`. Do **not** flag it.

---

### R2. Verify mock calls with exact arguments

Every mocked dependency must be asserted with `toHaveBeenCalledWith` using exact expected arguments. `toHaveBeenCalled` (without arguments) is insufficient. Limit `expect.any()`, `expect.objectContaining()`, and `expect.arrayContaining()` to cases where the concrete value genuinely cannot be determined.

**Violation:**
```javascript
expect(mockClient.getBalance).toHaveBeenCalled()
```

**Correct:**
```javascript
expect(mockClient.getBalance).toHaveBeenCalledWith(EXPECTED_ADDRESS)
```

**Exception:** `toHaveBeenCalled()` without argument verification is acceptable when the method takes no arguments — `toHaveBeenCalledWith()` adds nothing there.
```javascript
expect(mockClient.disconnect).toHaveBeenCalled()   // disconnect() takes no args — OK
```

---

### R3. One unit per unit test

Each unit test must exercise exactly one method of the system under test. A test that calls `sign()` and then passes the result to `verify()` is an integration test. Hardcode the expected signature instead.

**Violation:**
```javascript
test('should sign and verify', () => {
  const sig = account.sign(MESSAGE)
  expect(account.verify(MESSAGE, sig)).toBe(true)   // exercises two units
})
```

**Correct:**
```javascript
test('should sign a message', () => {
  expect(account.sign(MESSAGE)).toBe(EXPECTED_SIGNATURE)
})
```

---

### R4. Only test the public API

Don't write tests for internal/private modules. Don't access private properties (prefixed with `_`). Internal components are tested indirectly through the public API.

**Violation:**
```javascript
account._tronWeb = mockTronWeb
expect(account._account.privateKey).toBe(null)
```

**Correct:** use `jest.unstable_mockModule` to inject mocks via the module system instead of assigning private properties.

If a test file targets an internal utility (e.g., `src/memory-safe/`, `src/multicall.js`), flag it for removal.

---

### R5. Assert specific error messages

When testing that a method throws, always include the expected message. Bare `.toThrow()` doesn't verify the right error was thrown.

**Violation:**
```javascript
await expect(account.send(opts)).rejects.toThrow()
```

**Correct:**
```javascript
await expect(account.send(opts)).rejects.toThrow('Insufficient balance')
await expect(account.send(opts)).rejects.toThrow(/Insufficient/)   // regex for long messages
```

---

### R6. Assert all relevant return fields

When a method returns an object (e.g., `{ hash, fee, status, to, value }`), assert every relevant field — not just `status`.

**Violation:**
```javascript
const result = await account.sendTransaction(opts)
expect(result.status).toBe('success')   // missing hash, fee
```

**Correct:**
```javascript
const { hash, fee, status } = await account.sendTransaction(opts)
expect(hash).toBe(EXPECTED_HASH)
expect(fee).toBe(EXPECTED_FEE)
expect(status).toBe('success')
```

---

### R7. Use `toBe` for primitives, `toEqual` only for objects/arrays

For primitives (strings, numbers, bigints, booleans), use `toBe` (strict equality). Reserve `toEqual` for deep comparisons of objects and arrays.

---

### R8. Do not call system-under-test methods in hooks

Test hooks (`beforeAll`, `beforeEach`) must not call methods from the class being tested to set up state. Use hardcoded expected values or external tools (`bitcoin-cli`, hardhat RPC, ethers provider).

**Violation:**
```javascript
beforeEach(async () => {
  address = await account.getAddress()   // calling the SUT
})
```

**Correct:**
```javascript
const ADDRESS = '0x1234...'   // hardcoded known value
```

---

### R9. Only mock external dependencies

Only mock methods that interact with external services (network, I/O, blockchain nodes). Pure functions, static utility methods, and local computations must use their real implementations.

**Violation:** mocking `TronWeb.address.toHex`, `decodeSparkAddress`, or `ethers.parseEther` — these are pure functions.

**Correct:** mock `provider.getBalance`, `client.sendTransaction`, `fetch` — these interact with external systems.

When using `jest.unstable_mockModule`, spread the real module exports and override only what needs mocking:
```javascript
import * as realModule from '#libs/spark-sdk'
jest.unstable_mockModule('#libs/spark-sdk', () => ({
  ...realModule,
  SparkClient: { create: jest.fn().mockReturnValue(mockClient) }
}))
```

---

### R10. New methods must have tests

Every new public method added in a PR must have corresponding unit tests. Flag any new method in source code that lacks coverage.

---

### R11. Cover parameter variants and edge cases

When a method accepts optional parameters (`limit`, `offset`, `direction`, `confirmationTarget`, etc.), there must be test cases covering those variants. Edge cases like null returns and boundary values must also be covered.

---

## Test Data Conventions

### Naming
- **Stub return values** (data returned by mocks): `DUMMY_` prefix + `SCREAMING_CASE` — `DUMMY_BALANCE`, `DUMMY_TX_HASH`, `DUMMY_BALANCE_MAP`.
- **Input parameters / options** (data passed to the method under test): no prefix, `SCREAMING_CASE` — `OPTIONS`, `TRANSACTION`.
- **Placeholder strings in values**: `dummy-` prefix (not `mock-`) — `'https://dummy-provider.url/'`.

### Realism
Test fixtures must use realistic values that match actual SDK types. Property names must match the real type definitions.

Violation: using `{ address: '0x...' }` when the real type defines `{ depositAddress: '0x...' }`.

### Seed phrases
Use realistic BIP-39 seed phrases, not trivially repeated words:
```javascript
// BAD
const SEED = 'abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about'
// GOOD
const SEED = 'cook voyage document eight skate token alien guide drink uncle term abuse'
```

---

## Test Structure Conventions

### Lifecycle hooks
- `beforeAll` — expensive shared setup only (server startup, heavy initialization).
- `beforeEach` — per-test state (account creation, blockchain state reset, mock configuration).
- `afterEach` — cleanup (`dispose`, `clearMocks` if needed).
- `afterAll` — expensive shared teardown (server shutdown).

Derive accounts at the beginning of each test when possible, not in shared hooks.

### Mock setup pattern
Define mock function references at the top level. Configure return values inside individual tests:
```javascript
const getBalanceMock = jest.fn()

jest.unstable_mockModule('tronweb', () => ({
  default: jest.fn(() => ({ trx: { getBalance: getBalanceMock } }))
}))

test('should return balance', async () => {
  getBalanceMock.mockResolvedValue(DUMMY_BALANCE)
  // ...
})
```

### Test naming
- Describe behavior, not implementation: `'should close and clean up connections'`, not `'should call wallet.cleanupConnections'`.
- Top-level `describe` uses the module name: `describe('@wdk/wallet-evm', () => { ... })`.
- Keep names concise.

### Directory structure
- Test directory: `tests/` (not `test/`).
- Unit tests: `tests/wallet-account-*.test.js`, `tests/wallet-manager-*.test.js`.
- Integration tests: `tests/integration/module.test.js`.
- Integration helpers: co-located in `tests/integration/helpers.js`.

### Constants placement
- Constants used in multiple tests within a `describe`: declare at the `describe` scope.
- Constants used in only one test: declare in the test body.
- Don't put all constants at the top of the file if they're only used in specific suites.

---

## Integration Test Conventions

### Environment
- Use hardhat to fork a testnet at a known block number — never run against a live testnet.
- Minimize required environment variables — use internally set values where possible.
- Integration tests must be atomic: one scenario per test.

### Structure
- Each integration test implements a complete scenario in its own `test()` block.
- Use `beforeEach`/`afterEach` for per-test setup/teardown, not `beforeAll` for per-test state.
- Reset blockchain state between tests with `hardhat_reset` or equivalent.
- Don't split a single scenario into multiple sequential tests that share state.

### Verification
- Use the local test node or ethers provider to independently verify transaction effects (Transfer events, balance changes, block inclusion).
- Assert all fields: `hash`, `fee`, `value`, `to`, `status`.
- Use exact fee assertions — fetch the actual fee from the chain and compare.
- For balance checks: fetch the real balance from the provider; don't rely on accumulated state.

---

## Review Workflow

1. **Assertions** — flag `typeof`, `toBeDefined`, `toBeGreaterThan`, `toMatch`, `expect.any()`, or missing `toHaveBeenCalledWith` arguments (R1, R2). `toBeUndefined()` is acceptable.
2. **Mock boundaries** — only external-facing methods mocked; pure functions use real implementations (R9).
3. **Test isolation** — each test exercises one method (R3); hooks don't call the SUT (R8); state reset per-test.
4. **Public API only** — no tests of internals; no access to `_private` properties (R4).
5. **Errors** — `.rejects.toThrow()` always includes a message or regex (R5).
6. **Return-value coverage** — every relevant field on returned objects is asserted (R6).
7. **Equality** — `toBe` for primitives, `toEqual` only for deep objects/arrays (R7).
8. **Coverage** — every new public method has tests (R10); parameter variants and edge cases covered (R11).
9. **Test data** — `DUMMY_`/`SCREAMING_CASE` naming, `dummy-` (not `mock-`) for placeholder strings, realistic SDK-shaped fixtures, real BIP-39 seed phrases.
10. **Structure** — `tests/` directory, lifecycle hooks used correctly, constants scoped to where they're used, top-level mock refs configured per-test.
11. **Integration** — hardhat fork at known block, one scenario per test, independent on-chain verification, all fields asserted.
