# Reusable Components Catalog — Factory Guide

All shared components live in `src/components/`. Import via `@/components/<Name>` alias.

---

## Overview

`src/components/` holds the UI shared by several features: one folder per component with an `index.ts` re-export, and `form/` for every form input. Feature-private UI stays under `src/pages/<feature>/components/`.

| Part | What it holds | Start here |
|---|---|---|
| `src/components/Accordion/` | collapsible section | `src/components/Accordion/Accordion.tsx::Accordion` |
| `src/components/AceEditor/` | Ace code editor wrapper | `src/components/AceEditor/AceEditor.tsx::AceEditorRef` |
| `src/components/Announcement/` | visually hidden polite live region, paired with useAnnouncementQueue | `src/components/Announcement/Announcement.tsx::Announcement` |
| `src/components/Avatar/` | user and assistant avatar image | `src/components/Avatar/Avatar.tsx::AvatarProps` |
| `src/components/BlockedImageBadge/` | badge shown in place of an image the allow list blocked | `src/components/BlockedImageBadge/BlockedImageBadge.tsx::BLOCKED_IMAGE_BADGE_CLASS` |
| `src/components/BudgetSelector/` | budget picker | `src/components/BudgetSelector/BudgetSelector.tsx::BudgetSelector` |
| `src/components/Button/` | the project button with type and size variants | `src/components/Button/Button.tsx::ButtonProps` |
| `src/components/Card/` | clickable card that opens on Enter or Space | `src/components/Card/Card.tsx::CardProps` |
| `src/components/Checker/` | check/status indicator | `src/components/Checker/Checker.tsx::Checker` |
| `src/components/CodeBlock/` | highlighted code block with copy | `src/components/CodeBlock/CodeBlock.tsx::CodeBlock` |
| `src/components/ConfirmationModal/` | destructive-action confirmation dialog built on Popup | `src/components/ConfirmationModal/ConfirmationModal.tsx::ConfirmationModalProps` |
| `src/components/CustomerSupportMessage/` | customer support contact message | `src/components/CustomerSupportMessage/CustomerSupportMessage.tsx::CustomerSupportMessage` |
| `src/components/DataOverlayButton/` | button that opens a data overlay | `src/components/DataOverlayButton/DataOverlayButton.tsx::DataOverlayButtonProps` |
| `src/components/DropdownButton/` | button with a dropdown menu | `src/components/DropdownButton/DropdownButton.tsx::DropdownButton` |
| `src/components/EditOutputForm/` | form for editing a generated output | `src/components/EditOutputForm/EditOutputForm.tsx::EditOutputForm` |
| `src/components/Editor/` | Quill rich-text editor wrapper | `src/components/Editor/Editor.tsx::EditorValue` |
| `src/components/ExpandedYaml/` | full-size YAML editor frame | `src/components/ExpandedYaml/ExpandedYamlFrame.tsx::ExpandedYamlFrame` |
| `src/components/FavoriteButton/` | favorite toggle | `src/components/FavoriteButton/FavoriteButton.tsx::FavoriteButton` |
| `src/components/FilterAccordionItem/` | accordion item inside a filter panel | `src/components/FilterAccordionItem/FilterAccordionItem.tsx::FilterAccordionItem` |
| `src/components/Filters/` | filter panel | `src/components/Filters/Filters.tsx::Filters` |
| `src/components/FloatingKataWindow/` | floating kata window kept inside the screen | `src/components/FloatingKataWindow/FloatingKataWindow.tsx::FloatingKataWindow` |
| `src/components/HelpLauncher/` | help panel launcher | `src/components/HelpLauncher/HelpLauncher.tsx::HelpLauncherProps` |
| `src/components/Hint/` | inline hint with tooltip | `src/components/Hint/Hint.tsx::Hint` |
| `src/components/InfoWarning/` | inline info or warning banner | `src/components/InfoWarning/InfoWarning.tsx::InfoWarning` |
| `src/components/Layouts/` | page layouts with tabs | `src/components/Layouts/Layout/Layout.tsx::LayoutTab` |
| `src/components/Link/` | styled link | `src/components/Link/Link.tsx::Link` |
| `src/components/MdEditor/` | markdown editor | `src/components/MdEditor/MdEditor.tsx::MdEditor` |
| `src/components/Message/` | inline message box | `src/components/Message/Message.tsx::InfoMessageProps` |
| `src/components/Navigation/` | main navigation menu | `src/components/Navigation/Navigation.tsx::Navigation` |
| `src/components/NavigationMore/` | overflow navigation menu | `src/components/NavigationMore/NavigationMore.tsx::NavigationItem` |
| `src/components/OAuthSignInButton/` | OAuth provider sign-in button | `src/components/OAuthSignInButton/OAuthSignInButton.tsx::OAuthSignInButtonProps` |
| `src/components/Onboarding/` | onboarding tour provider and steps | `src/components/Onboarding/index.ts::OnboardingProvider` |
| `src/components/OwnedIcon/` | icon with an ownership badge | `src/components/OwnedIcon/OwnedIcon.tsx::OwnedIcon` |
| `src/components/PageHeader/` | page title bar with header actions | `src/components/PageHeader/PageHeader.tsx::PageHeader` |
| `src/components/Pagination/` | pagination control | `src/components/Pagination/Pagination.tsx::PaginationProps` |
| `src/components/PinAssistantButton/` | pin-assistant toggle | `src/components/PinAssistantButton/PinAssistantButton.tsx::PinAssistantButton` |
| `src/components/Popup/` | the project modal; wraps the PrimeReact Dialog | `src/components/Popup/Popup.tsx::PopupProps` |
| `src/components/PremiumModelBadge/` | badge marking a premium model | `src/components/PremiumModelBadge/PremiumModelBadge.tsx::PREMIUM_MODEL_TOOLTIP` |
| `src/components/ProgressBar/` | progress indicator | `src/components/ProgressBar/ProgressBar.tsx::ProgressBar` |
| `src/components/ProjectNameCell/` | table cell renderer for the project column | `src/components/ProjectNameCell/ProjectNameCell.tsx::renderProjectNameCell` |
| `src/components/ProjectSelector/` | project picker | `src/components/ProjectSelector/ProjectSelector.tsx::ProjectSelector` |
| `src/components/ResizableSeparator/` | drag handle between resizable panes | `src/components/ResizableSeparator/ResizableSeparator.tsx::ResizableSeparator` |
| `src/components/SchemaForm/` | form rendered from a JSON schema | `src/components/SchemaForm/SchemaForm.tsx::SchemaForm` |
| `src/components/SearchableCombobox/` | searchable combobox | `src/components/SearchableCombobox/SearchableCombobox.tsx::ComboboxItem` |
| `src/components/SelectButton/` | segmented button group | `src/components/SelectButton/SelectButton.tsx::SelectButtonOption` |
| `src/components/Sidebar/` | collapsible sidebar container | `src/components/Sidebar/Sidebar.tsx::Sidebar` |
| `src/components/SidebarNavigation/` | grouped navigation inside a sidebar | `src/components/SidebarNavigation/SidebarNavigation.tsx::SidebarNavigation` |
| `src/components/SkipLink/` | skip-to-content link that focuses main | `src/components/SkipLink/SkipLink.tsx::SkipLink` |
| `src/components/Spinner/` | loading indicator | `src/components/Spinner/Spinner.tsx::Spinner` |
| `src/components/StatusBadge/` | status badge | `src/components/StatusBadge/StatusBadge.tsx::StatusEnum` |
| `src/components/Table/` | data table | `src/components/Table/Table.tsx::TableProps` |
| `src/components/Tabs/` | tab strip | `src/components/Tabs/Tabs.tsx::Tab` |
| `src/components/TabsMenu/` | tabs rendered as a menu | `src/components/TabsMenu/TabsMenu.tsx::TabsMenu` |
| `src/components/TextDiffView/` | side-by-side text diff | `src/components/TextDiffView/TextDiffView.tsx::TextDiffView` |
| `src/components/Thought/` | agent thought view | `src/components/Thought/Thought.tsx::ThoughtView` |
| `src/components/ToolCallPolicyDropdown/` | tool-call policy picker | `src/components/ToolCallPolicyDropdown/ToolCallPolicyDropdown.tsx::ToolCallPolicy` |
| `src/components/Tooltip/` | tooltip | `src/components/Tooltip/Tooltip.tsx::Tooltip` |
| `src/components/TooltipButton/` | button with a tooltip | `src/components/TooltipButton/TooltipButton.tsx::TooltipButton` |
| `src/components/TruncationBadge/` | marks output the backend cut at a token limit | `src/components/TruncationBadge/TruncationBadge.tsx::TruncationBadge` |
| `src/components/UserEmailAutocomplete/` | email picker backed by the user directory | `src/components/UserEmailAutocomplete/UserEmailAutocomplete.tsx::UserEmailAutocomplete` |
| `src/components/UserFilter/` | user filter control | `src/components/UserFilter/UserFilter.tsx::UserFilter` |
| `src/components/UserSelector/` | user picker | `src/components/UserSelector/UserSelector.tsx::UserOption` |
| `src/components/appLevel/` | components mounted once by App: popups, banner, toasts | `src/components/appLevel/SessionExpiredPopup.tsx::SessionExpiredPopup` |
| `src/components/details/` | detail-page building blocks | `src/components/details/DetailsBadge.tsx::DetailsBadge` |
| `src/components/form/` | form inputs, see the form patterns guide | `src/components/form/Input/Input.tsx::Input` |
| `src/components/guardrails/` | guardrail assignment UI | `src/components/guardrails/GuardrailAssignmentPanel/GuardrailAssignmentPanel.tsx::GuardrailAssignmentPanel` |
| `src/components/markdown/` | markdown renderer and its token components | `src/components/markdown/Markdown.tsx::Markdown` |
| `src/components/sharing/` | share dialog and the sharing API client | `src/components/sharing/sharingClient.ts::createSharingClient` |
| `src/components/subagents/` | built-in subagent catalog toggles | `src/components/subagents/BuiltinSubagentToggle.tsx::BuiltinSubagentCatalogItem` |

