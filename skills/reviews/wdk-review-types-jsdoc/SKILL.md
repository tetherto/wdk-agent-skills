---
name: wdk-review-types-jsdoc
description: Review WDK source files for compliance with established Types & JSDoc conventions (JSDoc format, type annotations, import/export rules, .d.ts alignment).
user-invocable: true
argument-hint: "[file or directory path to source files]"
---

# WDK Types & JSDoc Review

Review the source files at the provided path (or the current project's `src/` directory if none given). Apply all rules below.

For each violation found, report:
1. **Rule violated** (e.g., R1, R2, TD1, etc.)
2. **File and line**
3. **The offending code**
4. **Suggested fix** with corrected code

Group findings by file. After all files are reviewed, provide a summary with total violation counts per rule.

---

# WDK Types & JSDoc Standards

You are reviewing source files for a WDK (Wallet Development Kit) module. Apply the rules below to identify JSDoc and type-annotation violations and suggest fixes.

The rules are repository-agnostic. No WDK module is a "golden reference" — apply every rule to every module equally. If existing code conflicts with a rule, the code is the candidate for change, not the rule.

---

## JSDoc Format Rules

### R1. Every JSDoc block must start with a plain-text description before any tags

Tags like `@param`, `@returns`, `@type` must not appear before the description.

**Violation:**
```javascript
/**
 * @param {Provider} provider - An ethers provider.
 * @returns {Promise<string>} The address.
 */
```

**Correct:**
```javascript
/**
 * Returns the wallet address from the given provider.
 *
 * @param {Provider} provider - An ethers provider.
 * @returns {Promise<string>} The address.
 */
```

---

### R2. Use `{Object}` (capitalized) as the kind-marker in `@typedef` declarations, not `{object}`

```javascript
/** @typedef {object} GetBalanceResult */     // wrong — use Object as the kind-marker
/** @typedef {Object} GetBalanceResult */     // correct
```

This applies only to `@typedef` kind-markers. Using `{Object}` as a value type in `@param`, `@property`, `@returns`, or `@type` is forbidden — see R13.

---

### R3. Dash separator required in `@property` and `@param`

Every `@param` and `@property` tag must have a dash between the name and the description.

```javascript
@param {string} address The wallet address.       // wrong
@param {string} address - The wallet address.     // correct
```

---

### R4. No `[param=value]` default syntax — state default in description text

```javascript
@param {number} [timeout=15000] - Connection timeout.                      // wrong
@param {number} [timeout] - Connection timeout (default: 15,000).          // correct
```

---

### R5. `@private` only on class/namespace members, not module-level constants

Module-level constants (even those prefixed with `_`) shouldn't use `@private`.

**Violation:**
```javascript
/**
 * Compiled multicall constructor bytecode.
 * @private
 * @type {string}
 */
const _MULTICALL_BYTECODE = ...
```
Drop `@private`; keep the description and `@type`.

---

### R6. Private members get only `/** @private */` — no full JSDoc

Private methods and properties (prefixed with `_`) get a minimal block. Don't document parameters, return types, descriptions, or `@type` for private members.

**Violation:**
```javascript
/**
 * Returns the fee estimator for the given bundler URL.
 * @private
 * @param {string} bundlerUrl - The bundler URL.
 * @returns {Promise<FeeEstimator>} The fee estimator.
 */
async _getFeeEstimator (bundlerUrl) { ... }
```

**Correct:**
```javascript
/** @private */
async _getFeeEstimator (bundlerUrl) { ... }
```

Same for private properties: `/** @private */` only, nothing else.

---

### R7. `@protected` members need full documentation

Properties need description + `@protected` + `@type`. Methods need a full block: description, `@protected`, `@param` for every parameter, and `@returns`.

**Violation (property — missing description):**
```javascript
/** @protected @type {number} */
this._timeout = timeout
```

**Correct (property):**
```javascript
/**
 * Connection timeout in milliseconds.
 * @protected
 * @type {number}
 */
this._timeout = timeout
```

**Correct (method):**
```javascript
/**
 * Builds the Safe initialization data for the given config.
 *
 * @protected
 * @param {SafeConfig} config - The Safe construction config.
 * @returns {Promise<string>} The encoded initialization data.
 */
async _buildInitData (config) { ... }
```

---

### R8. Always provide full JSDoc on overridden methods

Copy the parent's JSDoc on the override (or rewrite it more specifically). `tsc` needs JSDoc on the override for accurate `.d.ts` generation. Make documentation more blockchain-specific when possible — add `@example`, narrow return types, clarify behavior.

```javascript
/**
 * Returns the wallet account at a specific index (BIP-44).
 *
 * @example
 * const account = await wallet.getAccount(1);   // m/44'/60'/0'/0/1
 * @param {number} [index] - The index of the account (default: 0).
 * @returns {Promise<WalletAccountEvm>} The account.
 */
async getAccount (index) { ... }
```

---

### R9. Use `@implements` for interfaces, `@extends` only for real inheritance

`@implements {IFoo}` for interface conformance; `@extends` only for actual class inheritance. Omit `@extends` when the `extends` keyword is already present — it's redundant.

**Violation:**
```javascript
/** @abstract @extends IElectrumClient */
export default class BaseClient extends IElectrumClient { ... }
```

**Correct:**
```javascript
/** @abstract @implements {IElectrumClient} */
export default class BaseClient { ... }
```

---

## Type Import/Export Rules

### R10. Import types via single-line `@typedef`; import from `index.js` when available

Don't use runtime `import` statements for type-only imports.

**Violation:**
```javascript
// eslint-disable-next-line no-unused-vars
import BaseClient from './transports/client/base-client.js'
```

**Correct:**
```javascript
/** @typedef {import('./transports/index.js').IElectrumClient} IElectrumClient */
```

---

### R11. Use short alias names in JSDoc, not full `import(...)` paths inline

Define a `@typedef` alias at the top of the file; use the short name in `@param`/`@returns`.

**Violation:**
```javascript
/** @param {import('./transports/mempool-electrum-client.js').MempoolElectrumConfig} config */
```

**Correct:**
```javascript
/** @typedef {import('./transports/index.js').MempoolElectrumConfig} MempoolElectrumConfig */
// later: @param {MempoolElectrumConfig} config
```

---

## Type Annotation Rules

### R12. Use `x[]` not `Array<x>` for consistency

```javascript
@returns {Promise<Array<ElectrumUtxo>>}     // wrong
@returns {Promise<ElectrumUtxo[]>}          // correct
```

---

### R13. Type-precision ladder: specific `@typedef` > SDK type > `Record<string, unknown>` > (never bare `Object`)

When choosing a type for a property, parameter, or return value, walk this ladder top-to-bottom and stop at the first option that fits:

1. **A specific named `@typedef`** — preferred whenever the shape is known.
2. **An SDK type imported via `@typedef`** — when the SDK already publishes the type for the concept.
3. **`Record<string, unknown>`** — only when the shape is genuinely arbitrary (user-provided metadata blob).
4. **Bare `Object`** — **forbidden as a value type** in `@param`, `@property`, `@returns`, `@type`. (`@typedef {Object} Foo` is fine — that's the typedef-kind marker, see R2.)

If you write `{Object}` outside a `@typedef` declaration, you've taken a shortcut. Find the right type.

**Violation:**
```javascript
@param {Object} txs - The transactions.
@returns {Promise<Object>} The receipt.
```

**Correct:**
```javascript
/** @typedef {import('abstractionkit').MetaTransaction} MetaTransaction */
/** @typedef {import('abstractionkit').UserOperationReceiptResult} UserOperationReceipt */
@param {MetaTransaction[]} txs - The transactions to send.
@returns {Promise<UserOperationReceipt>} The receipt.
```

`Record<string, unknown>` for known shapes is also wrong:
```javascript
@property {Record<string, unknown>} domain     // wrong (known shape)
@property {TypedDataDomain} domain             // correct
```

---

### R14. Use SDK types instead of redundant custom `@typedef`

When the SDK provides a type for a concept, reference it directly instead of redefining a local equivalent.

```javascript
@property {BaseClient} [client]               // wrong (custom)
@property {IElectrumClient} [client]          // correct (SDK type)
```

---

### R15. Create type aliases for SDK types to match module naming

When using types from external SDKs, create a `@typedef` alias matching the module's naming (`Transfer` → `SparkTransfer`).

**Violation:**
```javascript
@returns {Promise<Object | null>} The Spark transfer, or null if not found.
```

**Correct:**
```javascript
/** @typedef {import('@buildonspark/spark-sdk').Transfer} SparkTransfer */
@returns {Promise<SparkTransfer | null>} The Spark transfer, or null if not found.
```

---

### R16. `@returns` must include `| null` when method can return null

If the implementation can `return null`, the `@returns` type must explicitly include `| null`.

```javascript
@returns {Promise<SparkTransfer>}              // wrong (body returns transfer ?? null)
@returns {Promise<SparkTransfer | null>}       // correct
```

---

### R17. Use `@throws` for error conditions

Document errors with `@throws`. Use the specific custom error class, not generic `Error`.

```javascript
@throws {Error} If the configuration is invalid.                                    // wrong
@throws {ConfigurationError} If the configuration has missing required fields.      // correct
```

---

### R18. Use `@see` for external documentation URLs

Link external protocol specs, RFCs, or API docs with `@see`.

```javascript
@see https://electrum.readthedocs.io/en/latest/protocol.html#blockchain-address-get-balance
```

---

### R19. Use `@abstract` for abstract methods, `@interface` for interface classes

```javascript
/** @interface */
export default class IElectrumClient {
  /**
   * Connects to the Electrum server.
   *
   * @abstract
   * @returns {Promise<void>}
   */
  async connect () { throw new Error('Not implemented') }
}
```

---

### R20. Discriminated union types for mutually exclusive config options

When a config object has mutually exclusive options (paymaster token vs. sponsorship policy), model them as a discriminated union, not all properties on one type.

```javascript
/** @typedef {Object} TokenConfig
 *  @property {string} paymasterTokenAddress */
/** @typedef {Object} SponsorshipConfig
 *  @property {string} sponsorshipPolicyId */
/** @typedef {Partial<TokenConfig | SponsorshipConfig>} Config */
```

---

### R21. No blank line between `*/` and the declaration it documents

```javascript
/**
 * Verifies a message's signature.
 */
                                  // wrong (blank line)
async verify (message, sig) { ... }

/**
 * Verifies a message's signature.
 */
async verify (message, sig) { ... }   // correct
```

---

### R22. Define `@typedef` for structured return types

When a method returns an object with multiple properties, define a named `@typedef` instead of an inline anonymous type.

```javascript
@returns {Promise<{confirmed: bigint, unconfirmed: bigint}>}    // wrong
```

```javascript
/** @typedef {Object} GetBalanceResult
 *  @property {bigint} confirmed - Confirmed balance in satoshis.
 *  @property {bigint} unconfirmed - Unconfirmed balance in satoshis. */
@returns {Promise<GetBalanceResult>}
```

---

### R23. Ensure `.d.ts` types match runtime types

Generated `.d.ts` types must reflect runtime behavior. If a value is `bigint` at runtime, the `.d.ts` must not declare it as `number`.

---

### R24. Remove unused `@typedef` imports

If a `@typedef` import is no longer referenced in the file, remove it. Same for types re-exported from `.d.ts` that aren't part of the public API.

---

### R25. Use named `@typedef` for inline object types in `@type` annotations

When a class property's `@type` uses an inline object type with multiple properties, extract it into a top-level `@typedef`. This mirrors R22 but applies to `@type` on class members.

**Violation:**
```javascript
/**
 * Cached viem clients.
 * @protected
 * @type {{ publicClient: PublicClient, bundlerClient: BundlerClient } | undefined}
 */
this._viemClients = undefined
```
Define `ViemClients` as a typedef and reference it by name.

---

### R26. Use `undefined` (not `null`) for uninitialized optional/cached properties

```javascript
/** @protected @type {SomeType | null} */
this._cachedValue = null               // wrong

/** @protected @type {SomeType | undefined} */
this._cachedValue = undefined          // correct
```

---

### R27. Use overload-specific param names, not union names

When a method has `@overload` JSDoc, each overload's `@param` name should reflect the specific type for that overload — not the implementation's generic union name.

**Violation:**
```javascript
/** @overload @param {string | Uint8Array} seedOrAccount - The seed phrase. */
/** @overload @param {WalletAccountEvm} seedOrAccount - An existing account. */
```

**Correct:**
```javascript
/** @overload @param {string | Uint8Array} seed - The seed phrase. */
/** @overload @param {WalletAccountEvm} account - An existing account. */
```

---

### R28. Every documented field needs a meaningful description

`@param`, `@property`, `@returns`, `@throws`, and member descriptions must contain a non-empty, useful sentence that adds information beyond the type and name. Empty descriptions, single-word descriptions that just restate the parameter name, and "the X" tautologies are forbidden.

If a default exists, include it per R4.

For `@protected` methods (part of the extension surface — see AD5), this rule applies fully — protected ≈ public for documentation purposes. Private members under R6 are the only exception.

**Violation (no description):**
```javascript
@property {string} projectName
@property {boolean} [enabled]
```

**Violation (tautology):**
```javascript
@param {Config} config - The config.
@returns {Account} The account.
```

**Correct:**
```javascript
@property {string} projectName - Free-form identifier appended to every UserOperation callData. Max 50 bytes.
@property {boolean} [enabled] - Whether the identifier is appended (default: true).
```

---

### R29. No implementation-detail prefixes in public type names

Type names in the public API describe *what* the type represents, not *how* it's used. Prefixes like `Cached*`, `Internal*`, `Tmp*`, `Resolved*`, `Pending*` describe lifecycle/usage state — those go in code comments or in a `@property` description, not the type name.

```javascript
/** @typedef {Object} CachedTransactionQuote ... */     // wrong
/** @typedef {Object} TransactionQuote ... */           // correct
```

If the cache-vs-fresh distinction matters at the type level (rare), encode it as `@property {boolean} fromCache` rather than baking it into the name.

---

### R30. Compound type names use SDK-aligned word-break casing

When a type name is built from a multi-word technical term that has established casing in the relevant SDK or specification, match that casing exactly.

| Term | Wrong | Correct |
|---|---|---|
| On-chain identifier | `OnchainIdentifier` | `OnChainIdentifier` |
| User operation | `Useroperation` | `UserOperation` |
| Bundler URL | `Bundlerurl` | `BundlerUrl` |
| Typed data | `Typeddata` | `TypedData` |
| Multi-chain | `Multichain` | `MultiChain` |

When in doubt, search the SDK's published types for the exact casing.

---

### R31. Use `Pick<Config, ...>`, not `Partial<Config>`, when the function only needs specific fields

`Partial<Config>` says "any subset, including the empty object" — right for *override* parameters where the caller may supply zero or more fields. Wrong for a method that *requires* a specific subset.

If a method uses three of Config's eight properties, declare exactly those three: `Pick<Config, 'a' | 'b' | 'c'>`. Documents the dependency at the type level; prevents callers from passing unrelated fields.

**Violation:**
```javascript
/** @protected
 *  @param {Partial<EvmErc4337Config>} config - The configuration. */
async _createSafeAccount (config) {
  const v = config.safeModulesVersion ?? '0.3.0'
  const e = config.entryPointAddress
  const i = config.onchainIdentifier
  // only these three are read
}
```

**Correct:**
```javascript
/** @protected
 *  @param {Pick<EvmErc4337Config, 'safeModulesVersion' | 'entryPointAddress' | 'onchainIdentifier'>} config -
 *    The Safe-construction subset of the wallet config: module version, entry-point address, and optional on-chain identifier. */
async _createSafeAccount (config) { ... }
```

When the same `Pick` is used in multiple places, lift it to a named typedef.

---

## Type Definition Conventions

### TD1. Run `npm run build:types` before committing after API changes

After modifying public JSDoc, run `npm run build:types` (`tsc`) to regenerate `.d.ts`. Commit the updated type files alongside the source changes.

---

### TD2. Update `types/index.d.ts` when adding new public types

Every new `@typedef` that's part of the public API must appear in the generated `types/index.d.ts`. Verify after `build:types`.

---

### TD3. Export new types from `index.js`

New public types must be re-exported from `index.js` so consumers can import them.

```javascript
// index.js
export { default as WalletAccountBtc } from './src/wallet-account-btc.js'
/** @typedef {import('./src/wallet-account-btc.js').GetBalanceResult} GetBalanceResult */
```

---

### TD4. Re-export parent module types from child modules

When a module extends another WDK module's public types, re-export those types from the extending module's `index.js` so consumers don't need a direct dependency on the parent.

---

### TD5. Export custom error types from both `index.js` and `types/`

Custom error classes must be exported from `index.js` and have corresponding declarations in `types/`. Consumers need to catch and type-check errors.

---

### TD6. Exclude internal types from `types/` folder

Types for internal modules (prefixed with `_`, inside `src/internal/`) must not appear in the public `types/` output. Use `tsconfig.json` exclusions or `@internal`.

---

### TD7. Interfaces use `interface` in `.d.ts`

JSDoc `@interface` classes must generate `interface` declarations (not `class`) in `.d.ts`.

```typescript
export class IElectrumClient { ... }            // wrong
export interface IElectrumClient { ... }        // correct
```

---

### TD8. Abstract classes use `abstract class` in `.d.ts`

JSDoc `@abstract` classes must generate `abstract class` declarations.

---

### TD9. Regenerate `.d.ts` after addressing review changes

After resolving review feedback that touches JSDoc or type annotations, re-run `npm run build:types` and include the updated `.d.ts` files in the follow-up commit.

---

## Review Workflow

1. **JSDoc structure** — every block starts with a description (R1); `{Object}` not `{object}` (R2); no blank line before declaration (R21).
2. **Tag formatting** — dash separators in `@param`/`@property` (R3); defaults in description text (R4).
3. **Visibility** — `@private` only on class members (R5); private members get minimal JSDoc (R6); `@protected` has full annotation (R7).
4. **Inheritance** — overrides have full JSDoc (R8); `@implements` vs `@extends` (R9); `@abstract`/`@interface` (R19).
5. **Imports** — `@typedef {import(...)}` not runtime imports (R10); short aliases not inline paths (R11); no unused imports (R24).
6. **Type annotations** — `x[]` not `Array<x>` (R12); precision ladder (R13); SDK types preferred (R14, R15); discriminated unions (R20); typedef for structured returns (R22); typedef for inline `@type` on properties (R25); `undefined` not `null` for cached props (R26); `Pick<Config>` not `Partial<Config>` (R31).
7. **Completeness** — meaningful descriptions on every documented field (R28); `| null` when applicable (R16); `@throws` for errors (R17); `@see` for docs (R18).
8. **Naming** — no impl-detail prefixes (R29); SDK-aligned casing (R30); overload-specific param names (R27).
9. **`.d.ts` alignment** — runtime types match (R23); `build:types` after changes (TD1, TD9); public types exported (TD2-TD5); internal excluded (TD6); interface/abstract correct (TD7-TD8); diff `types/` after regen.
