# Routing Patterns — codemie-ui

## Overview

All routes are declared in `src/router.tsx`: one `RouteObject[]` per feature, spread into the root route whose element is `App`, plus the sign-in, sign-up and login-success routes beside it. `createBrowserRouter` uses `import.meta.env.BASE_URL` as basename; old `#/` URLs are rewritten by `redirectHashRoutes` before the router starts.

| Part | What it holds | Start here |
|---|---|---|
| `src/pages/analytics/` | usage and spending analytics dashboards | `src/pages/analytics/AnalyticsPage.tsx::AnalyticsPage` |
| `src/pages/applications/` | federated, iframe and hosted applications | `src/pages/applications/ApplicationsPage.tsx::ApplicationsPage` |
| `src/pages/assistants/` | assistant list, details, create and edit | `src/pages/assistants/AssistantsListPage.tsx::AssistantsListPage` |
| `src/pages/chat/` | chat page, history and shared chats | `src/pages/chat/ChatPage.tsx::ChatPage` |
| `src/pages/dataSources/` | data source list, details, create and edit | `src/pages/dataSources/DataSourcesPage.tsx::DataSourcesPage` |
| `src/pages/error/` | the root route's error element | `src/pages/error/ErrorPage.tsx::ErrorPage` |
| `src/pages/favorites/` | favorites by entity type | `src/pages/favorites/FavoritesPage.tsx::FavoritesPage` |
| `src/pages/help/` | help page and models catalog | `src/pages/help/HelpPage.tsx::HelpPage` |
| `src/pages/integrations/` | user and project integrations | `src/pages/integrations/IntegrationsPage.tsx::IntegrationsPage` |
| `src/pages/katas/` | katas list and editor | `src/pages/katas/KatasPage.tsx::KatasPage` |
| `src/pages/login-success/` | landing page after an external login | `src/pages/login-success/LoginSuccessPage.tsx::LoginSuccessPage` |
| `src/pages/releaseNotes/` | release notes page | `src/pages/releaseNotes/ReleaseNotesPage.tsx::ReleaseNotesPage` |
| `src/pages/schedulers/` | schedulers and their run history | `src/pages/schedulers/SchedulersPage.tsx::SchedulersPage` |
| `src/pages/settings/` | profile, administration and settings tabs | `src/pages/settings/AdministrationPage.tsx::AdministrationPage` |
| `src/pages/skills/` | skills list, details, create and edit | `src/pages/skills/SkillsListPage.tsx::SkillsListPage` |
| `src/pages/terms/` | terms and conditions page | `src/pages/terms/TermsAndConditionsPage.tsx::TermsAndConditionsPage` |
| `src/pages/workflows/` | workflow list, editor, details and executions | `src/pages/workflows/WorkflowsListPage.tsx::WorkflowsListPage` |

## Route Structure

### File layout

`src/router.tsx` declares one `RouteObject[]` array per domain feature, then spreads them
all into a single root route object (`src/router.tsx::routes`).

```
import { createBrowserRouter, Navigate, Outlet, redirect, RouteObject } from 'react-router'
const chatRoutes: RouteObject[] = […]
const assistantRoutes: RouteObject[] = […]
const skillRoutes: RouteObject[] = […]
const integrationRoutes: RouteObject[] = […]
const dataSourceRoutes: RouteObject[] = […]
const schedulerRoutes: RouteObject[] = […]
const katasRoutes: RouteObject[] = […]
const workflowRoutes: RouteObject[] = […]
const applicationRoutes: RouteObject[] = […]
const settingsRoutes: RouteObject[] = […]
const awsSettingsRoutes: RouteObject[] = […]
const analyticsRoutes: RouteObject[] = […]
const otherRoutes: RouteObject[] = […]
const favoritesRoutes: RouteObject[] = […]
export const routes: RouteObject[] = [{ id: 'root', path: '/', Component: App, ErrorBoundary: ErrorPage, … }, …]
export const router = createBrowserRouter(routes, { basename: import.meta.env.BASE_URL })
```

### Root route

```
id: 'root', path: '/', Component: App, ErrorBoundary: ErrorPage
  children: chatRoutes, assistantRoutes, skillRoutes, …
```

`App` renders `<Outlet />` which is replaced by the matched child route component.

### Auth routes (outside root, no nav shell)

```
{ id: 'login-success', path: '/login-success', Component: LoginSuccessPage }
{ id: 'sign-in', path: '/auth/sign-in', Component: SignInPage }
{ id: 'sign-up', path: '/auth/sign-up', Component: SignUpPage }
```

These are siblings of the root route — they render without the main navigation layout.