## CRITICAL RULE — Modals

**Never** import `Dialog` from PrimeReact directly in feature code.
**Always** use `Popup` (or `ConfirmationModal` for confirmations).

The `Popup` component wraps PrimeReact `Dialog` internally and enforces project conventions
(keyboard handling, footer layout, accessibility).
See `src/components/Popup/Popup.tsx::Dialog` — it imports Dialog; you never should.

---

## Component Catalog

### Layout & Structure

| Component | Purpose | Import path | Key props |
|---|---|---|---|
| `Layout` | App shell: sidebar + page area | `@/components/Layouts/Layout` | `children` |
| `PageHeader` | Page title bar with action slot | `@/components/PageHeader` | `headline?`, `headerActions?: ReactNode` |
| `Sidebar` | Collapsible left nav | `@/components/Sidebar` | internal, used by Layout |
| `SidebarNavigation` | Nav links inside sidebar | `@/components/SidebarNavigation` | internal |

---

### Buttons & Actions

| Component | Purpose | Import path | Key props |
|---|---|---|---|
| `Button` | Primary action button | `@/components/Button` | `type`, `variant`, `size`, `disabled`, `isLoading`, `buttonType` |
| `DropdownButton` | Button with dropdown menu | `@/components/DropdownButton` | varies |
| `TooltipButton` | Button that shows a tooltip | `@/components/TooltipButton` | varies |
| `SelectButton` | Segmented button group | `@/components/SelectButton` | varies |

