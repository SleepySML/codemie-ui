# Performance Patterns

> Optimization patterns for CodeMie UI — measure first, then apply.

---

## Overview

Re-render cost comes from `useSnapshot` subscriptions and long lists; the chat sidebar virtualizes its rows with `@tanstack/react-virtual`, and route pages are imported by `src/router.tsx`.

| Part | What it holds | Start here |
|---|---|---|
| `src/pages/chat/components/ChatSidebar/ChatList/` | virtualized chat list | `src/pages/chat/components/ChatSidebar/ChatList/useSidebarVirtualList.ts::useSidebarVirtualList` |
| `src/hooks/useDebounceApply.ts` | debounced apply for inputs | `src/hooks/useDebounceApply.ts::useDebouncedApply` |

## Guiding Principles

1. **Measure first** — profile before optimizing with React DevTools Profiler.
2. **Lazy load** — load code and data when actually needed.
3. **Memoize expensive work** — cache results, not cheap computations.
4. **Minimize re-renders** — prevent unnecessary React updates.
5. **Do not optimize prematurely** — simple components under 50 lines need nothing.

### When to optimize

| Optimize | Skip |
|----------|------|
| Component render > 16ms | Small, fast components |
| Lists with 100+ items | One-time calculations |
| Expensive CPU calculations | Simple components < 50 lines |
| Bundle > 500KB initial | Heavy library used on every screen |

---

## React.memo

Wrap a component in `React.memo` only when:
- It receives the same props on parent re-renders, **and**
- It is non-trivial enough that diffing saves more time than adding it.

```tsx
// Good — memoize an expensive list item
export default React.memo(Card, (prev, next) =>
  prev.title === next.title && prev.description === next.description
)

// Bad — overhead outweighs benefit
const SimpleText = React.memo(({ text }: { text: string }) => <span>{text}</span>)
```

`React.memo` with a custom comparator is useful when prop objects/arrays are re-created on every parent render but their content is stable.

---

## useMemo

Cache expensive calculations; do not cache trivial ones.

| Use useMemo | Skip useMemo |
|-------------|--------------|
| Sort + filter large arrays | `items.length` |
| Build graph/edge structures for workflow | Boolean flag derivation |
| Regex compilation | Simple string concatenation |

```tsx
// Good
const sortedItems = useMemo(() =>
  items.sort((a, b) => a.name.localeCompare(b.name)).filter(i => i.active),
  [items]
)

// Bad — not worth the overhead
const count = useMemo(() => items.length, [items])
```

---

## useCallback

Memoize a callback **only** when it is passed as a prop to a `React.memo` component.

```tsx
// Good — functional update removes the dependency
const handleClick = useCallback(() => setCount(c => c + 1), [])

// Bad — useCallback with nothing to gain
const handleClose = useCallback(() => setOpen(false), [])  // inline is fine if child is not memoized
```

Prefer functional state updates (`c => c + 1`) over closures over state values — they eliminate the state variable from the dependency array.

---

## Lazy Loading

### Route-level

`src/router.tsx` imports every route page statically today; the only `lazy()` route split in `src/` is the Keycloak theme (`src/authentication/keycloak-theme/kc.gen.tsx::KcLoginPage`). Shape of a lazy route page:

```tsx
const AssistantsPage = lazy(() => import('@/pages/assistants/AssistantsPage'))
const WorkflowsPage  = lazy(() => import('@/pages/workflows/WorkflowsPage'))
// Wrap each route element in <Suspense fallback={<Loading />}>
```

### Component-level (heavy, rarely shown)

```tsx
const PdfViewer  = lazy(() => import('@/components/PdfViewer'))
const CodeEditor = lazy(() => import('@/components/CodeEditor'))
// Use inside <Suspense fallback={<Spinner />}>
```

### Dynamic import (heavy utility, infrequently called)

```tsx
const handleExport = async () => {
  const { exportToPdf } = await import('@/utils/pdfExport')
  await exportToPdf(data)
}
```

Never import a heavy library at the top level if it is only needed on a specific user action.

---

## Valtio Snapshot Optimization

### Destructure only what you need

```tsx
// Good — re-renders only when assistants changes
const { assistants } = useSnapshot(assistantsStore)

// Bad — re-renders on any store field change
const state = useSnapshot(assistantsStore)
```

### Batch store mutations

```typescript
// Good — one re-render for all three mutations
const update = () => {
  assistantsStore.loading = true
  assistantsStore.error = null
  assistantsStore.assistants = newAssistants
}

// Bad — three separate re-renders
assistantsStore.loading = true      // re-render
assistantsStore.error = null        // re-render
assistantsStore.assistants = data   // re-render
```

