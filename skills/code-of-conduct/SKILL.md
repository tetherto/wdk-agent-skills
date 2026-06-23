---
name: code-of-conduct
description: Code of conduct that contributors — including AI coding assistants — MUST follow when writing, modifying, or reviewing code for any Tether Wallet Development Kit (WDK) module (wallets, protocols, core). Covers project workflow (atomic single-feature pull requests, design-before-implementation specs), code quality (avoid code smells, never violate S.O.L.I.D. principles, document every public/protected component, use narrow types, avoid defensive programming, mark every thrown error with @throws), and testing (arrange-act-assert, test only the public API, assert real values not patterns, define jest mocks at the top level, mock all external-service dependencies).
---

# WDK Code of Conduct

Binding rules every contributor — human or AI coding assistant — MUST follow when writing, modifying, or reviewing code for any Wallet Development Kit (WDK) module (wallets, protocols, core).

- **When writing or changing WDK code**, comply with every applicable rule before you consider the work done.
- **When reviewing WDK code**, flag each violation, cite its rule ID, and propose a concrete fix.

The rules are repository-agnostic. No WDK module is a "golden reference" — apply every rule to every module equally. If existing code conflicts with a rule, the code is the candidate for change, not the rule.

Rules are grouped by prefix: **P** = project & workflow, **C** = code, **T** = testing.