`Button` variants (from `ButtonType` constant): `base`, `primary`, `action`, `secondary`, `tertiary`, `delete`.
`Button` sizes (from `ButtonSize` constant): `small`, `medium`, `large`.
Real reference: `src/components/Button/Button.tsx::ButtonProps` (ButtonProps interface).

Usage pattern:
```tsx
import Button from '@/components/Button'
import { ButtonType, ButtonSize } from '@/constants'
<Button variant={ButtonType.PRIMARY} size={ButtonSize.MEDIUM} onClick={handleClick}>Label</Button>
```

---

### Modals & Feedback

| Component | Purpose | Import path | Key props |
|---|---|---|---|
| **`Popup`** | **Generic modal — ALWAYS USE THIS** | `@/components/Popup` | `visible`, `onHide`, `header?`, `onSubmit?`, `submitText?`, `busy?`, `hideFooter?`, `footerContent?`, `headerContent?`, `children` |
| `ConfirmationModal` | Destructive-action confirm dialog | `@/components/ConfirmationModal` | `visible`, `header`, `message`, `onConfirm`, `onCancel`, `confirmText?`, `confirmButtonType?` |
| `Spinner` | Loading indicator | `@/components/Spinner` | `className?`, `inline?` |
| `InfoWarning` | Inline info/warning banner | `@/components/InfoWarning` | `message`, `type?` (`InfoWarningType` enum), `header?` |
| `ProgressBar` | Progress indicator | `@/components/ProgressBar` | varies |

