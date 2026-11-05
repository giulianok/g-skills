---
name: typescript-style
description: Giuliano's personal TypeScript/JavaScript coding style. Use whenever writing, editing, refactoring, or reviewing TypeScript or JavaScript code (.ts, .tsx, .js, .mjs) — file layout (types and constants on top), JSDoc on every function, arrow functions, functional/immutable style, small declarative functions, and strict blank-line spacing rules.
---

# TypeScript Coding Style

Apply these rules to every TypeScript/JavaScript file you write or touch. When editing an existing file, match these rules in the code you add, but don't reformat unrelated code unless asked.

If a project has its own lint/prettier config, the formatter wins on mechanical details (quotes, semicolons, line width). These rules cover what formatters don't.

## 1. File layout: types and constants on top

Order every module top-to-bottom like this:

1. **Imports**
2. **Schemas** (e.g. zod) — needed first because types are inferred from them
3. **Types** — `type` aliases, inferred types, documented object types
4. **Constants** — `UPPER_SNAKE_CASE` config, endpoints, lookup tables (`as const` where useful)
5. **Module state** — the rare `let` caches (see §4 exceptions)
6. **Functions** — private helpers first, exported functions after the helpers they use

Never declare a type or constant in the middle of the functions section.

```ts
import { z } from 'zod';

const userSchema = z.object({ id: z.string(), name: z.string() });

type User = z.infer<typeof userSchema>;
type Env = 'dev' | 'prod';

const API_URLS: Record<Env, string> = {
  dev: 'https://api.dev.example.com',
  prod: 'https://api.example.com',
} as const;

/**
 * Fetches a user by id.
 * @param id user id
 * @param env target environment
 * @returns the parsed user
 */
export const getUser = async (id: string, env: Env): Promise<User> =>
  userSchema.parse(await fetch(`${API_URLS[env]}/users/${id}`).then((r) => r.json()));
```

## 2. Every function has JSDoc

Every function — exported or not, however small — gets a JSDoc block:

- A one-line description in the present tense ("Gets…", "Returns…", "Finds…", "Extracts…").
- `@param name description` for each parameter whose meaning isn't completely obvious.
- `@returns description` when the function returns something.
- `@example` when the input/output format is non-obvious (URLs, version strings, parsing).

Non-trivial object types also get a short `/** ... */` explaining what they represent.

```ts
/**
 * Extracts the tenant name from a module map URL.
 * @example "https://cdn.example.com/map/acme-prod/map.json" -> "acme"
 * @returns tenant name, or an empty string if no URL is configured
 */
```

Inline comments explain *why*, not *what*.

## 3. Arrow functions

- Use `const name = (...) => ...` for all functions. No `function` declarations (unless a library or hoisting genuinely requires it).
- Use an **expression body** when the function is a single expression; use a block body only when it needs statements.
- Add explicit return types on exported functions and wherever inference isn't obvious.
- Prefer default parameters over in-body fallbacks: `(cwd = CWD) => ...`.
- Put parameters on separate lines once the signature gets long.

## 4. Functional programming, no mutation

- `const` by default. `let` only when it's truly needed.
- Never mutate inputs. Build new values with spread, `map`, `filter`, `reduce`, `Object.entries`/`Object.fromEntries`.
- Prefer chained array methods over loops: `.filter().map().filter()`.
- Prefer expressions over statements: ternaries, `??`, `?.`, `||` fallbacks.
- Use type predicates to narrow inside chains: `.filter((t): t is [string, string] => !!t[0])`.
- Use generics on `map`/`reduce` when inference needs help: `.reduce<Record<string, string>>(...)`.
- Validate untrusted data (env, JSON, HTTP responses) with schemas and infer types from them, rather than casting.

**Allowed exceptions (call them out with a comment):**

- A library or API forces mutation.
- Performance is *really* affected (hot paths, very large collections). Don't use this as an excuse for normal code.
- A `for...of` loop that needs early `return`/`break`. Add a justification, e.g.
  `// eslint-disable-next-line no-restricted-syntax -- can stop early when returning`.
- Module-level lazy caches: `let CACHE: T | null = null;` filled once with `CACHE = CACHE ?? compute()`.

## 5. Small functions that do one *declarative* thing

- Each function does one declarative thing that its name says: `getPkg`, `getArtifacts`, `getSupportedVersions`, `findMatchingVersionInRanges`.
- Name functions after what they produce: `get*`, `find*`, `is*`/`has*`, `parse*`, `to*`, or a plain noun for derived values (`tenancy`).
- **Balance.** Don't split into dozens of trivial one-liners that make the flow hard to follow, and don't write long multi-purpose functions either. Extract a helper when it gets a meaningful name, is reused, or makes the caller read like a description of the steps.
- Handle errors with guard clauses: check, `throw new Error(...)` with a descriptive message (include the bad value in a template literal), then continue on the happy path without extra nesting.

## 6. Spacing (blank lines)

Blank lines separate *kinds* of statements. They don't separate statements of the same kind.

- **Consecutive declarations stay together.** No blank line between `const a` and `const b`, even when they span several lines.
- **Blank line before and after side-effect calls** (a statement that is a function call whose result isn't assigned).
- **Blank line before and after control-flow blocks** (`if`, `for`, `try`, `switch`), unless the block is the first or last statement of its scope.
- **Blank line before `return`**, unless it is the only statement in the block.
- After a block or call, the next group of declarations starts after a blank line.

```ts
// Good
const a = getA();
const b = getB(a);

doSomething(b);

if (!b) {
  throw new Error(`Invalid b: ${b}`);
}

const c = toC(b);

return { a, b, c };
```

```ts
// Bad
const a = getA();

const b = getB(a);       // related declarations split apart

doSomething(b);
if (!b) {                 // block glued to the call
  throw new Error(`Invalid b: ${b}`);
}

const c = toC(b);
return { a, b, c };       // return glued to the declaration
```

## 7. Naming and small conventions

- `camelCase` for variables and functions, `PascalCase` for types, `UPPER_SNAKE_CASE` for module-level constants and caches.
- Suffix schemas with `Schema` (`userSchema`) and derive the type from them (`type User = z.infer<typeof userSchema>`).
- Use `Record<K, V>` for lookup maps and `as const` for literal config.
- Use the `node:` prefix for Node built-ins (`import fs from 'node:fs'`).
- Use destructuring for tuples and objects: `const [major, minor, patch] = version.split('.')`.
- Use template literals for interpolated strings and error messages.

See `references/examples.md` for a full annotated example.
