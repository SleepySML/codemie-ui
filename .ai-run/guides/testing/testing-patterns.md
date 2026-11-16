# Testing Patterns — CodeMie UI

> Vitest 5 + React Testing Library. Two test projects: `unit` and `integration`.

---

## Overview

Vitest 5 runs two projects from `vite.config.ts`: `unit` (Valtio snapshot and the API client mocked) and `integration` (real stores, `fetch` mocked, full route config rendered). Both load `src/setupTests.tsx`; each adds its own setup file. Shared helpers for driving pages and PrimeReact widgets live in `src/test-utils/`.

| Part | What it holds | Start here |
|---|---|---|
| `vite.config.ts` | the two test projects, worker cap, fake-timer set, shuffle and coverage | `vite.config.ts::projects` |
| `src/setupTests.tsx` | global mocks for both projects, including the fetch registry | `src/setupTests.tsx::globalDefaults` |
| `src/setupTests.unit.ts` | unit-only mocks: api, storage, valtio | `src/setupTests.unit.ts::useSnapshot` |
| `src/setupTests.integration.ts` | integration-only setup: FeatureGuard bypass, testing-library timeouts | `src/setupTests.integration.ts::FeatureGuard` |
| `src/test-utils/` | `renderPage`, `mockAPI`, widget interaction helpers and recorded fixtures | `src/test-utils/integration.tsx::mockAPI` |
| `src/__tests__/` | app-level tests for `App` and the router | `src/__tests__/router.test.tsx::describe` |

## Test Organization

### File Location

Tests live in `__tests__/` folders co-located with the source:

```
src/components/Popup/
├── Popup.tsx
└── __tests__/
    └── Popup.test.tsx

src/pages/settings/administration/
├── ProvidersManagementPage.tsx
└── __tests__/
    ├── ProvidersManagementPage.test.tsx
    └── ProvidersManagementPage.integration.test.tsx
```

### Naming Convention

| File pattern | Project | What it runs |
|---|---|---|
| `*.test.tsx` | `unit` | Single-component isolation |
| `*.integration.test.tsx` | `integration` | Component → Store → API → UI chain |

Projects config: `test.projects` in `vite.config.ts` — two projects with separate setup files.

### Setup Files

- **Unit** — loads `setupTests.tsx` + `setupTests.unit.ts` (mocks `useSnapshot` and `@/utils/api`)
- **Integration** — loads `setupTests.tsx` + `setupTests.integration.ts`: real Valtio and real stores, `fetch` answered by the global mock in `setupTests.tsx` (`mockAPI` registrations, then `globalDefaults`), and `FeatureGuard` replaced by a pass-through (`src/setupTests.integration.ts::FeatureGuard`)

`SettingsLayout` and `useVueRouter` are mocked globally in `setupTests.tsx`; do not re-mock them in individual test files.

---

## Running Tests

```bash
npm run test:unit:slnt          # unit tests only
npm run test:integration:slnt   # integration tests only
npm test                   # all tests
npm test -- --watch        # watch mode
npm run test:coverage      # with coverage report
npm test -- ProvidersManagementPage.integration  # single file
```

---

## Unit Test Pattern (AAA)

Arrange / Act / Assert. Keep each `it` block focused on one behaviour.

```tsx
import { render, screen, fireEvent } from '@testing-library/react'
import { describe, it, expect, vi, afterEach } from 'vitest'
import { cleanup } from '@testing-library/react'
import MyComponent from '../MyComponent'

afterEach(cleanup)

describe('MyComponent', () => {
  it('renders label text', () => {
    render(<MyComponent label='Save' />)                    // Arrange + Act
    expect(screen.getByText('Save')).toBeInTheDocument()   // Assert
  })

  it('calls onClick handler', () => {
    const handleClick = vi.fn()                            // Arrange
    render(<MyComponent onClick={handleClick} />)
    fireEvent.click(screen.getByRole('button'))            // Act
    expect(handleClick).toHaveBeenCalledTimes(1)           // Assert
  })
})
```

Key rules:
- Always call `cleanup` in `afterEach` (prevents DOM leaks between tests)
- `vi.mock()` must be at **module level** — never inside `describe` or `it`
- One assertion focus per test; use multiple `expect` calls only when they test the same behaviour

---

## Mocking Patterns

### Mock `@/utils/api` in Unit Tests

`setupTests.unit.ts` mocks the module automatically. To control return values in a test:

