# Modal Patterns — Factory Guide

## Overview

Every dialog is a `Popup` (which wraps PrimeReact's `Dialog`) or a `ConfirmationModal` built on it; popups the app opens on its own are ordered by `AutoPopupManager`.

| Part | What it holds | Start here |
|---|---|---|
| `src/components/Popup/` | the modal component and its props | `src/components/Popup/Popup.tsx::PopupProps` |
| `src/components/ConfirmationModal/` | confirmation dialog for destructive actions | `src/components/ConfirmationModal/ConfirmationModal.tsx::ConfirmationModalProps` |
| `src/components/appLevel/AutoPopupManager.tsx` | priority order of automatic popups | `src/components/appLevel/AutoPopupManager.tsx::AutoPopupManager` |

## CRITICAL RULE

**NEVER import or use PrimeReact `Dialog` directly.**
All modals MUST use the `Popup` component from `src/components/Popup/`.

---

## Bad / Best

| Bad | Best |
|-----|------|
| `import { Dialog } from 'primereact/dialog'` | `import Popup from '@/components/Popup'` |
| Separate `<CustomModalHeader />` component | Use `header` / `headerContent` prop on `Popup` |
| Separate `<CustomModalFooter />` component | Use `footerContent` prop on `Popup` |
| 100+ lines of state/effects in modal component | Extract to `useXxxModal.ts` hook |
| Forget `form.reset()` on close | Call `form.reset()` inside `handleClose` |
| Forget debounce cleanup | `return () => debouncedFn.cancel()` in `useEffect` |
| `||` for default values | `??` (nullish coalescing) |

---

## Popup Props Reference

`src/components/Popup/Popup.tsx` — `PopupProps` is exported there; `index.ts` is a bare re-export

**Visibility & control**
- `visible?: boolean` — show/hide
- `onHide: () => void` — required close handler
- `onSubmit?: () => void` — optional submit handler

**Header**
- `header?: string` — plain text header (HTML accepted)
- `headerContent?: ReactNode` — custom header node
- `hideClose?: boolean`

**Footer**
- `hideFooter?: boolean`
- `footerContent?: ReactNode` — replaces default buttons
- `submitText?: string` — default: `'Create'`
- `submitDisabled?: boolean`
- `cancelText?: string` — default: `'Cancel'`
- `cancelButtonType?: ButtonType`
- `submitButtonType?: ButtonType`

**Sizing / styling**
- `className?: string`
- `bodyClassName?: string`
- `isFullWidth?: boolean`
- `limitWidth?: boolean` — constrains to `max-w-lg`
- `withBorder?: boolean` — header border (default `true`)

**Behaviour**
- `dismissableMask?: boolean` — click outside closes (default `true`)
- `busy?: boolean` — blocks dismissal and shows loading on the default submit button (default `false`).
  Pass `busy={submitting}`; disable form fields and custom footer actions yourself.
- `busyHint?: ReactNode` — reassurance text shown in the default footer after five seconds of being busy.

### Read first
| When you are | Exemplar | Why this one | Tests |
|---|---|---|---|
| saving from a Popup that must not be dismissed or submitted twice mid-request | `src/pages/settings/administration/components/projectsManagement/BulkBudgetOverrideModal.tsx::BulkBudgetOverrideModal` | `useInFlightGuard`'s `tryBegin` before the request and `end` in `finally`, `busy={inFlight}` on `Popup`, custom footer and a `fieldset` disabled while in flight | `src/pages/settings/administration/components/projectsManagement/__tests__/BulkBudgetOverrideModal.test.tsx::BulkBudgetOverrideModal` |

---

## Pattern 1 — Simple Modal (Default Footer)

Use for standard create/edit dialogs with submit + cancel.

```tsx
import Popup from '@/components/Popup'
import { ButtonType } from '@/constants'

<Popup
  visible={visible}
  onHide={onHide}
  header='Edit Item'
  submitText='Save Changes'
  onSubmit={handleSave}
  submitButtonType={ButtonType.PRIMARY}
  limitWidth
>
  {/* form fields */}
</Popup>
```

---

## Pattern 2 — Custom Footer

Use when you need multiple buttons, loading state in footer, or non-standard layout.

```tsx
const footer = (
  <div className='flex justify-end gap-3'>
    <Button variant={ButtonType.SECONDARY} onClick={onHide} disabled={loading}>
      Cancel
    </Button>
    <Button variant={ButtonType.PRIMARY} onClick={onSubmit} disabled={loading}>
      {loading ? 'Saving…' : 'Save'}
    </Button>
  </div>
)

<Popup
  visible={visible}
  onHide={onHide}
  header='Custom Footer'
  footerContent={footer}
  className='w-[800px]'
>
  {/* content */}
</Popup>
```

---

## Pattern 3 — Confirmation Modal

Use for delete / destructive actions. Import `ConfirmationModal`, not `Popup` directly.

```tsx
import ConfirmationModal from '@/components/ConfirmationModal'
import { ButtonType } from '@/constants'

<ConfirmationModal
  visible={visible}
  onCancel={onCancel}
  onConfirm={onConfirm}
  header='Delete Item'
  message={`Delete "${name}"?`}
  confirmText='Delete'
  confirmButtonType={ButtonType.DELETE}
  limitWidth
/>
```

---

## Pattern 4 — Modal With Custom Hook

Extract complex state/effects into a co-located hook. Keep the component thin.

```tsx
// useMyModal.ts
import { useState, useCallback, useRef } from 'react'
import { useFocusOnVisible } from '@/hooks/useFocusOnVisible'

export const useMyModal = (visible: boolean, onHide: () => void) => {
  const inputRef = useRef<HTMLInputElement>(null)
  const [data, setData] = useState({})
  const handleClose = useCallback(() => { setData({}); onHide() }, [onHide])
  useFocusOnVisible(inputRef, visible)
  return { data, setData, inputRef, handleClose }
}
```

```tsx
// MyModal.tsx
import Popup from '@/components/Popup'
import { useMyModal } from './useMyModal'

const MyModal = ({ visible, onHide }) => {
  const { data, inputRef, handleClose } = useMyModal(visible, onHide)
  return (
    <Popup visible={visible} onHide={handleClose} header='My Modal' hideFooter>
      <Input ref={inputRef} value={data.name} />
    </Popup>
  )
}
```

---

## Focus & Keyboard

| Requirement | Implementation |
|-------------|---------------|
| Auto-focus first input on open | `useFocusOnVisible(inputRef, visible)` — `src/hooks/useFocusOnVisible.ts` |
| ESC to close | Built into `Popup`. Only add `useEscapeKey` for custom cleanup logic. |
| Restore focus on close | `Popup` handles internally |
| Custom ESC handling | `import { useEscapeKey } from '@/hooks/useEscapeKey'` |

---

## State Reset on Close

Always reset form/state when the modal closes:

```tsx
const handleClose = useCallback(() => {
  form.reset()   // React Hook Form
  onHide()
}, [form, onHide])
```

---

## Debounce Cleanup (Mandatory)

If the modal runs a debounced function:

```tsx
useEffect(() => {
  return () => { debouncedFn.cancel() }
}, [debouncedFn])
```

---

## File Locations

| File | Purpose |
|------|---------|
| `src/components/Popup/Popup.tsx` | Popup wrapper (wraps PrimeReact Dialog) |
| `src/components/Popup/usePopupBusy.ts` | Busy lock behind `busy` / `busyHint` |
| `src/hooks/useInFlightGuard.ts` | Re-entry guard for an async submit (`inFlight`, `tryBegin`, `end`) |
| `src/components/ConfirmationModal/` | Reusable confirmation dialog |
| `src/hooks/useEscapeKey.ts` | ESC key handler hook |
| `src/hooks/useFocusOnVisible.ts` | Auto-focus hook |

---

## Pre-Delivery Checklist

- [ ] Import is `Popup` from `@/components/Popup`, not `Dialog`
- [ ] `onHide` prop always provided
- [ ] State/form reset on close
- [ ] Debounced functions cleaned up
- [ ] Icon-only buttons have `aria-label`
- [ ] Destructive actions use `ConfirmationModal` with `ButtonType.DELETE`
- [ ] Component under 300 lines (extract hook if approaching)

## Examples

### Popup props

Source: `src/components/Popup/Popup.tsx::PopupProps`

```ts
export interface PopupProps {
  isFullWidth?: boolean
  header?: string
  headerDescription?: ReactNode
  visible?: boolean
  onHide: () => void
  onSubmit?: () => void
  submitText?: string
  /** Selector hook for the submit button (e.g. `moderation-submit`). */
  submitTestId?: string
  /** Selector hook for the cancel button. */
  cancelTestId?: string
  submitDisabled?: boolean
  busy?: boolean
  busyHint?: ReactNode
  overlayClassName?: string
  hideFooter?: boolean
  cancelText?: string
  bodyClassName?: string
  withBorder?: boolean
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| Two automatic popups open at once | a popup opened on its own instead of through the manager | add it to `AutoPopupManager`'s priority order | `src/components/appLevel/AutoPopupManager.tsx::AutoPopupManager` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Add a confirmation step | render `ConfirmationModal` with `header`, `message`, `onConfirm`, `onCancel` | `src/components/ConfirmationModal/ConfirmationModal.tsx::ConfirmationModalProps` |
| Add a form modal | render `Popup` with `onSubmit` and `submitText` | `src/components/Popup/Popup.tsx::PopupProps` |