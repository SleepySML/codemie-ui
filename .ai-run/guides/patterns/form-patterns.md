# Form Patterns — Factory Guide

## Overview

Forms use react-hook-form with Yup schemas through `yupResolver`; the inputs under `src/components/form/` take a label, hint and error and wire their validation message through `FieldAnnouncer`.

| Part | What it holds | Start here |
|---|---|---|
| `src/components/form/Autocomplete/` | autocomplete input | `src/components/form/Autocomplete/Autocomplete.tsx::AutocompleteProps` |
| `src/components/form/CronScheduleInput/` | cron schedule input with validation | `src/components/form/CronScheduleInput/CronScheduleInput.tsx::CronScheduleInput` |
| `src/components/form/DatePicker/` | date picker | `src/components/form/DatePicker/DatePicker.tsx::DatePickerProps` |
| `src/components/form/DropzoneArea/` | drag-and-drop file area | `src/components/form/DropzoneArea/DropzoneArea.tsx::DropzoneArea` |
| `src/components/form/DynamicFieldsForm/` | fields rendered from a schema of settings | `src/components/form/DynamicFieldsForm/DynamicFieldsForm.tsx::DynamicFieldsForm` |
| `src/components/form/ExpandableTextarea/` | textarea with an expanded editor | `src/components/form/ExpandableTextarea/ExpandableTextarea.tsx::ExpandableTextareaProps` |
| `src/components/form/FieldAnnouncer/` | announces field errors to screen readers | `src/components/form/FieldAnnouncer/FieldAnnouncer.tsx::FieldAnnouncerRenderProps` |
| `src/components/form/File/` | single file input | `src/components/form/File/File.tsx::FileInput` |
| `src/components/form/FilesDropzone/` | multi-file drop zone | `src/components/form/FilesDropzone/FilesDropzone.tsx::FilesDropzone` |
| `src/components/form/FormAutocomplete/` | autocomplete bound to react-hook-form | `src/components/form/FormAutocomplete/FormAutocomplete.tsx::FormAutocomplete` |
| `src/components/form/ImageUpload/` | image upload with preview | `src/components/form/ImageUpload/ImageUpload.tsx::ImageUpload` |
| `src/components/form/InfoBox/` | info box inside a form | `src/components/form/InfoBox/InfoBox.tsx::InfoBox` |
| `src/components/form/Input/` | text input with label, hint and error | `src/components/form/Input/Input.tsx::Input` |
| `src/components/form/InputArray/` | list of text inputs | `src/components/form/InputArray/InputArray.tsx::InputArray` |
| `src/components/form/InputCopy/` | read-only input with copy | `src/components/form/InputCopy/InputCopy.tsx::InputCopy` |
| `src/components/form/LinksArray/` | list of link inputs | `src/components/form/LinksArray/LinksArray.tsx::LinksArray` |
| `src/components/form/MarkdownEditor/` | markdown editor field | `src/components/form/MarkdownEditor/MarkdownEditor.tsx::MarkdownEditor` |
| `src/components/form/MultiSelect/` | multi-value select | `src/components/form/MultiSelect/MultiSelect.tsx::MultiSelectSize` |
| `src/components/form/OrderList/` | reorderable list | `src/components/form/OrderList/OrderList.tsx::OrderList` |
| `src/components/form/RadioButton/` | single radio | `src/components/form/RadioButton/RadioButton.tsx::RadioButtonProps` |
| `src/components/form/RadioGroup/` | radio group | `src/components/form/RadioGroup/RadioGroup.tsx::RadioOption` |
| `src/components/form/RecordInput/` | key-value record input | `src/components/form/RecordInput/RecordInput.tsx::RecordItem` |
| `src/components/form/Select/` | dropdown select | `src/components/form/Select/Select.tsx::Select` |
| `src/components/form/Switch/` | toggle switch | `src/components/form/Switch/Switch.tsx::Switch` |
| `src/components/form/Textarea/` | multi-line text input | `src/components/form/Textarea/Textarea.tsx::TextareaProps` |
| `src/components/form/VersionedField/` | field with version tabs | `src/components/form/VersionedField/VersionedField.tsx::VERSIONED_FIELD_TAB_ID` |
| `src/components/form/YamlEditor/` | YAML editor field | `src/components/form/YamlEditor/YamlEditor.tsx::YamlEditor` |

## Stack

| Layer | Library | Version |
|-------|---------|---------|
| Form state | React Hook Form | 7.x |
| Validation | Yup | via `@hookform/resolvers/yup` |
| Form components | `src/components/form/` | project-local |

---

## Bad / Best

