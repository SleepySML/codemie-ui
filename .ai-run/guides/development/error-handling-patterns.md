# Error Handling Patterns

> Error handling strategy for CodeMie UI — stores, boundaries, forms, and notifications.

---

## Overview

Request failures are toasted by `api` from the error body unless the caller opts out; stores add their own message where needed and rethrow when the component must react. Render errors under the root route reach `ErrorPage`; A2UI blocks have their own boundary.

| Part | What it holds | Start here |
|---|---|---|
| `src/utils/toaster.ts` | toast API used for user-facing errors | `src/utils/toaster.ts::setToasterAnnouncer` |
| `src/utils/api.ts` | error parsing and default error toast | `src/utils/api.ts::formatErrorMessage` |
| `src/pages/error/` | route error element | `src/pages/error/ErrorPage.tsx::ErrorPage` |
| `src/a2ui/fallback.tsx` | surface-scoped error boundary | `src/a2ui/fallback.tsx::A2uiErrorBoundary` |

## Layer Responsibilities

| Layer | Responsibility | What it does |
|-------|---------------|--------------|
| **API Client** | Detect HTTP errors | Throw on non-2xx responses |
| **Store** | Handle errors, update state | `catch` → set `error`, show toaster |
| **Component** | Display error UI | Read `error` from snapshot, show retry |
| **Error Boundary** | Catch render crashes | Show fallback UI, log to console |

Never bypass a layer — components must not catch API errors directly.

---

## Error Boundaries

The root route renders `ErrorPage` as its React Router `ErrorBoundary` (`src/router.tsx::routes`), so a render error anywhere under it replaces the page. The one component boundary is scoped to a single surface, agent-authored A2UI blocks: `src/a2ui/fallback.tsx::A2uiErrorBoundary`.
Copy its shape rather than inventing a generic one.

What that boundary actually does:
- `state: { hasError: boolean }` — it does not retain the error object.
- `static getDerivedStateFromError()` returns `{ hasError: true }`.
- `componentDidCatch(error, info)` logs through `console.error` with a
  bracketed source tag, behind an `eslint-disable-next-line no-console`.
- Fallback is a bordered notice using theme tokens (`border-border-primary/40`,
  `text-text-secondary`) and a `data-testid` so tests can assert the degraded state.
- No `fallback` prop, no retry button. Degrade quietly; do not offer an action
  the boundary cannot perform.

**When to add one**: around a surface rendered from data you do not control —
agent-authored payloads, third-party widgets, anything that can throw during
render. Give each boundary its own log tag and `data-testid`; a shared boundary
tells you nothing about which surface failed.

---

## API Error Handling in Stores

`src/store/assistants.ts` — standard store error pattern:

```typescript
export const assistantsStore = proxy({
  loading: false,
  error: null as string | null,

  async fetchAssistants() {
    this.loading = true
    this.error = null
    try {
      const response = await api.get('assistants')
      if (!response.ok) throw new Error(`Failed: ${response.statusText}`)
      this.assistants = await response.json()
    } catch (error: any) {
      this.error = error.message
      toaster.error('Failed to load assistants')
      console.error('fetchAssistants error:', error)
    } finally {
      this.loading = false
    }
  },
})
```

Rules:
- Always reset `error = null` at the start of an action.
- Always use `finally` to reset `loading` — never leave it stuck on `true`.
- For mutation actions (`create`, `update`, `delete`): `toaster.error` in the store, then `throw error` so the component can respond (e.g. keep form open).

### DO / DON'T

| DON'T | DO |
|-------|----|
| `.data` (Axios pattern) | `await response.json()` |
| Show `error.stack` in toaster | Show user-friendly string |
| Leave `loading = true` on error | Use `finally` block |
| Catch API errors in components | Handle in store, expose `error` field |
| Generic `'Error'` message | `'Failed to create assistant: name already exists'` |

---

## Component Error Display

Components read `error` from `useSnapshot` and render error state:

```tsx
function AssistantsList() {
  const { assistants, loading, error } = useSnapshot(assistantsStore)

  if (error) {
    return (
      <div className='p-4 bg-red-50 border border-red-200 rounded-lg'>
        <p className='text-red-800'>{error}</p>
        <button onClick={() => assistantsStore.fetchAssistants()}
          className='mt-2 text-red-600 underline'>
          Retry
        </button>
      </div>
    )
  }
  if (loading) return <Spinner />
  return <AssistantGrid assistants={assistants} />
}
```

**Render order**: error → loading → empty → data. Always handle all four states.

---

## Form Validation Errors

