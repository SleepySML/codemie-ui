# Constants Usage

> Extracting and organizing magic values in CodeMie UI

---

## Overview

`src/constants/` holds named values per domain: route ids, feature-flag keys, statuses, limits. `index.ts` defines the shared values and enums and re-exports `INDEX_TYPES` and `FEATURE_FLAGS`; every other constant is imported from its own file.

| Part | What it holds | Start here |
|---|---|---|
| `src/constants/index.ts` | shared values and enums | `src/constants/index.ts::AssistantTab` |
| `src/constants/routes.ts` | route ids | `src/constants/routes.ts::ASSISTANT_DETAILS` |
| `src/constants/featureFlags.ts` | feature flag keys | `src/constants/featureFlags.ts::FEATURE_FLAGS` |
| `src/constants/workflows.ts` | workflow statuses and formats | `src/constants/workflows.ts::WORKFLOW_STATUSES` |

## When to Extract

| Extract when... | Skip when... |
|-----------------|--------------|
| Value used in 2+ places | Used exactly once |
| Value has business meaning | Obviously `0`, `1`, `true`, `false` |
| Value might change | Component-specific one-off UI value |
| Value represents configuration | Would be over-abstraction |

---

## Constant Types

### Enums — related string/number values

Use TypeScript `enum` for a closed set of related values that map to strings:

```typescript
// src/constants/index.ts
export enum AssistantTab {
  ALL = 'all',
  USER = 'user',
  TEMPLATES = 'templates',
  MARKETPLACE = 'marketplace',
  FAVORITES = 'favorites',
}
```

Use enum members as values directly — no raw strings in JSX or logic.

### Const objects — configuration groups

Use `as const` to lock down the type to literal values:

```typescript
// src/constants/index.ts
export const ENV = {
  LOCAL: 'local',
  PROD: 'prod',
} as const
```

`as const` narrows the type from `string` to the exact literal value (e.g. `'local'`), enabling exhaustive checks.

### Endpoints

There is no endpoint-constants module: a store builds its URL inline next to the call (`src/store/skills.ts::skillsStore.getSkillById` uses a `v1/skills/${id}` template literal).

See also: `.ai-run/guides/patterns/accessibility-patterns.md` § Cross-component ARIA id sharing — `src/utils/ariaIds.ts` is a constants-adjacent module of narrowly-named id-builder functions, not a generic keyed lookup.

### Status label/color maps

Pair a status map with the list of terminal statuses instead of `if/else` chains, as `src/constants/workflows.ts::WORKFLOW_STATUSES` and `WORKFLOW_FINAL_STATUSES` do.

---

## File Organization

`src/constants/` holds one file per domain (`assistants.ts`, `dataSources.ts`, `routes.ts`, `featureFlags.ts`, `workflows.ts` and others). `src/constants/index.ts` defines the shared values itself and re-exports only `INDEX_TYPES` and `FEATURE_FLAGS`; import a domain constant from its own file, as most callers do (`import { ASSISTANT_DETAILS } from '@/constants/routes'`).

---

## Naming Conventions

| Type | Convention | Example |
|------|-----------|---------|
| Enum | PascalCase | `AssistantTab` |
| Enum member | UPPER_SNAKE_CASE | `AssistantTab.ALL` |
| Const object | UPPER_SNAKE_CASE | `ENV` |
| Helper function in constants | camelCase | `isLiveMarketplaceStatus` |

**Descriptive names — no abbreviations:**

| Bad | Good |
|-----|------|
| `MAX_LEN` | `MAX_ASSISTANT_NAME_LENGTH` |

---

## Common Constant Groups

### Validation rules — used in Yup schemas

```typescript
// src/constants/validationRules.ts
export const VALIDATION_RULES = {
  ASSISTANT_NAME: { MIN_LENGTH: 3, MAX_LENGTH: 100 },
  SYSTEM_PROMPT: { MIN_LENGTH: 10, MAX_LENGTH: 50000 },
} as const
```

Schema usage: `yup.string().min(VALIDATION_RULES.ASSISTANT_NAME.MIN_LENGTH)`

### Timeouts — debounce, toast, polling

```typescript
// src/constants/timeouts.ts
export const TIMEOUTS = {
  DEBOUNCE_INPUT: 300,
  TOAST_DURATION: 3000,
  API_REQUEST: 30000,
  POLLING_INTERVAL: 5000,
} as const
```

### Feature flags

```typescript
// src/constants/featureFlags.ts
export const FEATURE_FLAGS = {
  ENABLE_MCP_SERVERS: true,
  ENABLE_AI_REFINE: false,
} as const
```

Guard features with `if (FEATURE_FLAGS.ENABLE_MCP_SERVERS) { ... }` — never hardcode `true`/`false` inline.

---

## Type Safety Rules

| Pattern | Why |
|---------|-----|
| `as const` on all object constants | Preserves literal types, prevents widening |
| `enum` for closed string sets | Exhaustive switch, no invalid values |
| Type guard functions alongside enums | Narrows type at call site |

Type guard example:
```typescript
export const isLocalAssistant = (type: string): type is AssistantType.LOCAL =>
  type === AssistantType.LOCAL
```

Derive union types from const objects:
```typescript
type StatusColor = typeof STATUS_COLORS[keyof typeof STATUS_COLORS]
// resolves to 'green' | 'gray', not string
```

---

## Integration with Other Layers

Route ids from `src/constants/routes.ts` name the route objects in `src/router.tsx` and the allowed targets of `navigateBack` (`src/utils/helpers.ts::navigateBack`); feature flag keys from `src/constants/featureFlags.ts` feed `FeatureGuard` and `useFeatureFlag`.

---

## DO / DON'T

| DON'T | DO |
|-------|----|
| `status === 'Succeeded'` | `status === WORKFLOW_STATUSES.SUCCEEDED` |
| Scatter magic numbers in components | Extract to `src/constants/` file |
| Export constants from component files | Put in `src/constants/` and barrel-export |

---

## JSDoc for Non-Obvious Constants

Add a JSDoc comment when the value's rationale is not self-evident:

```typescript
/**
 * Polling interval for workflow execution status.
 * Balances responsiveness with server load — do not lower below 1000.
 */
export const WORKFLOW_POLL_INTERVAL = 2000
```

---

## Examples

### A status map with its terminal set

Source: `src/constants/workflows.ts::WORKFLOW_FINAL_STATUSES`

```ts
export const WORKFLOW_FINAL_STATUSES = [
  WORKFLOW_STATUSES.SUCCEEDED,
  WORKFLOW_STATUSES.ABORTED,
  WORKFLOW_STATUSES.FAILED,
  WORKFLOW_STATUSES.INTERRUPTED,
  WORKFLOW_STATUSES.AUTHENTICATION_REQUIRED,
]
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A constant imported from `@/constants` is undefined | the barrel re-exports only `INDEX_TYPES` and `FEATURE_FLAGS` from other files | import it from its own file | `src/constants/index.ts::INDEX_TYPES` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Add a route id | export a string constant from `src/constants/routes.ts` | `src/constants/routes.ts::ASSISTANT_DETAILS` |