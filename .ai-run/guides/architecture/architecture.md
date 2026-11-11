# Architecture — codemie-ui

## Overview

CodeMie UI is a React 19 single-page app built by Vite. `src/main.tsx` mounts the browser router from `src/router.tsx` inside the toast announcer provider; the root route renders `App` (navigation, app-level popups, the initial data fetch) with `ErrorPage` as its error boundary, and pages read Valtio stores that call the backend through the single `api` client. A second Vite entry (`VITE_ENTRY=keycloakify`) builds the Keycloak login theme from `src/authentication/keycloak-theme/`.

| Part | What it holds | Start here |
|---|---|---|
| `src/pages/` | route components, one folder per feature | `src/router.tsx::routes` |
| `src/components/` | shared UI used by several features | `src/components/Popup/Popup.tsx::PopupProps` |
| `src/store/` | Valtio stores that own backend calls and shared state | `src/store/skills.ts::skillsStore` |
| `src/utils/` | the API client and framework-free helpers | `src/utils/api.ts::API` |
| `src/hooks/` | shared React hooks, app-level hooks under appLevel/ | `src/hooks/useFeatureFlags.ts::useFeatureFlag` |
| `src/types/` | entity types and generated API types | `src/types/common.ts::Pagination` |
| `src/constants/` | route ids, feature flag keys and other named values | `src/constants/featureFlags.ts::FEATURE_FLAGS` |
| `src/configs/` | onboarding flow definitions and release-notes data | `src/configs/onboarding/index.ts::navigationIntroductionFlow` |
| `src/authentication/` | local sign-in and sign-up pages and the Keycloak login theme | `src/authentication/local/SignInPage.tsx::SignInPage` |
| `src/a2ui/` | renderer for agent-authored A2UI surfaces in chat | `src/a2ui/registry.tsx::OWN_COMPONENTS` |
| `src/styles/` | PrimeReact pass-through presets | `src/constants/theme.ts::primeReactPtOptions` |
| `src/assets/` | global stylesheets, fonts, icons and images | `src/assets/stylesheets/main.scss::font-family-body-sans` |
| `src/test-utils/` | integration-test helpers and fixtures | `src/test-utils/integration.tsx::renderPage` |

## Key abstractions

| Abstraction | What it is | Where |
|---|---|---|
| API client | singleton over fetch: query params, reserved headers, manual redirects, error toasts, streaming and multipart | `src/utils/api.ts::API` |
| Valtio store | `proxy<Type>({...})` object holding state and the async methods that call `api` | `src/store/skills.ts::skillsStore` |
| Route guards | elements that throw a 404 for a disabled feature flag or for standalone mode | `src/components/FeatureGuard.tsx::FeatureGuard` |
| Router instance | the created router, readable without importing every page | `src/routerInstance.ts::getRouter` |
| A2UI registry | which implementation draws each A2UI catalog component | `src/a2ui/registry.tsx::OWN_COMPONENTS` |

## Layer Responsibilities

| Layer | Responsibility | Key paths |
|---|---|---|
| **Pages** | Route-level components, compose feature UI | `src/pages/<feature>/` |
| **Components** | Reusable UI primitives and shared UI | `src/components/` |
| **Store** | Global state, API calls, business logic, error/loading | `src/store/*.ts` |
| **Integration** | HTTP, auth headers, redirect handling, error parsing | `src/utils/api.ts` |
| **Hooks** | Reusable React logic; no direct API calls | `src/hooks/` |
| **Utils** | Pure functions, data transforms, URL builders | `src/utils/` |
| **Types** | Shared TypeScript interfaces and enums | `src/types/` |
| **Constants** | Named values; no magic strings/numbers | `src/constants/` |
| **Config** | Onboarding flow definitions and release-notes data; feature flags live in `src/constants/featureFlags.ts` | `src/configs/onboarding/index.ts::navigationIntroductionFlow` |
| **Authentication** | Local auth pages + Keycloak theme entry point | `src/authentication/` |

### What each layer must NOT do

| Layer | Forbidden |
|---|---|
| Page / Component | Call `api.*` directly; own business logic |
| Store | Render JSX; call `useNavigate`; import React components |
| Integration | Manage state; transform domain objects |
| Hook | Call `api.*` directly; import Valtio stores from outside |
| Utils | Mutate global state; trigger side effects (one deliberate exception, below) |

**Utils exception — sign-in window helpers.** `src/utils/openSignInWindow.ts` and
`src/utils/watchSignInWindow.ts` are deliberate side-effecting helpers: they open a browser window,
poll the backend and send a diagnostics beacon (`src/utils/mcpAuthDiagnostics.ts`, the beacon they
share with the auth callback listener hook). They sit in `src/utils/` so the hooks and the chat store
use one copy of the window logic. They must still not import hooks, stores or components. New side
effects do not belong in Utils on this precedent.