| Bad | Best |
|-----|------|
| Manual `useState` + validate function | `useForm` + `yupResolver(schema)` |
| Schema defined inside component body | Separate `formSchema.ts` file |
| `<input {...register('field')} />` | `<Controller name='field' control={control} render={...} />` |
| Passing `control` to leaf components | Pass explicit `value`, `onChange`, `error` props |
| Form component > 300 lines | Split into sub-components + custom hook |
| `||` for default values | `??` (nullish coalescing) |
| Arbitrary px values `p-[18px]` | Tailwind scale `p-4` |
| Forget `form.reset()` on close | Call in `handleClose` callback |

---

## Available Form Components (`src/components/form/`)

| Component | Use for |
|-----------|---------|
| `Input` | Single-line text |
| `Textarea` | Multi-line text |
| `Select` | Single dropdown |
| `MultiSelect` | Multi-select dropdown |
| `Autocomplete` | Typeahead input |
| `Checkbox` | Boolean toggle |
| `RadioButton` / `RadioGroup` | Exclusive choice |
| `Switch` | Toggle switch |

### Rules
- R1. Give a new form primitive its error wiring through `FieldAnnouncer` (it computes `aria-invalid` and `aria-describedby` and re-mounts the `role="alert"` node when the text changes, so a changed error is announced again). Evidence: `src/components/form/FieldAnnouncer/FieldAnnouncer.tsx::FieldAnnouncer`
| `File` / `FilesDropzone` | File upload (single file / several files) |

---

## Minimal Form Template

```tsx
import { useForm, Controller } from 'react-hook-form'
import { yupResolver } from '@hookform/resolvers/yup'
import * as yup from 'yup'
import Input from '@/components/form/Input'

const schema = yup.object({
  name: yup.string().required('Name is required'),
  email: yup.string().email('Invalid email').required('Email is required'),
})
type FormData = yup.InferType<typeof schema>

const MyForm = () => {
  const { control, handleSubmit, formState: { errors, isSubmitting } } =
    useForm<FormData>({ resolver: yupResolver(schema), defaultValues: { name: '', email: '' } })

  const onSubmit = async (data: FormData) => { /* call store */ }

  return (
    <form onSubmit={handleSubmit(onSubmit)} className='flex flex-col gap-4'>
      <Controller name='name' control={control}
        render={({ field }) => <Input {...field} label='Name' error={errors.name?.message} required />} />
      <Controller name='email' control={control}
        render={({ field }) => <Input {...field} label='Email' type='email' error={errors.email?.message} required />} />
      <Button type='submit' disabled={isSubmitting}>
        {isSubmitting ? 'Submitting…' : 'Submit'}
      </Button>
    </form>
  )
}
```

---

## Validation Schema Structure

Keep schemas in a dedicated `formSchema.ts`:

```ts
// formSchema.ts
import * as yup from 'yup'

export const myFormSchema = yup.object({
  name: yup.string().required('Name is required').min(3, 'Min 3 chars'),
  email: yup.string().email('Invalid email').required(),
  age: yup.number().typeError('Must be a number').positive().integer().min(18).max(100).nullable(),
  website: yup.string().url('Must be a valid URL').nullable(),
  accepted: yup.boolean().oneOf([true], 'Must accept terms'),
})

export type MyFormData = yup.InferType<typeof myFormSchema>
```

Use `yup.InferType<typeof schema>` — never write the TS type separately.

---

## Displaying Field Errors

Pass `errors.<field>?.message` to the component's `error` prop. The shared form primitives
(`Input`, `Select`, `MultiSelect`, `Textarea`, `FilesDropzone`) wire the ARIA contract internally:
`aria-invalid`, `aria-describedby`, and a keyed `role="alert"` error node with a `useId`-derived
id. `aria-invalid` is always overridden by the primitive; `aria-describedby` supplied by the
caller (e.g. to point at a hint span) is preserved and the error id is appended to it when an
error is present.

```tsx
<Controller name='age' control={control}
  render={({ field }) => (
    <Input {...field} label='Age' type='number' error={errors.age?.message} />
  )} />
```

