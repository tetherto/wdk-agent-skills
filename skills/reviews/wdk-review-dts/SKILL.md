---
name: wdk-review-dts
description: Review and maintain WDK generated .d.ts type definitions (manual updates, exports, internal exclusion, interface/abstract emission, no build:types).
user-invocable: true
argument-hint: "[project types/ directory or root]"
---

# WDK .d.ts Maintenance Review

You are reviewing the generated `.d.ts` type definitions of a WDK module. Apply the rules below. The rules are repository-agnostic — no WDK module is a "golden reference"; apply every rule to every module equally, and if existing declarations conflict with a rule, the declaration is the candidate for change, not the rule. Type *accuracy* itself (runtime types matching the `.d.ts`) is rule R23 in the companion skill **wdk-review-jsdocs**.

For each violation found, report:
1. **Rule violated** (e.g., TD1, TD2, etc.)
2. **File and line**
3. **The offending code**
4. **Suggested fix** with corrected code

Group findings by file. After all files are reviewed, provide a summary with total violation counts per rule.

---

## Type Definition Conventions

### TD1. Update `.d.ts` manually after API changes — do not run `npm run build:types`

After modifying public JSDoc, **do not** run `npm run build:types` (`tsc`). Instead, hand-edit the affected `.d.ts` files so they reflect the JSDoc/type changes, and commit the updated type files alongside the source changes. Only touch the declarations that actually changed.

---

### TD2. Update `types/index.d.ts` when adding new public types

Every new `@typedef` that's part of the public API must appear in `types/index.d.ts`. Add the corresponding declaration manually (see TD1).

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

### TD9. Update `.d.ts` manually after addressing review changes

After resolving review feedback that touches JSDoc or type annotations, hand-edit the affected `.d.ts` files (do not run `npm run build:types`) and include the updated declarations in the follow-up commit.

---

## Review Workflow

1. **Never regenerate** — do not run `npm run build:types` (`tsc`). Hand-edit the affected `.d.ts` files so they reflect the JSDoc/type changes, touching only the declarations that actually changed, and commit them alongside the source changes (TD1, TD9).
2. **Public types exported** — every new public `@typedef` appears in `types/index.d.ts` and is re-exported from `index.js`; parent-module types are re-exported from child modules; and custom error classes are exported from both `index.js` and `types/` (TD2–TD5).
3. **Internal types excluded** — types for internal modules (prefixed with `_`, inside `src/internal/`) must never appear in the public `types/` output; enforce this with `tsconfig.json` exclusions or `@internal` (TD6).
4. **Interface / abstract emission** — confirm `@interface` classes emit `interface` declarations (not `class`) and `@abstract` classes emit `abstract class` declarations in the `.d.ts` (TD7, TD8).
5. **Diff and audit** — diff the `types/` folder against its previous state and review every edit for correctness; treat every removed declaration as a possible public-API regression until you have confirmed the removal is intentional.
