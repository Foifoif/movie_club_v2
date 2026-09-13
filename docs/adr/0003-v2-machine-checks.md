---
status: accepted
---

# The v2 machine checks: few rules, none of them negotiable

The map's standing principle is *if a convention can be a lint rule, it must be
a lint rule* — because a cold agent session obeys a failing build every time and
a markdown file only sometimes. This ADR cashes that principle out, and in doing
so sets its limit: **a rule that fires on legitimate code teaches everyone to
reach for ignore comments, which is worse than no rule.** So v2 enforces a short
list of genuinely mechanical invariants, makes the architectural ones
unsuppressible, and records everything else as prose on purpose.

This is a hobby site. The aim is clean, not locked down.

Resolves map ticket
[The lint-rule set: which invariants become machine checks](https://github.com/a-movie-club/movie_club_v2/issues/12).
Builds on [ADR-0001](0001-v2-schema.md) and [ADR-0002](0002-v2-rls-policy-model.md),
and on the findings of
[the vertical-slice prototype](https://github.com/a-movie-club/movie_club_v2/issues/10).

## The whole inventory

| Invariant | Mechanism | Where it runs |
|---|---|---|
| Another feature is imported only through its `index.ts` | ESLint `no-restricted-imports` | pre-commit, CI |
| The Supabase client and generated DB types are imported only from `*.data.ts` | ESLint `no-restricted-imports` | pre-commit, CI |
| Neither of the above can be suppressed inline | `@eslint-community/eslint-plugin-eslint-comments` | pre-commit, CI |
| `anon` holds zero privileges in schema `v2` | pgTAP | CI |
| Every table in `v2` has RLS enabled | pgTAP | CI |
| Every `auth.users` row has a `v2.members` row | pgTAP | CI |
| Strict TypeScript | `tsc` | CI |

Everything not in this table is prose — see [Deliberately not mechanised](#deliberately-not-mechanised).

## Lint: ESLint 10 + Prettier

Flat config (`eslint.config.js`; ESLint 10 removed eslintrc entirely). Prettier
formats. The baseline is deliberately small:

- `eslint:recommended`
- `typescript-eslint` — the **non-type-checked** recommended set
- `eslint-plugin-react-hooks`
- `@tanstack/eslint-plugin-query` and `@tanstack/eslint-plugin-router`
- `@eslint-community/eslint-plugin-eslint-comments`

ESLint runs with **`--max-warnings 0`**. A warning nobody has to fix is a
comment with a worse UI.

No `eslint-plugin-boundaries`, no `eslint-plugin-import-x`. Both architectural
rules ride on stock `no-restricted-imports`, which supports glob patterns and a
custom message; TypeScript already catches unresolved imports; and an import
resolver is config surface agents routinely get wrong.

### The two boundary rules

Illustrative — the exact paths, the client module's location and the `@/`
alias belong to
[Folder conventions and the data-layer contract](https://github.com/a-movie-club/movie_club_v2/issues/13)
and land with the scaffold. What is fixed here is the **shape**: a filename
suffix for the data-layer exemption, and an `index.ts` public surface per feature.

```js
const featureBoundary = {
  group: ['@/features/*/**'],
  message:
    'Import from `@/features/<name>` — its `index.ts` is the contract. ' +
    'Deep paths into another feature are private. Within the same feature, use a relative import. ' +
    "If you need something it doesn't export, either export it there, " +
    'or move it to `src/components`/`src/lib` — it now has two consumers.',
};

const dataLayer = {
  group: ['@supabase/supabase-js', '@/lib/supabase', '**/database.types'],
  message:
    'Only `*.data.ts` may import the Supabase client or generated database types. ' +
    "Move this query into the feature's `*.data.ts` and import its TanStack Query hook instead — " +
    'components consume hooks and feature models, never database rows.',
};

export default [
  // ...baseline configs
  {
    files: ['src/**/*.{ts,tsx}'],
    rules: {
      'no-restricted-imports': ['error', { patterns: [featureBoundary, dataLayer] }],
    },
  },
  {
    files: ['src/**/*.data.ts', 'src/lib/supabase/**'],
    rules: {
      'no-restricted-imports': ['error', { patterns: [featureBoundary] }],
    },
  },
];
```

**Gotcha worth keeping:** in flat config, a later block that sets the same rule
**replaces** its options rather than merging them. That is why the exempt block
restates `featureBoundary` instead of listing only what it lifts. Any future
restricted-import block must restate every pattern that still applies, or it
silently switches the others off for those files.

The **feature-boundary message carries the promotion rule.** The rule itself
can't be mechanised (see below), but the moment an agent reaches into another
feature is exactly the moment it applies — so its trigger condition is stated
in the one place an agent is guaranteed to read.

The data-layer rule restricts the generated types file as well as the client.
That is the mechanical half of the vertical slice's first correction, where a
generated row leaked `release_date` into a component: components can no longer
*name* a row type. The other half — whether a given mapping still leaks database
semantics — is judgment, and stays prose.

The suffix, rather than a `data/` directory, is deliberate: one glob, no
`data/helpers.ts` sneaking under the exemption, and whether a file may touch the
client is visible in its own name.

### The architectural rules cannot be suppressed

An agent that hits a failing rule has two moves: fix the code, or silence the
rule. Silencing is always cheaper. For these two rules it is not available:

```js
rules: {
  '@eslint-community/eslint-comments/no-restricted-disable': ['error', 'no-restricted-imports'],
  '@eslint-community/eslint-comments/no-unlimited-disable': 'error',
},
```

`no-unlimited-disable` closes the obvious bypass — a bare `/* eslint-disable */`
names no rule at all. Everywhere else, suppressions remain available.

The remaining way round is editing `eslint.config.js`. That is intended: it
turns "silence the architecture" into a pull request a human reads.

Install the **scoped** `@eslint-community` package. The unscoped
`eslint-plugin-eslint-comments` was last published in 2022.

## TypeScript

```jsonc
{
  "strict": true,
  "noUncheckedIndexedAccess": true,
  "verbatimModuleSyntax": true,
  "noImplicitOverride": true
}
```

**Not `exactOptionalPropertyTypes`.** It fights generated Supabase types and
third-party declaration files constantly, and agents resolve that fight with
`as` casts — a net loss in type safety.

## Database assertions: pgTAP

Supabase's documented path: SQL tests in `supabase/tests/database/`, run by
`supabase test db` against a local database started by `supabase db start`.
These three are catalog queries, so they cover **tables that don't exist yet**
— which is why ADR-0002 rated them above the rest of the RLS suite.

Each description is written as a fix instruction, because pgTAP prints it on
failure:

1. **`anon has zero privileges on schema v2`** — *ADR-0002 revokes `anon`
   deliberately. Supabase's custom-schema doc prints `grant all ... to anon`;
   do not paste it. Policies are the second layer, not the first.*
   `schema_privs_are('v2', 'anon', array[]::text[])` covers the schema itself.
   Tables, sequences and routines need a catalog query too, because a stray
   object grant is harmless only for as long as schema `usage` stays revoked.
   Whoever writes it must decide how to treat `PUBLIC`: new functions carry
   `EXECUTE` to `PUBLIC` by default, which `anon` inherits, and ADR-0002 only
   revokes that on `v2.is_admin()`.
2. **`every table in schema v2 has row level security enabled`** — not stock
   pgTAP. Use Supabase's `tests.rls_enabled()` helper, or the equivalent
   one-line check over `pg_class.relrowsecurity`.
3. **`every auth.users row has a matching v2.members row`** — *the trigger on
   `auth.users` is the only door into `members`.*

The rest of ADR-0002's behavioural suite (per-role reads and writes, admission,
self-promotion) lives alongside these and is not restated here.

Because the database is local and ephemeral, **this suite needs no Supabase
credentials** in CI.

## The gate

```
npm run check  =  tsr generate  →  tsc --noEmit  →  eslint --max-warnings 0  →  vite build
```

- **Route generation comes first.** `tsc` cannot typecheck file routes against a
  route tree that hasn't been generated (vertical-slice finding).
- **Pre-commit** runs Husky + lint-staged: Prettier and ESLint over *staged
  files only*. It is kept fast on purpose — a slow hook gets bypassed with
  `--no-verify`, and agents reach for that flag readily.
- **CI** runs `npm run check`, and separately `supabase db start` →
  `supabase test db`. Nobody has Docker locally, so the database suite is
  CI-only by necessity.
- **The TanStack router packages are pinned to exact versions** — the router,
  its Vite plugin and its CLI. They version independently, and a mismatch
  breaks generation. Don't caret-range them back.

## Deliberately not mechanised

Each of these was considered and rejected as a check, because it is judgment
wearing a lint rule's clothes.

- **No raw colour values.** Tokens are the convention (see
  [the design-tokens prototype](https://github.com/a-movie-club/movie_club_v2/issues/11)).
  No single tool covers both `.tsx` source and CSS; `no-restricted-syntax` needs
  three selectors and still misses assembled strings; shadcn/ui's own repo
  enforces nothing here. Not worth the cost on a hobby site.
- **Nothing but `movies` carries a film's title, year or poster.** The invariant
  stands (ADR-0001, `CONTEXT.md`). A check would have to guess at column names —
  `film_name` and `poster_path` slip through, and a legitimate raw-title column
  in some future import log would trip it. It was known to be problematic, so
  it was let go.
- **Whether a mapping leaks database semantics.** The mechanical half is
  enforced above. "A `Pick<Row>` is fine when no naming or semantics leak" is a
  review question.
- **The promotion rule** — shared code only on the second consumer, never in
  anticipation. No linter can see anticipation. Its trigger is stated in the
  feature-boundary message.
- **The exact directory shape of a feature.** Folder conventions defines it.

## Considered options

- **Biome** (linter, or linter + formatter). With raw colours out of scope,
  Biome 2.5's `noRestrictedImports` and per-path overrides can express both
  boundary rules. Rejected on ecosystem: the TanStack Query and Router ESLint
  plugins have no Biome equivalent, and query keys are exactly what the
  data-layer contract is about. ESLint + Prettier is also the configuration
  every agent on every vendor already knows. *ESLint for lint, Biome for format*
  was a real option, rejected as two configs for a gain measured in milliseconds.
- **Type-aware linting** (`projectService`, `no-floating-promises` and friends).
  Lint time becomes roughly build time, and `recommendedTypeChecked` fires
  constantly on ordinary agent code. Rejected entirely rather than trimmed to a
  few rules — not worth it here.
- **`eslint-plugin-boundaries`.** An element-type/dependency-matrix model is
  heavier than two glob patterns, and its v7 has just deprecated its classic
  rules.
- **`--no-inline-config`.** Blocks every suppression in the repo, not just the
  architectural ones.
- **Linting SQL migration files.** Squawk has no custom rules; SQLFluff needs a
  shipped Python plugin. Nothing left in the inventory needs either — database
  facts are asserted against the database.
- **Vitest + node-postgres** for the database assertions. Agents write better
  Vitest than pgTAP, but these are catalog queries, SQL either way, and
  `supabase test db` supplies the plumbing for free.
- **Running the full gate pre-commit.** See The gate.

## Consequences

- **The gate is only a gate if `main` requires it.** Branch protection must mark
  both CI jobs as required checks; otherwise this ADR describes suggestions.
- **Features must export through `index.ts`.** An unexported symbol is
  unreachable from other features by construction.
- **Loosening a boundary rule is a config change in a reviewed PR**, never an
  inline comment. That is the intended friction.
- **Two invariants live in the database rather than here.** `rated_at`
  immutability and `updated_at` are triggers (ADR-0002); a lint rule cannot see
  an edit made through the dashboard or `psql`.