```tsx
import api from '@/utils/api'
vi.mock('@/utils/api')

beforeEach(() => {
  vi.mocked(api).mockResolvedValue({
    json: () => Promise.resolve([{ id: '1', name: 'AWS' }]),
  } as Response)
})
```

Place `vi.mock('@/utils/api')` at the top of the file (module level), not inside `describe`.

### Spy on Functions

```tsx
import * as storeModule from '@/store/myStore'
vi.spyOn(storeModule, 'loadItems').mockResolvedValue(undefined)
```

### Mock a Module Partially

```tsx
vi.mock('@/hooks/useFeatureFlag', () => ({
  useFeatureFlag: vi.fn(() => true),
}))
```

### Dynamic Response (No `mockAPI` Factory)

`mockAPI` does not support factory functions. For responses computed from request body use:

```tsx
vi.spyOn(global, 'fetch').mockImplementationOnce(async (url, init) => {
  const body = JSON.parse((init as RequestInit).body as string)
  return new Response(JSON.stringify({ id: body.name }), { status: 201 })
})
```

---

## Integration Test Pattern

Integration tests use **real Valtio reactivity** — store state changes trigger actual re-renders.
`global.fetch` is stubbed once in `setupTests.tsx`; `mockAPI` registers per-test intercepts.

### Core Utilities (import from `@/test-utils/integration`)

| Export | Purpose |
|---|---|
| `renderPage(path)` | Renders full app at route via `createMemoryRouter` (`src/test-utils/integration.tsx::renderPage`) |
| `mockAPI(method, url, data, statusOrParams?)` | Registers fetch mock for one test (`src/test-utils/integration.tsx::mockAPI`) |
| `navigate` | `vi.fn()` spy replacing `useNavigate` globally |

### `mockAPI` Matching Rules

| 4th argument | Matching behaviour |
|---|---|
| omitted | Prefix match: `v1/items` matches `v1/items?page=0` but not `v1/items/1` |
| HTTP status number | Prefix match + error response |
| Params object | Exact match: path equals and all specified params present |

Last `mockAPI` call for the same key wins. Registry cleared automatically in `afterEach`.

### Basic Integration Test

```tsx
import { screen, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { describe, it, expect, beforeEach } from 'vitest'
import { mockAPI, navigate, renderPage } from '@/test-utils/integration'

describe('MyPage — Integration', () => {
  beforeEach(() => {
    mockAPI('GET', 'v1/items', [{ id: '1', name: 'Item One' }])
  })

  it('loads and displays items', async () => {
    renderPage('/settings/my-page')
    await waitFor(() =>
      expect(screen.getByText('Item One')).toBeInTheDocument()
    )
  })

  it('navigates after delete', async () => {
    mockAPI('DELETE', 'v1/items/1', null)
    renderPage('/settings/my-page')
    const row = screen.getByText('Item One').closest('tr')!
    await userEvent.click(within(row).getByTestId('actions-menu'))
    await userEvent.click(await screen.findByRole('button', { name: 'Delete' }))
    await userEvent.click(screen.getByTestId('confirm-button'))
    await waitFor(() =>
      expect(navigate).toHaveBeenCalledWith('/settings/items')
    )
  })
})
```

### Reactive Flow

1. Component calls store method (e.g., `store.loadItems()`)
2. Store makes fetch call → stub intercepts → returns mock data → updates proxy state
3. Real `useSnapshot` detects change → triggers React re-render
4. Test asserts on updated DOM

`useNavigate` is a spy — navigate calls are captured but do **not** update router history.
Assert with `expect(navigate).toHaveBeenCalledWith(path)` instead of checking the rendered page.

---

## Query Priority

Prefer role-based queries — they verify element type AND visible label simultaneously.

| Priority | Query | When to use | Example |
|---|---|---|---|
| 1st | `getByRole` | Buttons, headings, column headers | `getByRole('button', { name: 'Add' })` |
| 2nd | `findByRole` | Same, but async (waits for element) | `findByRole('button', { name: 'Delete' })` |
| 3rd | `getByPlaceholderText` | Form inputs | `getByPlaceholderText('Category Name')` |
| 4th | `getByLabelText` | Labelled inputs | `getByLabelText('Email address')` |
| 5th | `getByText` | Table cell content, messages | `getByText('Productivity')` |
| Last | `getByTestId` | When role queries produce multiple matches | `getByTestId('confirm-button')` |