Never render your own error span next to a primitive — you get a duplicate live region and
the announcement fires twice. If your form uses a raw `<input>` / `<textarea>` (not a
primitive), reproduce the same contract locally, following
[`accessibility-patterns.md`](./accessibility-patterns.md#form-validation) and the
`Textarea` reference implementation in `src/components/form/Textarea/Textarea.tsx`.

---

## Form with Custom Hook (Complex Forms)

Extract all logic when form approaches 200+ lines.

```ts
// useMyForm.ts
import { useForm } from 'react-hook-form'
import { yupResolver } from '@hookform/resolvers/yup'
import { useCallback, useRef } from 'react'
import { useFocusOnVisible } from '@/hooks/useFocusOnVisible'
import { myFormSchema, MyFormData } from './formSchema'
import { myStore } from '@/store/myStore'

export const useMyForm = (visible: boolean, onHide: () => void) => {
  const inputRef = useRef<HTMLInputElement>(null)
  const form = useForm<MyFormData>({ resolver: yupResolver(myFormSchema) })

  useFocusOnVisible(inputRef, visible)

  const handleClose = useCallback(() => { form.reset(); onHide() }, [form, onHide])

  const onSubmit = form.handleSubmit(async (data) => {
    await myStore.createItem(data)
    handleClose()
  })

  return { form, inputRef, handleClose, onSubmit }
}
```

The component then stays thin:
```tsx
const MyForm = ({ visible, onHide }) => {
  const { form, inputRef, handleClose, onSubmit } = useMyForm(visible, onHide)
  return (
    <Popup visible={visible} onHide={handleClose} onSubmit={onSubmit}
      submitDisabled={form.formState.isSubmitting}>
      {/* Controllers only */}
    </Popup>
  )
}
```

---

## Presentation vs Container Components

### Container / Section (can hold `control`, 200–300 lines)
- Owns `Controller` wrappers
- Coordinates section-level state
- Passed `control` + `errors` props

### Presentation / Field (< 100 lines, no RHF dependency)
```tsx
// Explicit props — no `control`, no `Controller` inside
interface EmailFieldProps {
  value: string
  onChange: (v: string) => void
  onBlur: () => void
  error?: string
}
const EmailField = ({ value, onChange, onBlur, error }: EmailFieldProps) => (
  <Input value={value} onChange={onChange} onBlur={onBlur} error={error} label='Email' />
)

// Usage in container:
<Controller name='email' control={control}
  render={({ field, fieldState }) => (
    <EmailField {...field} error={fieldState.error?.message} />
  )} />
```

---

## File Structure for Complex Forms

```
MyComplexForm/
├── index.tsx          # Main component (<300 lines)
├── formSchema.ts      # Yup schemas + InferType exports
├── formTypes.ts       # Extra TS interfaces
├── formHelpers.ts     # Pure helper functions
├── useMyForm.ts       # Custom form hook
├── SectionA.tsx       # Sub-component for section A
└── SectionB.tsx       # Sub-component for section B
```

---

## Accessibility in Forms

| Requirement | Implementation |
|-------------|---------------|
| Visible label | `label` prop on form component, or `<label htmlFor>` |
| Required indicator | `required` prop on component |
| Error announced | `role='alert'` on error span, or `aria-live='polite'` |
| Link input to error | `aria-describedby='fieldname-error'` |
| Invalid state | `aria-invalid={!!errors.field}` |

---

## Validation Lives in the Schema, Not in Component State

Any field invariant ("at least one selected", "required", format) must be enforced by a Yup
validator in the field's config/schema — that is what gates the Save button. Component-local
state (blocked flags, inline guards) is UX feedback only: it can always be bypassed by
`reset()`, `setValue()`, or data loaded from the backend. If removing the component-level
guard would let an invalid value be saved, the schema is missing a validator.

---

## Pre-Delivery Checklist

- [ ] Schema in separate `formSchema.ts`, not in component body
- [ ] Every field invariant enforced by a Yup validator, not only by component state
- [ ] `yup.InferType` used for type — no duplicate TS interface
- [ ] All inputs use `Controller`, not `register`
- [ ] Leaf components receive explicit props, not `control`
- [ ] `form.reset()` called on modal close
- [ ] Loading state: submit button disabled + label change
- [ ] Error messages wired to `error` prop and `aria-describedby`
- [ ] Component under 300 lines

## Examples

### Error wiring for a form field

Source: `src/components/form/FieldAnnouncer/FieldAnnouncer.tsx::FieldAnnouncer`

```ts
const FieldAnnouncer = ({
  id,
  error,
  errorClassName,
  describedBy,
  children,
}: FieldAnnouncerProps) => {
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A screen reader does not re-announce a changed validation error | `role="alert"` fires only when the node is inserted | render the error through `FieldAnnouncer`, which keys the node by its text | `src/components/form/FieldAnnouncer/FieldAnnouncer.tsx::FieldAnnouncer` |
| A required list input has no visible or announced required marker | `required` was not passed to `InputArray` | pass `required`; the label renders the marker | `src/components/form/InputArray/InputArray.tsx::InputArray` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Add a text field | render `Input` with `name`, `label` and the form's error for that name | `src/components/form/Input/Input.tsx::Input` |
| Add a dropdown | render `Select` | `src/components/form/Select/Select.tsx::Select` |
| Add a cron field | render `CronScheduleInput` | `src/components/form/CronScheduleInput/CronScheduleInput.tsx::CronScheduleInput` |