---

## Data Flow (Component → Store → API)

The canonical pattern is strictly three steps. Components observe the store snapshot; they
never touch the API client directly.

Step 1 — Page subscribes and triggers:
  `src/pages/assistants/AssistantsListPage.tsx::AssistantsListPage` (`useSnapshot(assistantsStore)`)

Step 2 — Store owns the API call and state mutation:
  `src/store/assistants.ts::assistantsStore` (proxy definition), `src/store/assistants.ts::assistantsStore.indexAssistants` (method calling `api.get`)

Step 3 — Response is parsed with `.json()`, never `.data`:
  `src/store/assistants.ts::assistantsStore.indexAssistants` (`response.json()`)

A store that exposes loading or error state sets it on its own fields around the call and resets it in `finally` (`src/store/skills.ts::skillsStore.getSkillById`). Not every store does: `src/store/assistants.ts::assistantsStore.indexAssistants` keeps no loading field and leaves the failure toast to `api`.

Session expiry handling lives in `src/store/user.ts::redirectHandler`: `api.redirectHandler` sets
`userStore.isSessionExpired = true`, which `SessionExpiredPopup` observes.

---

## Authentication Architecture

Two independent auth paths share zero code:

| Path | Entry | Use case |
|---|---|---|
| Local auth | `src/authentication/local/SignInPage`, `SignUpPage` | Dev/standalone deployments |
| Keycloak SSO | `src/authentication/keycloak-theme/keycloakify.tsx` (separate Vite entry) | Production; EPAM SSO via Azure Entra ID |

For Keycloak builds, set `VITE_ENTRY=keycloakify`; `vite.config.ts::isKeycloak` then builds from
`index-keycloak.html` (the `rename-keycloak-html` plugin renames it to `index.html` in `dist/`), which renders only
the `KcPage` component (`src/authentication/keycloak-theme/login/KcPage.tsx::KcPage`) — no router, no app shell.

The API client adds `credentials: 'include'` only when `getIsLocalAuth()` reports the `local` identity provider (`src/utils/utils.ts::getIsLocalAuth`); every request uses `redirect: 'manual'`, and an opaque redirect from an expired single sign-on session goes to `api.redirectHandler` (`src/utils/api.ts::API.makeRequest`).

User state after login: `userStore.user.isAuthenticated` and `userStore.user.isAdmin`
(`src/types/entity/user.ts::User`), populated in `src/store/user.ts::userStore`.

---

## Module Federation

Config: `vite.config.ts::federation`.

- Host name: `codemie-ui-host`.
- Declared remote: `angular-upgrade-app` (value populated at runtime from `/applications` API).
- Remote apps are loaded in `src/pages/applications/ApplicationFederationPage.tsx` (route `applications/:slug`).
- Iframe fallback: `src/pages/applications/ApplicationIframePage.tsx` (route `applications/iframe/:slug`).
- React, react-dom, and react-router are in the shared list to prevent version conflicts.

To register a new remote: add an entry to the `remotes` object in `vite.config.ts`, then
create a page under `src/pages/applications/` that dynamically imports from it.

---

## Design Patterns

| Pattern | Description | Key reference |
|---|---|---|
| Valtio proxy store | `proxy<StoreType>(…)` — auto-reactive global state | `src/store/assistants.ts::assistantsStore` |
| `useSnapshot` read binding | Creates a frozen read-only snapshot inside components | `src/App.tsx::useSnapshot` |
| Typed store interface | Most stores pass a named type to `proxy<…>` (`*StoreType`, `*Store` or `*State`); `dataSourceStore` and `llmModelsStore` call `proxy({…})` untyped — do not copy them | `src/store/assistants.ts::AssistantsStoreType` |
| Custom `API` class | `get/post/patch/put/delete/stream/postMultipart` methods | `src/utils/api.ts::API` |
| `navigateBack` utility | History-first back with path-matching fallback | `src/utils/helpers.ts::navigateBack` |
| Feature `goBack*` wrappers | Per-feature `navigateBack` call with allowed route IDs | `src/pages/assistants/utils/goBackAssistants.ts::goBackAssistants` |
| Route ID constants | Exported string constants consumed by both router and `navigateBack` | `src/constants/routes.ts` |
| `useHistoryStack` | Valtio proxy tracking navigation history stack | `src/hooks/appLevel/useHistoryStack.tsx::history` |
| `FeatureGuard` on route elements | Wraps a route element so it renders only when its feature flags are on; the router itself spreads every route array unconditionally | `src/components/FeatureGuard.tsx::FeatureGuard` |
| `useFeatureFlag(name)` hook | Runtime feature flag from API; non-React context uses `src/utils/featureFlags` | `src/hooks/useFeatureFlags.ts::useFeatureFlag` |
| Component-scoped files | Sub-components, utils, and tests collocated inside feature folder | `src/pages/workflows/editor/` |
| Workflow editor extraction | Complex canvas logic extracted to `utils/` to keep components slim | `src/pages/workflows/editor/utils/` |
| `ErrorBoundary` on root route | `ErrorPage` as `ErrorBoundary` prop on the root route object | `src/router.tsx::routes` |