### Scoping to a Table Row

```tsx
import { within } from '@testing-library/react'
const row = screen.getByText('Item Name').closest('tr')!
await userEvent.click(within(row).getByTestId('actions-menu'))
```

---

## DO / DON'T Table

| Category | DON'T | DO |
|---|---|---|
| **Mocking** | `vi.mock()` inside `describe` or `it` | Place at module top level |
| **Cleanup** | Omit `afterEach(cleanup)` | Always call `cleanup` in `afterEach` |
| **API client** | Import Axios or raw `fetch` | Mock `@/utils/api` for unit tests |
| **Integration mock** | Use `mockRequest` (removed) | Use `mockAPI` |
| **Dynamic mock** | Pass factory to `mockAPI` | Use `vi.spyOn(global, 'fetch')` |
| **renderPage** | Pass JSX component | Pass route path string: `'/settings/providers'` |
| **Navigation assert** | Check that target page rendered | `expect(navigate).toHaveBeenCalledWith(path)` |
| **Queries** | `getByTestId` for every element | Use `getByRole` / `findByRole` first |
| **Async** | Assert immediately after action | Wrap in `waitFor(...)` |
| **Sub-resource URLs** | `v1/assistants` for `v1/assistants/1` | Use exact sub-path to avoid prefix shadowing |
| **Valtio (integration)** | `vi.unmock('valtio')` | Real valtio is used automatically |
| **SettingsLayout** | Mock in each test file | Already mocked globally in `setupTests.tsx` |

---

## Checklist: New Integration Test

- [ ] File name: `ComponentName.integration.test.tsx` inside `__tests__/`
- [ ] Imports from `@/test-utils/integration`: `mockAPI`, `navigate`, `renderPage`
- [ ] `mockAPI(...)` called **before** `renderPage(...)` with URLs matching the store file
- [ ] `renderPage('/route/path')` — string, not JSX
- [ ] Prefer `getByRole` / `findByRole` over `getByText` for interactive elements
- [ ] All async assertions wrapped in `waitFor(...)`
- [ ] Tests assert user-visible behaviour, not internal store state

---

## Tests Failing in CI but Passing Locally

When the user reports this or pastes a CI `test:coverage` log:

