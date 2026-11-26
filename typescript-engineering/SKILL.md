---
name: typescript-engineering
description: Rules for writing, modifying, refactoring, or reviewing TypeScript (.ts, .tsx, .mts, .cts) in any project. Use this skill whenever you touch TypeScript code, add a dependency, parse or validate external data (API responses, request bodies, env/config, file contents), or are asked to review TypeScript, even for small edits. Contains hard checkpoints where you must ask the user before proceeding.
license: MIT
---

# TypeScript Engineering

General good-practice (readability, naming, simplicity, `const` by default) is assumed. This skill covers only what is easy to get wrong.

## 1. Before you write anything

Inspect the project first. Do not rely on assumptions.

1. Read `tsconfig.json` (note `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, module settings) and the lint/format config.
2. Read `package.json`: dependencies, and the scripts for test, lint, and typecheck.
3. Find how the project already solves your problem (grep for similar code). Reuse its types, utilities, and patterns.
4. Do not introduce a second pattern for something the project already handles (error modeling, HTTP client, logging, date handling, state, etc.).

Make focused changes. No unrelated refactors, no config changes, and no behavior changes unless asked.

## 2. Type safety rules

- Use `unknown`, not `any`. Narrow before use.
- Avoid `as` assertions and `!` non-null assertions. If one is genuinely unavoidable, add a one-line comment saying why it is safe.
- `as` on data from outside the program (JSON, network, storage, `process.env`) is never acceptable. See section 4.
- Model states so invalid ones are unrepresentable. Prefer discriminated unions over optional-field bags.
- Add an exhaustiveness check (`never`) when switching over a union.
- Annotate exported function signatures and other boundaries. Let inference handle local variables.
- Prefer `satisfies` over `as` when checking a value against a type without widening.

```ts
// Bad: optional-field bag allows impossible states
type Result = { ok: boolean; data?: User; error?: string };

// Good: each state carries exactly what it needs
type Result =
  | { ok: true; data: User }
  | { ok: false; error: string };
```

## 3. Error handling

- Follow the project's existing error model (throw vs. Result types). Do not mix them.
- Never swallow errors with an empty `catch`. Either handle, rethrow, or log with context.
- When wrapping an error, preserve the original: `new Error("context", { cause: err })`.
- In `catch`, the error is `unknown`. Narrow it before reading `.message`.
- Do not add defensive checks for conditions the types or surrounding code already guarantee. Do validate at trust boundaries (section 4).

## 4. Validation of untrusted data (STOP AND ASK)

**Scope:** structured data crossing a trust boundary: API responses, request bodies, query params, env vars, config files, file contents, `JSON.parse` output, message-queue payloads.

**Out of scope:** trivial local checks such as `typeof x === "string"`, null checks, or narrowing inside already-typed code. Do these freely.

Rules:

- TypeScript types do not validate runtime data. A type annotation or `as` cast is not validation.
- If the project already uses a schema library (Zod, Valibot, io-ts, ArkType, etc.), use it. Never add a second one.
- If the project has no schema library, **STOP AND ASK** (see section 6) before writing validation. Do not hand-roll validation of structured data by default.

## 5. Dependencies and reinventing the wheel

- Check existing dependencies and utilities before writing common functionality.
- Prefer built-in language/runtime features for simple needs. Do not add a package for something like a one-line `Array.prototype` operation.
- For non-trivial common problems (schema validation, date math, retries, parsing, HTTP, CLI args), prefer an established, maintained library over a custom implementation.
- If no suitable library is installed and one would clearly help, **STOP AND ASK** before adding it or writing a custom substitute.

## 6. How to STOP AND ASK

Both checkpoints (sections 4 and 5) use this format. Keep it short:

1. State the decision in one sentence.
2. Offer 2-3 concrete options, each with a one-line tradeoff.
3. Give your recommendation and why.
4. Wait for an answer. Do not proceed with a guess.

Example:

> This endpoint parses an untrusted JSON body and the project has no schema library. Options: (a) add Zod, the most common choice, with good TS inference; (b) add Valibot, smaller bundle size; (c) hand-written checks, no dependency but more code and drift risk. I recommend (a). Which do you want?

**If no human can answer** (autonomous or non-interactive run): do not silently decide. Choose the most conventional option, isolate it behind a small module so it is easy to swap, and flag it prominently in your final summary as "decision needing review."

Never silently bypass a STOP AND ASK checkpoint.

## 7. Verification

Before declaring work done:

- Run the project's typecheck (usually a `package.json` script, otherwise `tsc --noEmit`).
- Run the linter and the relevant tests. Use the project's existing test framework and conventions.
- Add or update tests for new behavior and edge cases (empty, invalid, boundary inputs). Test observable behavior, not implementation details.
- If you could not run a check, say so explicitly instead of implying it passed.

## 8. Judgment

When these rules and a reasonable engineering call conflict, pick the simplest option that meets the requirements, and say what tradeoff you made. When an ambiguous decision would change a public API or user-visible behavior, ask rather than assume. The STOP AND ASK checkpoints are the only hard rules here. Everything else is guidance.