---

## Feature Folder Convention

Assistants, skills and workflows follow the full layout below, `goBack<Feature>` included (`src/pages/workflows/utils/goBackWorkflows.ts::goBackWorkflows`); other areas such as `src/pages/dataSources/DataSourcesPage.tsx::DataSourcesPage` and `src/pages/integrations/IntegrationsPage.tsx::IntegrationsPage` keep the page files and `__tests__/` without the `utils/` navigation helpers.

```
src/pages/<feature>/
  <Feature>ListPage.tsx          route component — list view
  <Feature>DetailsPage.tsx       route component — detail view
  New<Feature>Page.tsx           route component — create form
  Edit<Feature>Page.tsx          route component — edit form
  components/                    feature-private components
    <Feature>Form/
    <Feature>List/
  utils/
    goBack<Feature>.ts           wraps navigateBack with allowed route IDs
    get<Feature>Link.ts          URL builder for this feature
  __tests__/                     co-located tests
```

Route array for the feature lives in `src/router.tsx` as `const <feature>Routes: RouteObject[]`
and is spread into the root children array (`src/router.tsx::routes`).

---

## Extension Points (How to Add a New Feature)

1. **Types** — `src/types/entity/<feature>.ts` — define request/response interfaces.
2. **Constants** — add route IDs to `src/constants/routes.ts`; other magic values to `src/constants/<feature>.ts`.
3. **Store** — `src/store/<feature>.ts` — export `proxy<FeatureStoreType>(…)`. Add to `src/store/index.ts`.
4. **Pages** — follow the feature folder convention above.
5. **Routes** — add `const <feature>Routes: RouteObject[]` in `src/router.tsx`; spread into root children.
6. **Navigation** — add entry to `src/components/Navigation/Navigation.tsx`.
7. **Feature gate** — the router spreads every route array unconditionally; wrap a gated route element in `FeatureGuard` (see `src/router.tsx::dataSourcesSurfaceFlags`).
8. **Tests** — co-locate in `__tests__/` inside the feature folder.

---

## Cross-Cutting Concerns

| Concern | Where it lives | Notes |
|---|---|---|
| Error display | `toaster.error()` in store catch blocks | `src/utils/toaster.ts` |
| Session expiry | `api.redirectHandler` → `userStore.isSessionExpired` | `src/store/user.ts::redirectHandler` |
| Unsaved changes | `UnsavedChangesProvider` + `useUnsavedChangesWarning` hook | `src/App.tsx::UnsavedChangesProvider` |
| Initial data fetch | `useInitialDataFetch` hook mounted in `App.tsx` | `src/hooks/appLevel/useInitialDataFetch.tsx` |
| Theme toggling | `usePrismThemeToggle` and `themeService` | `src/utils/themeService.ts` |

---

## Boundaries (DO/DON'T)

| DO | DON'T |
|---|---|
| Call `api.*` inside store methods only | Call `api.*` in components, hooks, or utils |
| Parse responses with `.json()` | Access `.data` (Axios pattern); `axios` is listed in `package.json` but no file under `src/` imports it |
| Use Tailwind classes for all styling | Write custom CSS or inline styles |
| Use `Popup` component for all dialogs | Import `Dialog` from PrimeReact directly |
| Read global state via `useSnapshot(store)` | Directly read or subscribe to the mutable proxy in JSX |
| Keep components under 300 lines; extract sub-components | Build monolithic page components |
| Use `??` for defaults | Use `\|\|` which coerces empty strings and zero |
| Use single quotes for strings | Use double quotes (ESLint enforces single quotes) |
| Store route IDs as constants in `src/constants/routes.ts` | Hardcode path strings at call sites |
| Use feature-scoped `goBack*` utilities on create/edit pages | Call `navigate(-1)` directly — breaks on refresh |
| Put business rules in store methods | Put domain logic in components or utils |
| Set `loading` and `error` fields on the store | Manage loading state in component `useState` |

## Structure and boundaries