### Derived values

`valtio/utils` `derive` is not used in this repository; compute a derived value from the snapshot inside the component, with `useMemo` when the computation is expensive.

---

## List Rendering

### Keys

| Pattern | When |
|---------|------|
| `key={item.id}` | Always for dynamic lists |
| `key={index}` | Only for truly static, never-reordered lists |
| `key={Math.random()}` | Never — forces full re-mount every render |

### Virtual scrolling (1000+ items)

Virtualize a long list with `@tanstack/react-virtual`'s `useVirtualizer`, as the chat sidebar does (`src/pages/chat/components/ChatSidebar/ChatList/useSidebarVirtualList.ts::useSidebarVirtualList`); `react-window` is not a dependency.

### Conditional rendering vs CSS hide

```tsx
// Good — inactive tab not in DOM
{activeTab === 0 && <Tab1Content />}
{activeTab === 1 && <Tab2Content />}

// Bad — all tabs rendered, hidden with className
<div className={activeTab !== 0 ? 'hidden' : ''}><Tab1Content /></div>
```

---

## Network Optimization

### Debounce search inputs

```tsx
const debouncedSearch = useMemo(
  () => debounce(async (q: string) => {
    const r = await api.get('/search', { params: { q } })
    setResults(await r.json())
  }, TIMEOUTS.DEBOUNCE_INPUT),
  []
)

useEffect(() => {
  if (query) debouncedSearch(query)
  return () => debouncedSearch.cancel()
}, [query, debouncedSearch])
```

Always cancel the debounced function in the cleanup return.

### Pagination over bulk fetch

Always paginate list endpoints — never `GET /assistants` without `page` and `pageSize`. Use `PAGINATION.DEFAULT_PAGE_SIZE` from constants.

### Request deduplication

For identical concurrent requests, cache the in-flight `Promise` and return it to subsequent callers. Clear the cache entry after ~5 seconds.

---

## Asset Optimization

| Rule | Reason |
|------|--------|
| Import SVG as `?react` component | Inlined, tree-shakeable |
| Use `loading="lazy"` on images below fold | Defers network cost |
| Provide explicit `width` and `height` on images | Prevents layout shift (CLS) |
| Prefer WebP with PNG fallback | Smaller transfer size |

---

## Performance Budget

| Metric | Target | Action if exceeded |
|--------|--------|--------------------|
| Initial bundle | < 500KB | Code-split, lazy load |
| Component render | < 16ms | Profile → memoize |
| Time to Interactive | < 3s | Split critical path |
| Long list | < 100 items rendered | Add virtualization |

### Red flags — investigate immediately

- A component takes > 16ms (React Profiler `actualDuration`).
- Bundle initial chunk > 1MB.
- API called on every keystroke without debounce.
- `useSnapshot` on the entire store when only one field is needed.
- Large list rendered without virtualization.
- Heavy library imported at top level but used in one rare action.

---

## DO / DON'T Summary

| DON'T | DO |
|-------|----|
| `React.memo` every component | Only memoize when re-renders are measurably costly |
| `useMemo` for `items.length` | `useMemo` for sort/filter/map of large arrays |
| `useCallback` for all handlers | `useCallback` only for props to memoized children |
| Import entire page at startup | `lazy()` all route-level components |
| `key={Math.random()}` | `key={item.id}` |
| Render 10 000 items directly | `useVirtualizer` from `@tanstack/react-virtual` |
| `useSnapshot(entireStore)` | Destructure only used fields |
| Three separate store mutations | Batch in one function call |
| Fetch all records | Paginate with `PAGINATION.DEFAULT_PAGE_SIZE` |

## Examples

### Measuring a row with its margin

Source: `src/pages/chat/components/ChatSidebar/ChatList/useSidebarVirtualList.ts::getRootFontSizePx`

```ts
export const getRootFontSizePx = () =>
  parseFloat(getComputedStyle(document.documentElement).fontSize) || 16
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| Virtualized rows overlap or leave gaps | `getBoundingClientRect` excludes margins | measure with the margin, as `measureWithMargin` does | `src/pages/chat/components/ChatSidebar/ChatList/useSidebarVirtualList.ts::useSidebarVirtualList` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Virtualize a long list | use `useVirtualizer` from `@tanstack/react-virtual` | `src/pages/chat/components/ChatSidebar/ChatList/useSidebarVirtualList.ts::useSidebarVirtualList` |