`Popup` full props defined at: `src/components/Popup/Popup.tsx::PopupProps`
`ConfirmationModal` props at: `src/components/ConfirmationModal/ConfirmationModal.tsx::ConfirmationModalProps`

Popup usage pattern:
```tsx
import Popup from '@/components/Popup'
<Popup visible={isOpen} onHide={() => setOpen(false)} header='Title' onSubmit={handleSubmit}>
  {/* modal body */}
</Popup>
```

ConfirmationModal usage pattern:
```tsx
import ConfirmationModal from '@/components/ConfirmationModal'
<ConfirmationModal
  visible={showConfirm} header='Delete item?' message='This cannot be undone.'
  onConfirm={handleDelete} onCancel={() => setShowConfirm(false)} />
```

---

### Form Inputs (`src/components/form/`)

| Component | Purpose | Import path | Key props |
|---|---|---|---|
| `Input` | Text input | `@/components/form/Input` | `name`, `label?`, `hint?`, `error?`, `required?`, `leftIcon?`, `rightIcon?`, `orientation?` |
| `Textarea` | Multi-line text | `@/components/form/Textarea` | `name`, `label?`, `error?` |
| `Select` | Dropdown select | `@/components/form/Select` | varies |
| `MultiSelect` | Multi-value select | `@/components/form/MultiSelect` | varies |
| `Autocomplete` | Autocomplete input | `@/components/form/Autocomplete` | varies |
| `Checkbox` | Checkbox | `@/components/form/Checkbox` | varies |
| `RadioButton` | Single radio | `@/components/form/RadioButton` | varies |
| `RadioGroup` | Radio group | `@/components/form/RadioGroup` | varies |
| `Switch` | Toggle switch | `@/components/form/Switch` | varies |
| `File` | File upload | `@/components/form/File` | varies |
| `InputArray` | Dynamic list input | `@/components/form/InputArray` | varies |
| `InfoBox` | Info display inside form | `@/components/form/InfoBox` | varies |

`Input` full props at: `src/components/form/Input/Input.tsx::InputProps`

---

### Data Display

| Component | Purpose | Import path | Key props |
|---|---|---|---|
| `Card` | Entity card (title, avatar, actions) | `@/components/Card` | `title`, `description`, `avatar`, `onClick`, `id`, `actions?`, `status?`, `label?` |
| `Table` | Sortable data table | `@/components/Table` | `items`, `columnDefinitions`, `customRenderColumns?`, `sort?`, `onSort?` |
| `EmptyList` | Empty-state placeholder | `@/components/Table` (same dir, `EmptyList.tsx`) | `message` |
| `Tabs` | Tab navigation + content | `@/components/Tabs` | `tabs`, `activeTab?`, `onChange?`, `isSmall?`, `headerContent?` |
| `Pagination` | Page controls | `@/components/Pagination` | `currentPage`, `totalPages`, `setPage`, `perPage?` |

`Card` props at: `src/components/Card/Card.tsx::CardProps`
`Tabs`/`Tab` interfaces at: `src/components/Tabs/Tabs.tsx::Tab`, `src/components/Tabs/Tabs.tsx::TabsProps`
`Pagination` props at: `src/components/Pagination/Pagination.tsx::PaginationProps`

---

### Details Views (`src/components/details/`)

| Component | Purpose | Import path |
|---|---|---|
| `DetailsSidebar` | Slide-in details panel | `@/components/details/DetailsSidebar` |
| `DetailsProperty` | Label + value row | `@/components/details/DetailsProperty` |
| `DetailsCopyField` | Copyable value row | `@/components/details/DetailsCopyField` |

---

### Navigation & Filters

| Component | Purpose | Import path | Key props |
|---|---|---|---|
| `Filters` | Filter panel | `@/components/Filters` | varies |
| `TabsMenu` | Tab-style menu | `@/components/TabsMenu` | varies |
| `NavigationMore` | Overflow actions menu | `@/components/NavigationMore` | varies |
| `UserFilter` | User picker filter | `@/components/UserFilter` | varies |

---

### Tooltips

Use the **global react-tooltip instance** — no import or extra component required.

```tsx
<span data-tooltip-id='react-tooltip' data-tooltip-content='Text to show'>...</span>
```

The global instance is mounted in `src/utils/tooltip.ts` and registered in `main.tsx`.

**Never** use `@/components/Tooltip` (PrimeReact) or `data-pr-tooltip` in new code.