1. Take the failing tests from the `FAIL` lines at the end of the log.
2. If the error is `Test timed out in 5000ms` (often followed on the retry by "Found multiple
   elements"), run only that file with coverage:
   `npx vitest run --coverage --project <unit|integration> <path>`.
3. Passes locally → it is coverage overhead in CI. Raise `CI_COVERAGE_TEST_TIMEOUT_MS` in
   `vite.config.ts`; do not rewrite the test.
4. Fails locally, fails on every CI run, or fails on an assertion rather than a timeout → the
   timeout is not the cause. Debug it; a common cause is a mock changed with `mockReturnValue`
   inside a test instead of reset in `beforeEach`.

The worker cap (`maxWorkers` in `vite.config.ts`), the unit-only coverage pass and the timeout
are explained in `README.md` § Test workers and timeouts. Do not remove the cap to speed up a
local run; pass `--maxWorkers` on the command line instead.

## Organization and naming

### Rules
- R1. Put a test in a `__tests__/` folder beside its source and name an integration test `*.integration.test.tsx` (the `integration` project includes only that pattern and the `unit` project excludes it). Evidence: `vite.config.ts::projects`

## Test shape

### Rules
- R1. Drive an integration test through `renderPage(path)`, which renders the real route config in a memory router and rethrows a render error instead of showing `ErrorPage`. Evidence: `src/test-utils/integration.tsx::renderPage`
- R2. Keep each test file independent of every other file (`sequence.shuffle.files` is on, so file order changes between runs). Evidence: `vite.config.ts::shuffle`

### Read first
| When you are | Exemplar | Why this one | Tests |
|---|---|---|---|
| driving a PrimeReact select, multi-select or autocomplete in a test | `src/test-utils/component-interactions/select.ts::openSelectDropdown` | opens the overlay the way a user does and returns the options | no covering test |

## Fixtures and factories

### Rules
- R1. Take marketplace-management payloads from `src/test-utils/fixtures/marketplace/` and refresh them with `npm run fixtures:refresh`; never edit a fixture JSON or invent a row (the files are recorded backend responses). Evidence: `src/test-utils/fixtures/marketplace/index.ts::overviewFixture`

## Mocking

### Rules
- R1. Do not re-mock in a test file what `src/setupTests.tsx` mocks for every project: `fetch` defaults, `SettingsLayout`, `useVueRouter`, `@/utils/toaster` and `react-router`'s navigation. Evidence: `src/setupTests.tsx::globalDefaults`
- R2. In a unit test, treat `useSnapshot` as returning the raw store and `@/utils/api` methods as `vi.fn()` resolving empty JSON; set a response per test on the mocked method (`setupTests.unit.ts` installs both). Evidence: `src/setupTests.unit.ts::useSnapshot`

## Database testing

### Rules
- R1. Test browser persistence, the app's only store of record on the client, against the mocks the setup installs: `@/utils/storage` is a `vi.fn()` object in unit tests, and `localStorage` is the in-memory `localStorageMock` in both projects. Evidence: `src/setupTests.tsx::localStorageMock`

## API testing

### Rules
- R1. Register each backend response an integration test needs with `mockAPI(method, url, data, statusOrParams)` before rendering (registrations are keyed by `METHOD:url`; a number fourth argument is the status, an object is the expected query params). Evidence: `src/test-utils/integration.tsx::mockAPI`

## Coverage

### Rules
- R1. Read coverage from the unit pass: `npm run test:coverage` collects it for `--project unit` only, and `coverage.include` reports every untested `src/**` file at 0 %. Evidence: `vite.config.ts::coverage`

## Examples

### Registering a backend response

Source: `src/test-utils/integration.tsx::mockAPI`

```ts
export function mockAPI(
  method: Method,
  url: string,
  data: unknown,
  params?: Record<string, unknown>
): void
```

### Driving a select in a test

Source: `src/test-utils/component-interactions/select.ts::openSelectDropdown`

```ts
export async function openSelectDropdown(
  element: HTMLElement,
  user?: ReturnType<typeof userEvent.setup>
): Promise<void> {
  const userInstance = user ?? userEvent.setup()

  await userInstance.click(element)

  // Wait for panel to appear
  await waitFor(() => {
    const panel = document.querySelector('.p-dropdown-panel')
    if (!panel) throw new Error('Select panel did not open')
  })
}
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A `requestAnimationFrame` or `performance` stub stops working under `vi.useFakeTimers()` | Vitest 5 fakes every timer API by default | the config pins `fakeTimers.toFake` to the timers and `Date`; widen it per test with `vi.useFakeTimers({ toFake: [...] })` | `vite.config.ts::fakeTimers` |
| A test depends on the order of `afterEach` hooks | `sequence.hooks` is pinned to `parallel` for the integration setup | do not rely on hook order between the setup files and the test file | `vite.config.ts::sequence` |
| A page renders `ErrorPage` content instead of failing with the real error | React Router turns render exceptions into its error element | render through `renderPage`, whose `onCaughtError` rethrows | `src/test-utils/integration.tsx::renderPage` |
| jest-dom matchers missing on `Assertion` in type-check | Vitest 5's `Assertion<R, T>` does not merge jest-dom's augmentation | keep the `Matchers` declaration file | `src/types/jest-dom-vitest.d.ts::Matchers` |
| A test passes alone but a later, unrelated test crashes with `localStorage is not defined` | the page under test keeps a self-rescheduling poll (`DataSourcesPage` refreshes every `REFRESH_TIMEOUT`) whose timer fires after that test's jsdom is gone | track timers created during the test and clear them in `afterEach`; faking timers also fakes `waitFor`'s own timers | `src/pages/dataSources/DataSourcesPage.tsx::REFRESH_TIMEOUT` |
| `proxyState is not iterable` from the settings layout | `SettingsLayout` reads stores that are not proxies in unit tests | rely on the global mock in `setupTests.tsx`; do not re-mock it | `src/setupTests.tsx::SettingsLayout` |
| The page after a navigation never renders | `useNavigate` is a spy in both projects | assert the `navigate` call instead of the next page | `src/test-utils/integration.tsx::navigate` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Write an integration test for a page | `mockAPI` each request, then `renderPage('/path')` and query by role | `src/test-utils/integration.tsx::renderPage` |
| Assert navigation | `expect(navigate).toHaveBeenCalledWith(path)` with the spy from `@/test-utils/integration` | `src/test-utils/integration.tsx::navigate` |
| Understand worker and timeout settings | read the README section before changing them | `README.md` § Test workers and timeouts |