Schemas live **next to the form that uses them**, not in a shared `validation.ts`.
The reference implementation is `src/pages/assistants/components/AssistantForm/AssistantForm.tsx`:
`formSchema` is declared at module scope with `Yup.object()`, and the form's value
type is derived from it rather than hand-written:

```typescript
const formSchema = Yup.object().shape({ /* fields */ })
export type AssistantFormSchema = Yup.InferType<typeof formSchema>
```

Deriving the type with `Yup.InferType` is the point — a hand-written interface
drifts from the schema silently, and the compiler never notices.

Compose shared field groups instead of duplicating them. `AssistantForm` pulls
its guardrail fields in via `.shape(guardrailAssignmentsSchema)`, imported from
the component that owns them.

Wire the schema with `yupResolver`, typing `useForm` on the inferred type:

```typescript
const { register, handleSubmit, formState: { errors } } =
  useForm<AssistantFormSchema>({ resolver: yupResolver(formSchema) })
```

Display inline error below each field:
```tsx
<input {...register('name')} className={cn('input', errors.name && 'border-red-500')} />
{errors.name && <p className='mt-1 text-sm text-red-600'>{errors.name.message}</p>}
```

Form submit error handling — store throws, component catches and logs (toaster already shown by store):
```typescript
const onSubmit = async (data: AssistantData) => {
  try {
    await assistantsStore.createAssistant(data)
    navigate('/assistants')
  } catch {
    // toaster already shown in store — do not double-notify
  }
}
```

---

## Toast Notifications

`src/utils/toaster.ts` — wraps `toastify-js`; each method ignores empty text and also announces the text to screen readers (`src/utils/toaster.ts::toaster`):

| Method | Duration | Position | Use for |
|--------|----------|----------|---------|
| `toaster.success(msg)` | 3000ms | top-right | Completed mutations |
| `toaster.error(msg)` | 10000ms | top-right | Failures |
| `toaster.info(msg)` | 3000ms | top-right | Neutral status updates |

Toasts are called from **stores only** (never from components). Components handle navigation/state transitions after success.

---

## Common Async Patterns

### Standard async action

```typescript
const handleAction = async () => {
  try {
    setLoading(true)
    await performAction()
    toaster.success('Action completed')
  } catch (error: any) {
    toaster.error(error.message || 'Action failed')
    console.error(error)
  } finally {
    setLoading(false)
  }
}
```

### Graceful degradation (non-critical data)

```typescript
const getData = async () => {
  try {
    return await api.get('data').then(r => r.json())
  } catch {
    console.warn('Failed to fetch, using fallback')
    return getDefaultData()
  }
}
```

### Retry with exponential back-off

```typescript
const fetchWithRetry = async (endpoint: string, retries = 3) => {
  for (let i = 0; i < retries; i++) {
    try {
      const r = await api.get(endpoint)
      if (r.ok) return r
    } catch (error) {
      if (i === retries - 1) throw error
      await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)))
    }
  }
  throw new Error('Max retries exceeded')
}
```

---

## Error Message Quality

| Bad | Good |
|-----|------|
| `'Error'` | `'Failed to save assistant. Please try again.'` |
| `'POST /assistants returned 500'` | `'Server error — please try again later.'` |
| Show `error.stack` to user | Log stack to console, show friendly string |
| Generic across all errors | Specific to the failed operation |

---

## Examples

### A surface-scoped error boundary

Source: `src/a2ui/fallback.tsx::A2uiErrorBoundary`

```ts
export class A2uiErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false }

  static getDerivedStateFromError(): State {
    return { hasError: true }
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    // eslint-disable-next-line no-console
    console.error('[A2UI] failed to render surface', error, info)
  }

  componentDidUpdate(prevProps: Props) {
    if (this.state.hasError && prevProps.resetKey !== this.props.resetKey) {
      this.setState({ hasError: false })
    }
  }

  render() {
    if (this.state.hasError) {
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| Two toasts for one failure | `api` toasts and the store toasts again | pass `skipErrorHandling: true` where the store owns the message | `src/store/skills.ts::skillsStore.getSkillById` |
| `loading` stays true after a failure | no `finally` reset | reset loading in `finally` | `src/store/skills.ts::skillsStore.getSkillById` |
| A malformed agent payload breaks a chat message | render error inside an A2UI surface | keep the surface inside `A2uiErrorBoundary` | `src/a2ui/fallback.tsx::A2uiErrorBoundary` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Show an error to the user | `toaster.error(message)` in the store | `src/utils/toaster.ts::setToasterAnnouncer` |