---

## Utility Components

| Component | Purpose | Import path | Notes |
|---|---|---|---|
| `Checker` | Status/health check display | `@/components/Checker` | varies |
| `Link` | Styled anchor | `@/components/Link` | wraps React Router `Link` |
| `ProjectSelector` | Project-scope dropdown | `@/components/ProjectSelector` | varies |
| `Hint` | Inline tooltip hint icon | `@/components/Hint` | `src/components/Hint/Hint.tsx::HintProps` |

---

## Accessing Real Component Sources

When in doubt about exact prop names or types, read the interface directly:

| Component | Props interface location |
|---|---|
| `Popup` | `src/components/Popup/Popup.tsx::PopupProps` |
| `Button` | `src/components/Button/Button.tsx::ButtonProps` |
| `Input` | `src/components/form/Input/Input.tsx::InputProps` |
| `Card` | `src/components/Card/Card.tsx::CardProps` |
| `Tabs` | `src/components/Tabs/Tabs.tsx::TabsProps` |
| `Pagination` | `src/components/Pagination/Pagination.tsx::PaginationProps` |
| `ConfirmationModal` | `src/components/ConfirmationModal/ConfirmationModal.tsx::ConfirmationModalProps` |
| `Spinner` | `src/components/Spinner/Spinner.tsx::SpinnerProps` |
| `InfoWarning` | `src/components/InfoWarning/InfoWarning.tsx::InfoWarningProps` |
| `Accordion` | `src/components/Accordion/Accordion.tsx::AccordionProps` |
| `AceEditor` | `src/components/AceEditor/AceEditor.tsx::AceEditorProps` |
| `Avatar` | `src/components/Avatar/Avatar.tsx::AvatarProps` |
| `BudgetSelector` | `src/components/BudgetSelector/BudgetSelector.tsx::BudgetSelectorProps` |
| `Checker` | `src/components/Checker/Checker.tsx::CheckerProps` |
| `CodeBlock` | `src/components/CodeBlock/CodeBlock.tsx::CodeBlockProps` |
| `DropdownButton` | `src/components/DropdownButton/DropdownButton.tsx::DropdownButton` |
| `Editor` | `src/components/Editor/Editor.tsx::EditorProps` |
| `ExpandedYamlFrame` | `src/components/ExpandedYaml/ExpandedYamlFrame.tsx::ExpandedYamlFrameProps` |
| `FavoriteButton` | `src/components/FavoriteButton/FavoriteButton.tsx::FavoriteButtonProps` |
| `FilterAccordionItem` | `src/components/FilterAccordionItem/FilterAccordionItem.tsx::FilterAccordionItemProps` |
| `Filters` | `src/components/Filters/Filters.tsx::Filters` |
| `Hint` | `src/components/Hint/Hint.tsx::HintProps` |
| `Link` | `src/components/Link/Link.tsx::Link` |
| `MdEditor` | `src/components/MdEditor/MdEditor.tsx::MdEditorProps` |
| `PageHeader` | `src/components/PageHeader/PageHeader.tsx::PageHeaderProps` |
| `ProgressBar` | `src/components/ProgressBar/ProgressBar.tsx::ProgressBar` |
| `ProjectSelector` | `src/components/ProjectSelector/ProjectSelector.tsx::ProjectSelectorProps` |
| `ResizableSeparator` | `src/components/ResizableSeparator/ResizableSeparator.tsx::ResizableSeparatorProps` |
| `SchemaForm` | `src/components/SchemaForm/SchemaForm.tsx::SchemaForm` |
| `SearchableCombobox` | `src/components/SearchableCombobox/SearchableCombobox.tsx::SearchableComboboxProps` |
| `SelectButton` | `src/components/SelectButton/SelectButton.tsx::SelectButton` |
| `SkipLink` | `src/components/SkipLink/SkipLink.tsx::SkipLink` |
| `StatusBadge` | `src/components/StatusBadge/StatusBadge.tsx::StatusBadgeProps` |
| `Table` | `src/components/Table/Table.tsx::TableProps` |
| `TabsMenu` | `src/components/TabsMenu/TabsMenu.tsx::CustomTabMenuProps` |
| `TextDiffView` | `src/components/TextDiffView/TextDiffView.tsx::TextDiffView` |
| `Tooltip` | `src/components/Tooltip/Tooltip.tsx::TooltipProps` |
| `TooltipButton` | `src/components/TooltipButton/TooltipButton.tsx::TooltipButtonProps` |
| `TruncationBadge` | `src/components/TruncationBadge/TruncationBadge.tsx::TruncationBadgeProps` |
| `UserEmailAutocomplete` | `src/components/UserEmailAutocomplete/UserEmailAutocomplete.tsx::UserEmailAutocompleteProps` |
| `UserFilter` | `src/components/UserFilter/UserFilter.tsx::UserFilterProps` |
| `UserSelector` | `src/components/UserSelector/UserSelector.tsx::UserSelectorProps` |
| `DetailsBadge` | `src/components/details/DetailsBadge.tsx::DetailsBadge` |
| `Markdown` | `src/components/markdown/Markdown.tsx::Markdown` |
| `Layout` | `src/components/Layouts/Layout/Layout.tsx::Layout` |