### Loader-based redirect

`workflowRoutes` uses a loader to redirect `/workflows` to the default sub-route:
```
src/router.tsx::workflowRoutes  loader: () => redirect('/workflows/my')
```
`applicationRoutes` redirects legacy paths (`/apps/:id/manage`, `/apps/:id/standalone`, `/applications`) with loaders too (`src/router.tsx::applicationRoutes`); the settings tree redirects with a `<Navigate to=… replace />` element instead (`src/router.tsx::settingsRoutes`).

---

## Navigation

### Declarative (Link component)

Import from `'react-router'`. Used for in-template links.

```
import { Link } from 'react-router'
<Link to='/assistants'>All Assistants</Link>
<Link to={`/assistants/${id}`}>View</Link>
```

### Programmatic (useNavigate)

```
import { useNavigate } from 'react-router'
// src/pages/settings/administration/MCPManagementPage.tsx::MCPManagementPage
const navigate = useNavigate()
navigate('/assistants')           // push
navigate(-1)                      // go back — avoid on create/edit pages (see goBack section)
navigate('/path', { state: { from: 'source' } })   // with state
```

### Navigation with state

Pass state on navigate; read it with `useLocation`:
```
navigate('/assistants/new', { state: { from: 'marketplace' } })

import { useLocation } from 'react-router'
const location = useLocation()
const from = location.state?.from
```

### Navigating after a store action

Store methods return the new entity ID. The component navigates after awaiting the call.
Pattern: `src/pages/katas/NewKataPage.tsx::NewKataPage` (useNavigate) + store returns id.

---

## Route Parameters

### URL params — useParams

```
import { useParams } from 'react-router'
// src/pages/katas/EditKataPage.tsx::EditKataPage
const { kataId } = useParams<{ kataId: string }>()

// src/pages/skills/EditSkillPage.tsx::EditSkillPage
const { id } = useParams<{ id: string }>()

// src/pages/applications/ApplicationFederationPage.tsx::ApplicationFederationPage
const { slug } = useParams<{ slug: string }>()
```

Always destructure with a typed generic. Guard against `undefined` before using the param in
store calls or `useEffect` dependencies.

### Query params — two variants

**Variant A — `react-router` native** (simple cases, no hash quirk):
```
import { useSearchParams } from 'react-router'
// src/pages/chat/hooks/useChatInitialPrompt.tsx::useChatInitialPrompt
const [searchParams, setSearchParams] = useSearchParams()
const value = searchParams.get('tab') ?? 'all'
```

**Variant B — project custom hook** (writes `window.history` directly, persists to `sessionStorage` per pathname):
```
import { useSearchParams } from '@/hooks/useSearchParams'
// src/hooks/useSearchParams.ts::useSearchParams
const [searchParams, setSearchParams, clearParams] = useSearchParams()
```

Use Variant B on list/filter pages where query params must survive page refresh.
Use Variant A for one-shot params that do not need persistence (e.g. chat initial prompt).

---

## "Go Back" Navigation (Create / Edit Pages)

`navigate(-1)` breaks when the user opens a create/edit page via a direct link or page refresh
because there is no browser history entry to go back to. Use feature-scoped `goBack*` utilities
instead.

### Pattern A — path-matching (multiple valid previous destinations)

```
// src/pages/assistants/utils/goBackAssistants.ts::goBackAssistants
export const goBackAssistants = (defaultPath = ASSISTANTS_DEFAULT) =>
  navigateBack(defaultPath, ASSISTANTS_DEFAULT, ASSISTANTS_PROJECT, …)
```

`navigateBack` (defined at `src/utils/helpers.ts::navigateBack`) checks `history.stack` for the last
matching route ID; if none, climbs the current URL path segment by segment.

### Pattern B — history stack (single destination)

```
// src/pages/skills/utils/goBackSkills.ts::goBackSkills
export const goBackSkills = (router, defaultRoute = SKILLS_ALL) => {
  const prevRoute = history.stack[history.currentIndex - 1]
  if (prevRoute) { router.back(); return }
  router.push({ name: defaultRoute })
}
```

`history` is a Valtio proxy updated by `useHistoryStack` in `App.tsx`
(`src/hooks/appLevel/useHistoryStack.tsx::history`).

| Use | When |
|---|---|
| `goBackAssistants()` on `NewAssistantPage` | Multiple valid parent routes (project / marketplace / templates) |
| `goBackSkills(router)` on `NewSkillPage` | Single destination; legacy router object passed in |
| Direct `navigate(-1)` | Only inside pages where history is guaranteed (e.g. detail → edit flow inside one session) |

---

## Protected Routes