| ID | Rule |
|----|------|
| [P001](#p001-pull-requests-should-only-include-single-features) | Pull requests should only include single features |
| [P002](#p002-design-then-implement) | Design, then implement |
| [C001](#c001-avoid-code-smells) | Avoid code smells |
| [C002](#c002-code-should-never-violate-solid-principles) | Code should never violate S.O.L.I.D. principles |
| [C003](#c003-provide-proper-documentation-for-all-public-and-protected-components) | Provide proper documentation for all public and protected components |
| [C004](#c004-use-narrow-types-for-properties-and-methods) | Use narrow types for properties and methods |
| [C005](#c005-avoid-defensive-programming) | Avoid defensive programming |
| [C006](#c006-always-mark-errors-that-a-method-can-throw) | Always mark errors that a method can throw |
| [T001](#t001-follow-the-arrange-act-assert-pattern) | Follow the arrange, act, assert pattern |
| [T002](#t002-test-only-components-that-are-part-of-the-public-api) | Test only components that are part of the public API |
| [T003](#t003-always-test-against-real-values-not-properties) | Always test against real values, not properties |
| [T004](#t004-define-and-assign-jest-functions-at-the-top-level-stub-them-in-test-cases) | Define and assign jest functions at the top level, stub them in test cases |
| [T005](#t005-mock-all-depended-on-components-that-interact-with-external-services) | Mock all depended-on components that interact with external services |

> **Related skills:** `wdk-review-types-jsdoc` enforces the JSDoc & type conventions behind C003/C004 in depth; `wdk-review-tests` enforces the T-rules in depth. Reach for them when you want a focused review of an existing file.

---

## Project guidelines

### P001: Pull requests should only include single features

Keep every pull request to the **minimum code needed to implement one atomic feature** — a single piece of functionality — so reviews stay fast. For a public-API class, each public method is a separate atomic feature.

You may bundle several *minor* features in one PR at your discretion to cut down the number of PRs, but only when doing so keeps the PR small and quick to go over. Large, multi-feature PRs are not acceptable.

---

### P002: Design, then implement

Before implementing any change to the **public API** or any new module from scratch, **design first**: write a technical specification document and reach consensus on it before writing a line of implementation.

The specification must:

- describe **only** public-API components — stay fully agnostic of implementation details;
- include a **technical reference** for every new public component (e.g. for a method: its full documentation, including types and descriptions);
- optionally capture the rationale and critical thinking behind the new change or module.

Reviewers and technical leaders review the document, discussing and requesting changes. **Implementation starts only after the specification reaches common consensus.**

#### Example

A document template is a work in progress. In the meantime, use `wdk-protocol-swap-aori-evm`'s [specification](https://docs.google.com/document/d/1jmMPFD1OTHVZ3yYuGsoSsGfXXeBdyRYHENhvE7R_BYA/edit?tab=t.0#heading=h.uvhtjdaqr4re) as a reference.

---

## Code guidelines

### C001: Avoid code smells

Write cleaner, more readable, more maintainable code by avoiding every [code smell](https://refactoring.guru/refactoring/smells). If you are unsure what counts as a smell, consult refactoring.guru's list or the [code smell catalog](https://luzkan.github.io/smells/). Common offenders: magic numbers, duplicated code, long methods, large classes, and feature envy.

#### Example

Magic numbers are usually a code smell — extract them into named constants.

```javascript
export default class WalletManagerEvm extends WalletManager {
    async getFeeRates () {
        [...]

        // Bad: magic numbers are usually a code smell.
        return { 
            normal: feeRate * 110n / 100n,
            [...]
        }

        // Good:
        return {
            normal: feeRate * WalletManagerEvm._FEE_RATE_NORMAL_MULTIPLIER / 100n,
            [...]
        }
  }
}
```

---

### C002: Code should never violate S.O.L.I.D. principles

Know all five **S.O.L.I.D.** principles and make sure your code violates none of them: [Single Responsibility](https://www.brainstobytes.com/the-single-responsibility-principle/), [Open–Closed](https://www.brainstobytes.com/the-open-closed-principle/), [Liskov Substitution](https://www.brainstobytes.com/the-liskov-substitution-principle/), [Interface Segregation](https://www.brainstobytes.com/interface-segregation-principle/), and [Dependency Injection](https://www.brainstobytes.com/dependency-injection/).

#### Example

Adding a stricter post-condition to an override breaks the Liskov Substitution Principle.

```javascript
export default class WalletAccountReadOnlyEvm extends WalletAccountReadOnly {
    /**
     * Returns the account balance for a specific token.
     *
     * @param {string} tokenAddress - The smart contract address of the token.
     * @returns {Promise<bigint>} The token balance (in base unit).
     * // Bad: the following post-condition breaks the liskov substitution principle, since the super-type of
     * the 'WalletAccountReadOnlyEvm' class doesn't define the 'getTokenBalance' to throw when the account
     * has no funds. For this reason, code that depends on the 'WalletAccountReadOnly' abstract class will
     * not expect the method to throw and will not handle such case. So, this invalid post-condition could
     * lead to unsafe behavior when using instances of the 'WalletAccountReadOnlyEvm' class on code that
     * depends on 'WalletAccountReadOnly'. For more insights, read the reference above.
     * @throws {Error} If the account has no balance for the given token.
     */
    async getTokenBalance (tokenAddress) {
        [...]
    }
}
```

---

### C003: Provide proper documentation for all public and protected components

Every component that is part of the public API needs proper documentation — **meaningful descriptions and types**. `@protected` members count: protected visibility is part of the public API. Documenting private fields and methods is not as essential, but still a significant readability improvement, especially for complex components.

#### Example

A `@protected` property is part of the public API and needs a description and `@type`.

```javascript
export default class WalletAccountEvm extends WalletAccountReadOnlyEvm {
    constructor (seed, path, config = { }) {
        // Bad: the _config property has protected visibility and so it's part of the public api and
        // requires proper documentation.
        /** @protected */
        this._config = config

        // Good:
        /** 
         * The wallet account configuration.
         *
         * @protected
         * @type {EvmWalletConfig}
         */
        this._config = config
    }
}
```

---

### C004: Use narrow types for properties and methods

Always extract and use the **narrowest type that fits** a property or method.

- For a value that can be any type, or whose type is not known, prefer `unknown` over `any`.
- For an object that should hold arbitrary user-chosen data, use `Record<string, unknown>` instead of `Object`.

Wide types leak terrible types to consumers and shift the effort of validating a value from the type checker into your own runtime code.

#### Example

Narrow the `body` field to the types it actually accepts.

```javascript
// Bad: the type of the 'body' field is too wide, which makes it accept more values than it should. Other
// than providing terrible types to external code, this also moves the effort to check that 'body'
// actually contains a valid value from the type checker to our code.
/**
 * @typedef {Object} TonTransaction
 * @property {string} to - The transaction's recipient.
 * @property {number | bigint} value - The amount of tons to send to the recipient (in nanotons).
 * @property {boolean} [bounceable] - If set, overrides the bounceability of the transaction.
 * @property {Object} [body] - Optional message body for smart contract interactions.
 */

// Good:
/**
 * @typedef {Object} TonTransaction
 * @property {string} to - The transaction's recipient.
 * @property {number | bigint} value - The amount of tons to send to the recipient (in nanotons).
 * @property {boolean} [bounceable] - If set, overrides the bounceability of the transaction.
 * @property {string | Cell} [body] - Optional message body for smart contract interactions.
 */
```

---

### C005: Avoid defensive programming

Defensive programming means checking that a method's pre-conditions hold before proceeding. The only pre-conditions WDK uses are the **types of a method's arguments**, and you should always assume those are correct — the generated `.d.ts` declarations together with the consumer's type checker already report type errors. Runtime null/type guards on already-typed parameters are redundant; do not write them.

#### Example

Do not re-validate a parameter the type already guarantees.

```javascript
export default class WalletAccountReadOnlyTon {
    /**
     * Returns the balance of the account for a specific token.
     *
     * @param {string} tokenAddress - The smart contract address of the token.
     * @returns {Promise<bigint>} The token balance (in base unit).
     */
    async getTokenBalance (tokenAddress) {
        // Bad: the contract of the 'getTokenBalance' method defines the 'tokenArgument' as a string.
        // Since we do not implement defensive programming, we should assume that its value is always
        // valid and of the correct type. For this reason, there's no real need to check for nullability
        // or that its type is actually 'string'. 
        if (!tokenAddress || typeof tokenAddress !== 'string') {
            throw new Error('Invalid value for the token address.')
        }

        [...]
    }
}
```

---

### C006: Always mark errors that a method can throw

Document **every** error a method might throw in its documentation, using the `@throws` tag to state both the **error type** and the **condition** under which it is thrown. This lets consumers know which errors to expect and exactly when.

#### Example

```javascript
export default class WalletAccountEvm extends WalletAccountReadOnlyEvm {
    /**
     * Approves a specific amount of tokens to a spender.
     *
     * @param {ApproveOptions} options - The approve options.
     * @returns {Promise<TransactionResult>} The transaction's result.
     * @throws {Error} If trying to approve usdts on ethereum with allowance not equal to zero (due to the usdt allowance reset requirement).
     */
    async approve (options) {
        [...]
    }
}
```

---

## Testing guidelines

### T001: Follow the arrange, act, assert pattern

Structure every unit and integration test as **Arrange → Act → Assert**:

- **Arrange** — set up all constants, mocks, and data the test needs, including any `before`/`after` hook.
- **Act** — run the code under test, e.g. one method call on the [SUT](http://xunitpatterns.com/SUT.html).
- **Assert** — make all the assertions on the result of the act step.

#### Example

```javascript
describe('WalletAccountTron', () => {
    describe('signTransaction', () => {
        test('should sign a transaction', async () => {
            // Arrange:
            const EXPECTED_SIGNATURE = 'e2fbd0590d2a6150952afdcdb8c0b137a8828fe45dacc6f17f552b10234baa9231811488104983c4d4333ad51c90343801aa72e41b1d576719cc798c4c98546100'

            sendTrxMock.mockResolvedValue(DUMMY_SEND_TRX_RESULT)

            // Act:
            const transaction = await account.signTransaction(TRANSACTION)

            // Assert:
            expect(sendTrxMock).toHaveBeenCalledWith(TRANSACTION.to, TRANSACTION.value, ACCOUNT.address)

            expect(transaction).toEqual({
                ...DUMMY_SEND_TRX_RESULT,
                signature: [EXPECTED_SIGNATURE]
            })
        })
    })
})
```

---

### T002: Test only components that are part of the public API

Unit and integration tests must cover and use **only public-API components**. This hides implementation details, keeps tests passing when internals change, and forces you to test against the specification from the user's point of view. Do not test private members (prefixed with `_`) directly — exercising the public API reaches protected and private code indirectly, so coverage still flows through.

#### Example

Do not write a test suite for a private method.

```javascript
describe('WalletAccountTron', () => {
    // Bad: the '_signTransaction' method has private visibility, so it's not part of the public
    // api and tests should not cover it. Note that by testing methods that are part the public
    // api, code coverage will indirectly reach protected and private components as well.
    describe('_signTransaction', () => {
        // [...]
    })
})
```

---

### T003: Always test against real values, not properties

Always assert and compare outputs against **concrete, real expected values**, instead of properties or patterns that merely *should* apply to them. A pattern/shape check passes for any value of the right form — even a wrong one — so it does not prove correctness.

#### Example

```javascript
describe('WalletAccountTron', () => {
    describe('sign', () => {
        test('should return the correct signature', async () => {
            // Bad: the following assertion checks that the output of the 'sign' method is a valid
            // signature, but not that it is correct. If the 'sign' method returns a string that
            // matches the pattern above, the test will pass regardless of whether the signature
            // actually encodes the given message.
            const SIGNATURE_PATTERN = /^(?:0x)?[0-9a-fA-F]{128}(?:00|01|1b|1c|1B|1C)$/

            const signature = await account.sign('Hello world!')

            expect(signature).toMatch(SIGNATURE_PATTERN)

            // Good:
            const EXPECTED_SIGNATURE = '0x0e6d4a25bc8da8fcc08a227612d1caa5f33635f2c9b490f26ea08e228ec879a670607c5bfb68f45fbb1e67972420a366e66aadf18c9dc7806201ea202354c6fc1b'

            const signature = await account.sign('Hello world!')

            expect(signature).toBe(EXPECTED_SIGNATURE)
        })
    })
})
```

---

### T004: Define and assign jest functions at the top level, stub them in test cases

When a unit test must mock one of the SUT's [depended-on components](http://xunitpatterns.com/DOC.html), **define and assign the jest mock functions at the top level** of the test suite. Because WDK uses ESM, this usually requires jest's [`unstable_mockModule`](https://jestjs.io/docs/ecmascript-modules#module-mocking-in-esm). Configure the **stubs** (return values) only inside `before` hooks or individual test cases. The Assert step must always include assertions that verify each stub was **called with the proper arguments**.

#### Example

```javascript
const sendTrxMock = jest.fn()

jest.unstable_mockModule('tronweb', () => ({
    transactionBuilder: {
        sendTrx: sendTrxMock
    }
}))

describe('WalletAccountTron', () => {
    describe('signTransaction', () => {
        test('should sign a transaction', async () => {
            const EXPECTED_SIGNATURE = 'e2fbd0590d2a6150952afdcdb8c0b137a8828fe45dacc6f17f552b10234baa9231811488104983c4d4333ad51c90343801aa72e41b1d576719cc798c4c98546100'

            sendTrxMock.mockResolvedValue(DUMMY_SEND_TRX_RESULT)

            [...]

            expect(sendTrxMock).toHaveBeenCalledWith(TRANSACTION.to, TRANSACTION.value, ACCOUNT.address)
        })
    })
})
```

---

### T005: Mock all depended-on components that interact with external services

Unit tests must mock **all** the SUT's [depended-on components](http://xunitpatterns.com/DOC.html) that interact with **external services** (network, blockchain nodes, I/O). This keeps unit tests fast, isolated, and deterministic. Integration tests take care of covering the actual integration with the external service.

#### Example

```javascript
const getNetworkMock = jest.fn()

jest.unstable_mockModule('ethers', () => ({
  ...ethers,
  JsonRpcProvider: jest.fn().mockImplementation(() => ({
    getNetwork: getNetworkMock
  }))
}))
```

---

## Compliance checklist

Before considering any WDK code change complete, confirm:

**Project**
- [ ] PR implements a single atomic feature and stays small and quick to review (P001)
- [ ] Public-API or new-module work began from an agreed-upon specification document (P002)

**Code**
- [ ] No code smells — magic numbers, duplication, long methods, large classes (C001)
- [ ] No S.O.L.I.D. violation (C002)
- [ ] Every public and protected component has a meaningful description and types (C003)
- [ ] Narrowest fitting types used; `unknown` over `any`, `Record<string, unknown>` over `Object` (C004)
- [ ] No defensive checks on already-typed parameters (C005)
- [ ] Every throwable method documents each error with `@throws` (type + condition) (C006)

**Testing**
- [ ] Tests follow Arrange → Act → Assert (T001)
- [ ] Only public-API components are tested — no `_private` suites (T002)
- [ ] Assertions compare against real expected values, not patterns or properties (T003)
- [ ] Jest mock functions defined at the top level, stubbed per test, and asserted with exact arguments (T004)
- [ ] All external-service dependencies are mocked in unit tests (T005)