---

## Choosing Between Popup and ConfirmationModal

| Situation | Use |
|---|---|
| Generic form, settings, detail view in overlay | `Popup` |
| User about to perform a destructive or irreversible action | `ConfirmationModal` |
| Need custom footer buttons or non-standard layout | `Popup` with `footerContent` prop |
| Simple yes/no with a single message string | `ConfirmationModal` |

`ConfirmationModal` internally uses `Popup` — it is a thin wrapper with an icon, standard
header/message layout, and preset confirm/cancel buttons.

---

## DO / DON'T

| Scenario | DON'T | DO |
|---|---|---|
| Modal / dialog | `import { Dialog } from 'primereact/dialog'` | `import Popup from '@/components/Popup'` |
| Destructive confirm | Custom modal with yes/no buttons | `ConfirmationModal` |
| Loading state | Custom spinner HTML | `<Spinner />` |
| Empty list | Custom empty-state JSX inline | `<EmptyList message='...' />` |
| Tooltip | `data-pr-tooltip` / PrimeReact Tooltip | `data-tooltip-id='react-tooltip'` attribute |
| Form input | Raw `<input>` element | `Input` from `@/components/form/Input` |
| Button variant | Hardcoded string `type='primary'` | `ButtonType.PRIMARY` from `@/constants` |
| Error/warning message | Custom div with border | `InfoWarning` with `InfoWarningType` enum |
| Tab navigation | Raw `<button>` tab row | `Tabs` component from `@/components/Tabs` |
| Collection props on config-driven components | Assume the config always passes them (`options.map(...)`) | Default in the destructure (`options = []`) — configs type them optional, TS won't save you |

## Examples

### A small shared component

Source: `src/components/Spinner/Spinner.tsx::Spinner`

```ts
const Spinner: React.FC<SpinnerProps> = ({ className = '', rootClassName, inline }) => {
  return (
    <output
      className={cn(
        'flex justify-center items-center',
        inline ? 'pt-5' : 'min-h-screen',
        rootClassName
      )}
      aria-label="Loading"
    >
      <LoaderSvg className={cn('text-text-primary animate-spin', className)} aria-hidden="true" />
    </output>
  )
}
```

### A table cell renderer kept at module scope

Source: `src/components/ProjectNameCell/ProjectNameCell.tsx::renderProjectNameCell`

```ts
export const renderProjectNameCell = (item: { project_name: string }) => (
  <ProjectNameCell projectName={item.project_name} />
)
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| Sonar flags a nested component definition in a table | the cell renderer is declared inline inside the table component | declare the renderer at module scope, as `renderProjectNameCell` is | `src/components/ProjectNameCell/ProjectNameCell.tsx::renderProjectNameCell` |
| An end-to-end spec cannot find a modal's submit button | the button text changes between flows | pass `submitTestId` / `cancelTestId` to `Popup` | `src/components/Popup/Popup.tsx::PopupProps` |
| Toggling `disabled` on the rich-text editor has no effect | PrimeReact sets Quill's readOnly only at init | keep the effect that calls `quill.disable()` / `enable()` when `disabled` changes | `src/components/Editor/Editor.tsx::Editor` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Show a modal | use `Popup`; for a destructive confirm use `ConfirmationModal` | `src/components/Popup/Popup.tsx::PopupProps` |
| Show a loading state | render `Spinner` | `src/components/Spinner/Spinner.tsx::Spinner` |
| Add a page title bar | render `PageHeader` with `headline` and `headerActions` | `src/components/PageHeader/PageHeader.tsx::PageHeader` |