The project does not use a generic `ProtectedRoute` wrapper component. Protection is handled
by two complementary mechanisms:

**1. Feature-flag gate** — `src/router.tsx::routes` spreads every route array unconditionally; a flagged route wraps its element in `FeatureGuard` (`src/components/FeatureGuard.tsx::FeatureGuard`), as the data-source routes do with `src/router.tsx::dataSourcesSurfaceFlags`.

**2. Component-level role check** — admin-only UI is hidden inside the component using the
store snapshot:
```
src/pages/settings/AdministrationPage.tsx::AdministrationPage
const { user } = useSnapshot(userStore)
{user?.isAdmin ? <AdminToolsCard /> : null}
```

There is no `loader`-based redirect guard in the production codebase. Actual protection is in-component or
through `FeatureGuard` on the route element.

---

## Code Splitting / Lazy Loading

The router does **not** use `React.lazy` or `import(…)` for route components. All page
components are statically imported at the top of `src/router.tsx`. Bundle
splitting is handled by Vite/Rollup at build time, not by runtime lazy loading in the router.

---

## Route ID Constants

Route IDs serve double duty: they identify routes in the router config and are the strings
passed to `navigateBack` / `router.push({ name })`. Always use the constants, never inline strings.

```
src/constants/routes.ts::CHATS
CHATS, ASSISTANTS_DEFAULT, ASSISTANT_DETAILS, NEW_WORKFLOW, VIEW_WORKFLOW,
EDIT_WORKFLOW, ANALYTICS, SKILLS_ALL, NEW_SKILL, SKILL_DETAILS …
```

URL builder utilities keep path construction off call sites:
```
src/pages/assistants/utils/getAssistantLink.tsx::getAssistantLink
  getAssistantLink(assistant) → readable path from `buildReadablePath`, else `${getRootPath()}/assistants/${encodeURIComponent(assistant.id)}`
```

---

## Anti-Patterns (DO/DON'T)

| DO | DON'T |
|---|---|
| Import from `'react-router'` | Import from `'react-router-dom'` (separate package in v6; merged in v7) |
| Use `<Link>` for in-app navigation | Use `<a href='#/path'>` — bypasses router state |
| Use route ID constants from `src/constants/routes.ts` | Hardcode path strings at call sites |
| Use feature `goBack*` utilities on create/edit pages | Call `navigate(-1)` on pages reachable via direct link |
| Use `useParams<{ id: string }>()` with typed generic | Read `params.id` without typing |
| Guard `useEffect` with param check (`if (id)`) before fetching | Call store method with `undefined` |
| Use `useSearchParams` from `@/hooks/useSearchParams` for filter pages | Store filter state only in local component state |
| Use loader `redirect()` for default sub-routes | Add a catch-all `index` component that just calls `navigate` |
| Keep route arrays feature-scoped in `src/router.tsx` | Scatter route definitions across page files |
| Spread auth routes outside the root route object | Nest sign-in/sign-up under `App` (they must render without the nav shell) |

## Examples

### A feature's back navigation

Source: `src/pages/workflows/utils/goBackWorkflows.ts::goBackWorkflows`

```ts
export const goBackWorkflows = (
  defaultPath: string | (Partial<RouteOptions> & { name: string }) = WORKFLOWS_ALL
) => {
  navigateBack(
    defaultPath,
    CHATS,
    WORKFLOWS_ALL,
    WORKFLOWS_MY,
    WORKFLOWS_TEMPLATES,
    FAVORITES,
    FAVORITES_WORKFLOWS,
    WORKFLOWS_MARKETPLACE,
    VIEW_WORKFLOW,
    VIEW_WORKFLOW_TEMPLATE
  )
}
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| Back on a create or edit page leaves the app after a refresh | `navigate(-1)` has no history entry after a reload | use the feature's `goBack*` helper over `navigateBack` | `src/utils/helpers.ts::navigateBack` |
| A crafted `#/` link or stored return URL redirects off-site | protocol-relative or backslash paths reach `location` | keep `redirectHashRoutes` and `sanitizePostLoginRedirect` stripping them | `src/utils/postLoginRedirect.ts::sanitizePostLoginRedirect` |
| A guarded page shows the 404 page in a deployment that has the feature | its `FeatureGuard` flag key differs from the backend config id | use the key from `FEATURE_FLAGS` | `src/components/FeatureGuard.tsx::FeatureGuard` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Add a route | add a `RouteObject` with an `id` from `src/constants/routes.ts` to the feature's array | `src/router.tsx::routes` |
| Navigate from non-React code | `getRouter().navigate(...)` | `src/routerInstance.ts::getRouter` |