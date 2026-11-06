---
name: wdk-review-errors
description: Review WDK wallet and protocol modules for error-contract compliance with @tetherto/wdk-wallet (declared @throws only, external-dependency error mapping, LSP-safe narrowing, NotImplementedError/AssertionError usage).
user-invocable: true
argument-hint: "[file or directory path to source files]"
---

# WDK Error Contract Review

Review the source files at the provided path (or the current project's `src/` directory if none given). Apply all rules below.

For each violation found, report:
1. **Rule violated** (e.g., E1, E2, etc.)
2. **File and line**
3. **The offending code**
4. **Suggested fix** with corrected code

Group findings by file. After all files are reviewed, provide a summary with total violation counts per rule.

---

# WDK Error Standards

You are reviewing source files for a WDK (Wallet Development Kit) wallet or protocol module. Every such module implements an interface or extends an abstract class from `@tetherto/wdk-wallet` (e.g., `IWalletAccount`, `WalletAccountReadOnly`, `ISwapProtocol`, `ISwidgeProtocol`). The `@throws` tags of each method in that super-class **are the method's error contract**. Apply the rules below to identify places where an implementation breaks that contract and suggest fixes.

The rules are repository-agnostic. No WDK module is a "golden reference" — apply every rule to every module equally. If existing code conflicts with a rule, the code is the candidate for change, not the rule.

Breaking an error contract violates the Liskov Substitution Principle (LSP): a consumer written against the interface catches exactly the errors the interface declares, so any other error an implementation throws escapes that handling. Treat E1 and E5 violations as **critical** bugs.

To review a method, always open its definition in the super-class first (from the installed `node_modules/@tetherto/wdk-wallet/src/` — `errors.js` for wallet errors, `protocols/errors.js` for protocol errors), and compare the implementation against it — never against another implementation.

---

## Error Hierarchy (reference)

All errors extend `WdkError`. Throwing a **sub-class** of a declared error satisfies the contract (e.g., `ReadOnlyAccountRequiredError extends AccountRequiredError`, so a method declaring `@throws {AccountRequiredError}` may throw either).

| Module | Errors |
|--------|--------|
| `@tetherto/wdk-wallet` | `ValueError`, `NoSuchElementError`, `InvalidSignerError`, `InvalidTokenError`, `ProviderRequiredError`, `ProviderError` (with `ProviderErrorReason`), `TransactionError` (with `TransactionErrorReason`), `TransferError` (with `TransferErrorReason`), `MaximumFeeExceededError`, `TimeoutError`, `DisposalError`, `UnsupportedOperationError` |
| `@tetherto/wdk-wallet/protocols` | `AccountRequiredError`, `ReadOnlyAccountRequiredError`, `SwapError`, `BridgeError`, `SupplyError`, `WithdrawError`, `BorrowError`, `RepayError`, `BuyError`, `SellError`, `SwidgeError`, `SdaError` |
| Special (see E6–E8) | `NotImplementedError`, `AssertionError` |

This table is a snapshot — the installed `errors.js` files are authoritative.

---

## Rules

### E1. Only throw errors declared in the super-class `@throws` (or their sub-classes)

A method may throw only the errors listed in the `@throws` tags of its definition in the super-class, or errors whose class extends one of them. Throwing anything else — a generic `Error`, a `TypeError`, a raw error from an external library, or a `WdkError` the super-class doesn't declare — is a **critical** LSP violation. The same applies to the method's own `@throws` tags: an implementation must never declare an error that is not in the super-class contract.

**Violation** (`WalletAccountEvm.sendTransaction` in wdk-wallet-evm — `IWalletAccount.sendTransaction` declares no `@throws {Error}`):
```javascript
/**
 * ...
 * @throws {Error} If the simulation of the transaction reverts, as an ethers error with code `CALL_EXCEPTION`.
 */
async sendTransaction (tx) {
  ...
  return await this._sendPopulated(tx, ...)   // lets the ethers error escape as-is
}
```

**Correct** — translate the failure into the declared error, a `TransactionError` with the proper reason:
```javascript
/**
 * ...
 * @throws {TransactionError} If the transaction fails with an error.
 */
async sendTransaction (tx) {
  ...
  try {
    return await this._sendPopulated(tx, ...)
  } catch (error) {
    if (isError(error, 'INSUFFICIENT_FUNDS')) {
      throw new TransactionError('The account does not have enough funds to send the transaction.', {
        reason: TransactionErrorReason.INSUFFICIENT_BALANCE,
        cause: error
      })
    }

    throw error
  }
}
```

Always pass the original error as `cause` when translating, and always set `reason` on errors that require one (`ProviderError`, `TransactionError`, `TransferError`).

---

### E2. Map the errors of external dependencies to declared errors — after checking what they can throw

For every call into an external module (SDKs, RPC clients, `ethers`, `fetch`, etc.), check which errors the function or method can throw, decide which of them can actually occur in this implementation, and translate each of those into an error from the super-class contract (E1). Never let an expected external failure (network failure, auth failure, revert, insufficient funds, invalid input rejected by the library, …) escape unmapped.

Translate selectively: match on the error's class or code and re-throw what you didn't match. A blanket `catch` that converts every error into one declared error also converts bugs into contract errors and hides them (see E3).

**Violation:**
```javascript
async getBalance () {
  const res = await fetch(`${this._url}/balance/${address}`)   // network errors escape as TypeError
  ...
}

async quoteSwap (options) {
  try {
    return await this._sdk.quote(options)
  } catch (error) {
    throw new SwapError(error.message)                          // blanket catch: every failure becomes SwapError, no cause
  }
}
```

**Correct:**
```javascript
async getBalance () {
  let res
  try {
    res = await fetch(`${this._url}/balance/${address}`)
  } catch (error) {
    throw new ProviderError('Failed to connect to the provider.', { reason: ProviderErrorReason.NETWORK_ERROR, cause: error })
  }
  ...
}

async quoteSwap (options) {
  try {
    return await this._sdk.quote(options)
  } catch (error) {
    if (error instanceof SdkNoRouteError) {
      throw new SwapError('No route found for the swap.', { cause: error })
    }

    throw error
  }
}
```

---

### E3. Don't handle external errors that cannot occur by construction

You do not need to handle every error an external dependency can throw. If the implementation guarantees the condition that triggers an error never arises, don't expect it and don't handle it — e.g., a library function that throws when no provider is set, called by an implementation that always passes a valid provider. Letting such an error propagate untouched is the desired behavior: if it ever surfaces, it immediately tells us that our own implementation is wrong (e.g., it doesn't actually always pass a provider).

Flag handlers that translate or swallow errors which cannot occur given the surrounding code — they add dead code and, worse, would disguise our own bugs as legitimate contract errors.

**Violation:**
```javascript
// this._provider is always set in the constructor
try {
  return await sdk.getQuote({ provider: this._provider, ...options })
} catch (error) {
  if (error instanceof SdkMissingProviderError) {
    throw new ProviderRequiredError('The protocol requires a provider.')   // can never happen — and would hide a bug if it did
  }
  throw error
}
```

**Correct:**
```javascript
return await sdk.getQuote({ provider: this._provider, ...options })
```

When in doubt whether a condition can occur, verify it from the code paths that reach the call (constructor, config validation, earlier guards) — not from assumptions.

---

### E4. Implementations may narrow the super-class `@throws` — never widen it

A method may throw **fewer** errors than its super-class definition, excluding them from its own `@throws` tags, provided it genuinely never needs to throw them (see E5). For example, `ISwidgeProtocol.quoteSwidge` declares `ReadOnlyAccountRequiredError` for protocols that need a read-only account to quote costs; an implementation that can estimate costs without an account never throws it and may drop it from its `@throws`.

Narrowing is optional — keeping an inherited tag the implementation can't throw is not a violation. Widening is always a violation (E1).

**Correct (narrowed):**
```javascript
/**
 * Quotes the estimated costs and output of a swidge operation.
 *
 * @param {SwidgeOptions} options - The swidge options.
 * @returns {Promise<SwidgeQuote>} The quoted swidge details.
 * @throws {ValueError} If the swidge options are not valid.
 * @throws {ProviderError} If the provider fails to quote the swidge.
 * @throws {SwidgeError} If the swidge fails with an error.
 */
async quoteSwidge (options) { ... }   // never needs an account, so ReadOnlyAccountRequiredError is omitted
```

Also flag the inverse inconsistency: an error the body can throw (directly or by propagating from a callee) that is in the super-class contract but missing from the method's own `@throws`.

---

### E5. Narrowing must never change the method's behavior

An error may be excluded only when the condition that triggers it cannot arise in the implementation. Replacing a declared error with a different outcome — returning `0`, `0n`, `null`, `false`, an empty array, or silently ignoring the condition — changes the method's behavior and violates LSP, even if it looks like a purely semantic change. This is a **critical** violation.

**Violation** (`getTokenBalance` declares `@throws {ValueError} If the token's address is not valid.`):
```javascript
async getTokenBalance (tokenAddress) {
  if (!isAddress(tokenAddress)) {
    return 0n
  }
  ...
}
```

**Correct:**
```javascript
async getTokenBalance (tokenAddress) {
  if (!isAddress(tokenAddress)) {
    throw new ValueError(`'${tokenAddress}' is not a valid token address.`)
  }
  ...
}
```

---

### E6. Never declare `NotImplementedError` or `AssertionError` in `@throws`

These are special errors that are not part of any method's contract. `NotImplementedError` signals a missing override; `AssertionError` signals a bug in our implementation. Neither is something a consumer should ever handle, so neither may appear in a `@throws` tag — in interfaces, abstract classes, or implementations.

```javascript
@throws {NotImplementedError} If the method is not implemented.            // wrong — remove
@throws {AssertionError} If the account's address is not set.              // wrong — remove
```

---

### E7. `NotImplementedError` is only a placeholder for missing implementations

`NotImplementedError` may be thrown only by:
- methods of `@interface` classes, which always throw it;
- `@abstract` methods of abstract classes, which have no default implementation and must be overridden by sub-classes.

A concrete class must never throw `NotImplementedError`: every abstract/interface method must be overridden with a real implementation. If an implementation cannot support an operation, it may throw `UnsupportedOperationError` **only** when the super-class declares it for that method (E1); otherwise the operation must be implemented.

**Violation:**
```javascript
export default class WalletAccountReadOnlyFoo extends WalletAccountReadOnly {
  async getTokenBalance (tokenAddress) {
    throw new NotImplementedError('getTokenBalance(tokenAddress)')   // concrete class still throws the placeholder
  }
}
```

---

### E8. `AssertionError` only for internal invariants — and as rarely as possible

`AssertionError` is reserved for internal checks that must never fail in production: if one is thrown, our implementation is wrong. Use it only to guard delicate invariants that would otherwise be very hard to debug — e.g., `WalletAccountReadOnly.getAddress` asserts the address is set, catching a sub-class that calls `super(undefined)` and then forgets to override `getAddress`.

Flag any `AssertionError` used for:
- validating caller input — use `ValueError` (or the declared error);
- runtime conditions that can legitimately occur (missing provider, network failure, disposed account, …) — use the declared error;
- invariants that are trivially guaranteed by the surrounding code — the check is noise; remove it.

Generally we don't need assertion errors: question every new one.

---

## Review Workflow

1. **Resolve contracts** — for each public method, locate its definition in the `@tetherto/wdk-wallet` interface or abstract class it implements/overrides and collect that method's `@throws` tags.
2. **Thrown errors** — every `throw` in the method body (and every error propagating from internal helpers it calls) is a declared error or a sub-class of one; no generic `Error`, built-in, or undeclared `WdkError` (E1). Required `reason` and `cause` are set when translating.
3. **External dependencies** — for each external call, list what it can throw; every error that can occur is translated into a declared error with selective matching and `cause` (E2); errors that cannot occur by construction are not handled (E3).
4. **Own `@throws`** — the method's `@throws` is a subset of the super-class contract (E1, E4) and lists every contract error the body can actually throw (E4).
5. **Behavior preservation** — no declared error condition is replaced by a sentinel return value or silently ignored (E5).
6. **Special errors** — `NotImplementedError` and `AssertionError` never appear in `@throws` (E6); `NotImplementedError` is thrown only by interface and abstract methods (E7); every `AssertionError` guards a genuine internal invariant (E8).

> JSDoc formatting of `@throws` tags (one tag per condition, specific error class) is rule R17 in the companion skill **wdk-review-jsdocs**; exporting custom error classes from `index.js` and `types/` is rule TD5 in **wdk-review-dts**.