### Rules
- R1. Import the router through `getRouter()` from `src/routerInstance.ts` in a util, hook or store that only reads router state or navigates (`@/router` imports every page, which pulls the whole app into that module and its tests). Evidence: `src/routerInstance.ts::getRouter`
- R2. Mount an app-wide component that must also serve `/auth/sign-in` and `/auth/sign-up` above `RouterProvider` in `src/main.tsx`, not in `App.tsx` (those routes are siblings of the root route and render without `App`). Evidence: `src/main.tsx::AnnouncerHostProvider`

## Patterns in use

### Rules
- R1. Gate a route on a feature flag by wrapping its element in `FeatureGuard` (the router spreads every route array unconditionally; `FeatureGuard` throws the 404 that the root `ErrorPage` boundary renders). Evidence: `src/components/FeatureGuard.tsx::FeatureGuard`
- R2. Put a route that needs the CodeMie core backend among the `CoreGuard` children of the root route (`CoreGuard` throws a 404 when `features:core` is disabled, the standalone mode). Evidence: `src/components/CoreGuard.tsx::CoreGuard`

## Request flow

### Rules
- R1. Pass `{ skipErrorHandling: true }` when a store shows its own failure message (without it `api` already toasts the parsed error body, so the store's `toaster.error` adds a second toast). Evidence: `src/utils/api.ts::API.makeRequest`
- R2. Never pass `Authorization`, `Cookie`, `Host` or `user-id` in a request's headers (`api` throws for them; identity comes from the session cookie or `authHeaders()`). Evidence: `src/utils/api.ts::RESERVED_REQUEST_HEADERS`

## Extension points

### Rules
- R1. Register an A2UI component the project draws itself in `OWN_COMPONENTS` through a factory from `src/a2ui/factory.tsx`, with a comment saying why the SDK's version is not used (the registry is the one record of what the project owns). Evidence: `src/a2ui/registry.tsx::OWN_COMPONENTS`
- R2. After changing the supported A2UI catalog, run `npm run a2ui:manifest`, commit `src/a2ui/a2ui-manifest.json` and copy it to the backend as the script prints (the backend contract test reads that snapshot as a plain file). Evidence: `scripts/generate-a2ui-manifest.ts::outputPath`

## Configuration

### Rules
- R1. Read a deployment setting as `window._env_` first and `import.meta.env` second, as `api` does for `VITE_API_URL` (both are public in the browser build, so neither may hold a secret). Evidence: `src/utils/api.ts::API.BASE_URL`
- R2. Check a feature flag in a component with `useFeatureFlag` and act on a disabled flag only once its second value, `isLoaded`, is true (before the config fetch every flag reads as disabled); outside React use `isFeatureEnabled`, which returns false until then. Evidence: `src/hooks/useFeatureFlags.ts::useFeatureFlag`

## Examples

### A store method that owns a backend call

Source: `src/store/skills.ts::skillsStore.getSkillById`

```ts
  getSkillById: (id: string) => Promise<Skill>
```

### A route guard

Source: `src/components/CoreGuard.tsx::CoreGuard`

```ts
export const CoreGuard: FC<{ children: ReactNode }> = ({ children }) => {
  const { configs } = useSnapshot(appInfoStore)
  if (!isCoreAvailable(configs)) throw notFoundError()
  return children
}
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A request throws `The "authorization" request header is managed by the API client` | the caller passed a reserved identity header | drop the header; identity comes from the session cookie or `authHeaders()` | `src/utils/api.ts::RESERVED_REQUEST_HEADERS` |
| A signed-in user is sent to sign-in after a permission error | under local auth a 401 is treated as an expired session unless its body is an auth gate, a broker login or the backend's `Access denied` message | keep the backend message equal to `ACCESS_DENIED_MESSAGE` | `src/utils/api.ts::ACCESS_DENIED_MESSAGE` |
| Two error toasts for one failed request | `api` toasts the error body and the store calls `toaster.error` as well | pass `skipErrorHandling: true` where the store owns the message | `src/utils/api.ts::API.makeRequest` |
| A util's unit test loads every page of the app | the util imports `@/router` | call `getRouter()` from `src/routerInstance.ts` | `src/routerInstance.ts::getRouter` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Add a page | add a `RouteObject` to the feature's route array in `src/router.tsx`; wrap a flagged element in `FeatureGuard` | `src/router.tsx::dataSourcesSurfaceFlags` |
| Add a feature flag | add the key to `FEATURE_FLAGS`, then a `use<Name>Enabled` hook if components read it | `src/constants/featureFlags.ts::FEATURE_FLAGS` |
| Add a backend call | add an async method to the feature's store that calls `api` and parses `response.json()` | `src/store/skills.ts::skillsStore` |
| Find the project layout | read the structure section of the README | `README.md` § Project Structure |