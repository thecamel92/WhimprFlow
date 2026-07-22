# TypeScript style

Applies to `**/*.ts`, `**/*.tsx`.

## Types at boundaries

- Explicit parameter + return types on **exported** functions and shared utilities.
  Let TypeScript infer obvious local variables — don't annotate noise.
- `interface` for object shapes that get extended; `type` for unions, intersections,
  and mapped types.
- **Prefer string literal unions over `enum`** (`type CleanupLevel = "light" | "medium" | "high"`).

## Never `any` at a boundary — use `unknown`, then narrow

`any` switches type-checking off and the failure is silent. For anything arriving from
outside the system — a scrape, an API response, parsed JSON, a saved site — use
`unknown` and narrow it. The narrowing is what proves the shape.

```ts
// WRONG — any removes the guarantee, and nothing tells you when it breaks
function getErrorMessage(error: any) { return error.message }

// CORRECT — unknown forces you to prove what you have
function getErrorMessage(error: unknown): string {
  if (error instanceof Error) return error.message
  return "Unexpected error"
}
```

Anything arriving from outside this process — a Tauri `invoke` payload, a worker-process
response, a parsed JSON settings file — is untrusted the same way: parse it into a known
shape once, at the boundary, not spread as `any` through the pipeline.

**One source of truth for a boundary shape.** If a type is hand-written *next to* the
code that validates it, the two will drift. Derive one from the other. (If a
schema-validation library is ever added here, `type T = z.infer<typeof schema>` is the
canonical form — one declaration giving both the runtime check and the compile-time
type. Not currently a dependency; don't add one just for this.)

## Errors

- `catch (error: unknown)` and narrow — never `catch (e: any)`.
- **Never silently swallow.** No empty `catch {}`, no `.catch(() => [])` where the caller
  can't tell "empty" from "broken". If a fallback is deliberate, the caller must be able
  to distinguish the two states.
- Preserve the original error when re-throwing (`cause`), and log the error object, not
  just a message string.

## React

- Props get a named `interface`; type callback props explicitly.
- Don't use `React.FC` without a specific reason.

```ts
interface UserCardProps {
  user: User
  onSelect: (id: string) => void
}
function UserCard({ user, onSelect }: UserCardProps) { /* … */ }
```

## Immutability

Return new objects; don't mutate arguments. Take `Readonly<T>` when a function must not
write to its input.

## Hygiene

- No `console.log` left in code paths that ship — use the project's logging convention.
- No magic numbers: name the constant.
- Early-return instead of stacking nesting past ~4 levels.
