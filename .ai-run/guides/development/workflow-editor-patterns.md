# Workflow Editor Patterns

> Visual workflow editor using `@xyflow/react` (React Flow) + `dagre` for auto-layout.

---

## Overview

The workflow editor renders a React Flow canvas from `src/pages/workflows/editor/` and keeps every graph mutation in `src/utils/workflowEditor/`: `actions/` change states and connections, `build/` turns a configuration into nodes and edges and lays them out, `serialization/` converts between the YAML the backend stores and the editor model, and `helpers/` validate connections and iterators.

| Part | What it holds | Start here |
|---|---|---|
| `src/utils/workflowEditor/actions/` | state, connection and iterator mutations | `src/utils/workflowEditor/actions/connections/createConnection.ts::createConnectionAction` |
| `src/utils/workflowEditor/build/` | nodes, edges and dagre layout | `src/utils/workflowEditor/build/layout.ts::applyLayout` |
| `src/utils/workflowEditor/serialization/` | YAML to model and back | `src/utils/workflowEditor/serialization/serializer.ts::serialize` |
| `src/utils/workflowEditor/helpers/` | connection validation and node helpers | `src/utils/workflowEditor/helpers/connections/connectionValidator.ts::isValidConnection` |
| `src/pages/workflows/editor/` | canvas components | `src/pages/workflows/editor/WorkflowEditor.tsx::WorkflowEditor` |

## Architecture Overview

The editor separates **UI components** from **editor logic** strictly. Components render; utils mutate state.

```
src/pages/workflows/
├── EditWorkflowPage.tsx          # editor entry point (NewWorkflowPage.tsx for a new one)
├── WorkflowDetailsPage.tsx       # read-only view and executions
├── editor/                       # canvas: WorkflowEditor.tsx, ConfigPanel, Sidebar, IssuesPanel
├── details/                      # execution views, node panels, state forms
└── components/                   # list, form, version-history and marketplace popups

src/utils/workflowEditor/         # All editor logic (outside pages/)
├── actions/
│   ├── nodes/                    # handleIteratorDrop
│   ├── connections/              # createConnection, deleteConnection
│   ├── states/                   # createState, updateState, removeState, duplicateState, make(Non)Iterable
│   └── config/                   # config update helpers
├── build/
│   ├── nodes/                    # node builders
│   ├── edges/                    # edge builders
│   └── layout.ts                 # dagre auto-layout
├── serialization/
│   ├── serializer.ts             # React Flow → backend format
│   └── deserializer/             # backend format → React Flow
└── helpers/                      # node/state/connection utilities
```

**Rule**: Keep components thin. Any logic beyond JSX rendering belongs in `src/utils/workflowEditor/`.

---

## Node & Edge Types

The React Flow `type` of a node is its `NodeTypes` value as is (`assistant`, `tool`, `sub_workflow`, `conditional`, …; `src/types/workflowEditor/base.ts::NodeTypes`), with no `Node` suffix. Register a new node component in `src/pages/workflows/editor/nodes/index.tsx::nodeTypeComponents`, which `WorkflowEditor` passes as `nodeTypes`.

`buildNodes` leaves a node's `data` empty; `WorkflowEditor` merges its callbacks and execution status into `data`, and a node component reads its state with `data.findState(id)` (`src/pages/workflows/editor/nodes/ToolNode.tsx::ToolNode`) and renders UI only.

---

## State Management

### Store structure

```typescript
// src/hooks/useWorkflowEditor.ts
  const [nodes, setNodes] = useState<WorkflowNode[]>(manager.nodes)
  const [edges, setEdges] = useState<WorkflowEdge[]>(manager.edges)
```

React Flow nodes and edges live in `useWorkflowEditor`'s component state, built from the YAML configuration it receives (`src/hooks/useWorkflowEditor.ts::useWorkflowEditor`); `workflowsStore` holds only the backend `currentWorkflow` (`src/store/workflows.ts::workflowsStore`). Never put React Flow nodes in a store.

### Load and save

- Load: `workflowsStore.getWorkflow(id)` fetches the workflow; its YAML goes to `WorkflowEditor`, whose `useWorkflowEditor(yamlConfig, onConfigurationUpdate)` builds nodes and edges from it.
- Save: the editor hands every change back as a YAML string through `onConfigurationUpdate`; `EditWorkflowPage` saves the form values with `workflowsStore.updateWorkflow(id, values, errorFormat)` (`src/store/workflows.ts::workflowsStore`).

### DO / DON'T

| DON'T | DO |
|-------|----|
| Store React Flow nodes in `currentWorkflow` or any store | Keep them in `useWorkflowEditor` state, derived from the YAML configuration |
| Call API directly from node components | Use store actions |
| Recalculate edges on every render | `useMemo(() => buildEdges(...), [workflow.states])` |

---

## Node Operations

### State actions

`src/utils/workflowEditor/actions/states/` exports `createStateAction`, `updateStateConfigurationAction`, `removeStateAction`, `duplicateStateAction`, `makeStateIterableAction` and `makeStateNonIterableAction`. Each takes the current `WorkflowConfiguration` and returns a new one as `{ config }` (`src/utils/workflowEditor/actions/index.ts::ActionResult`); none touches a store. `useWorkflowEditor`'s manager applies the result and rebuilds nodes and edges.

`removeStateAction` returns the configuration unchanged for the start and end states, and drops assistants, tools and custom nodes no remaining state references (`src/utils/workflowEditor/actions/states/removeState.ts::removeStateAction`).

---

## Connection Operations

### Creating a connection

`src/utils/workflowEditor/actions/connections/createConnection.ts`

`createConnectionAction(connection, config)` returns `{ config }` unchanged, with a console warning and no toast, when the source or target is missing or both ends are meta states other than END; otherwise it returns the configuration with the connection added (start, to-decision, from-decision or direct) (`src/utils/workflowEditor/actions/connections/createConnection.ts::createConnectionAction`).

### Connection validation rules

`src/utils/workflowEditor/helpers/connections/connectionValidator.ts`

| Constant in `ERROR_MSGS` | Error message |
|------|---------------|
| `SELF_LOOP` | 'Cannot connect a node to itself' |
| `CIRCULAR_CONNECTION` | 'Cannot create circular connection: target node already connects back to source' |
| `ITERATOR_MULTIPLE_PARENTS` | 'Cannot connect to a node that is in an iterator and already has a parent' |

`isValidConnection` (`src/utils/workflowEditor/helpers/connections/connectionValidator.ts::isValidConnection`) applies these and the node-type rules before a connection is accepted.

---

## Serialization

### Deserializer — backend → React Flow

`src/utils/workflowEditor/serialization/deserializer/deserializer.ts`

- Map each `WorkflowState` to a `Node`: `type = '${state.type}Node'`, `position = state.position ?? { x: 0, y: 0 }`, `data = { state }`.
- Flatten `state.transitions` entries into `Edge` objects: `source = state.id`, `target = targetId`, `sourceHandle = handle`.

### Serializer — React Flow → backend

`src/utils/workflowEditor/serialization/serializer.ts`

`serialize(configuration)` turns the `WorkflowConfiguration` into the YAML string the backend stores: it splits the states into `states`, `orphaned_states` and `meta_states` and writes them with `yaml.dump` (`src/utils/workflowEditor/serialization/serializer.ts::serialize`).

**Always include `position` in serialization** — omitting it loses manual layout on reload.

---

## Auto-Layout (dagre)

`src/utils/workflowEditor/build/layout.ts`

`applyLayout(nodes, edges)` lays the graph out with dagre, left to right (`rankdir: 'LR'`); it skips iterator and note nodes, lays iterator children out as top-level nodes, then fits each iterator around its children and converts their positions back to relative (`src/utils/workflowEditor/build/layout.ts::applyLayout`). There is no `autoLayout` export.

---

## Iterator Nodes (Nested Nodes)

Iterator nodes can contain child nodes. When a node is dropped inside an iterator:

- Set `droppedNode.parentNode = iteratorNode.id`.
- Set `droppedNode.extent = 'parent'` to constrain movement within the parent.
- Push the child state into `iteratorState.states`.

Check containment with `isInsideIterator(droppedNode.position, iteratorNode)` before applying nesting.

---

## Viewport Management

| Action | API |
|--------|-----|
| Focus a specific node | `reactFlowInstance.setCenter(x + W/2, y + H/2, { zoom: 1.5, duration: 800 })` |
| Fit all nodes | `reactFlowInstance.fitView({ padding: 0.2, duration: 800 })` |

Obtain `reactFlowInstance` via the `useReactFlow()` hook inside a component wrapped in `<ReactFlowProvider>`.

---

## Testing Strategy

Test logic utilities — not the React Flow canvas itself:

```typescript
describe('createState', () => {
  it('generates a unique state ID', () => {
    const { state } = createState('ai', { x: 0, y: 0 })
    expect(state.id).toMatch(/^state_/)
  })
  it('sets correct React Flow node type', () => {
    const { node } = createState('decision', { x: 0, y: 0 })
    expect(node.type).toBe('decisionNode')
  })
})
```

Test `isValidConnection`, `serialize`, `deserialize` and `applyLayout` as pure functions with no DOM dependency.

## Examples

### Laying out the graph

Source: `src/utils/workflowEditor/build/layout.ts::applyLayout`

```ts
export const applyLayout = (
  nodes: WorkflowNode[],
  edges: WorkflowEdge[]
): WorkflowNode[] => {
  if (nodes.length === 0) return nodes

  const { iterators, childrenToLayout, topLevelToLayout } = categorizeNodes(nodes)
  const nodesToLayout = [...topLevelToLayout, ...childrenToLayout]

  // Layout all non-iterator nodes with dagre (children get absolute positions)
  const layoutedNodes = layoutNodes(nodesToLayout, edges)

  // For each iterator, fit it around its children and convert child positions to relative
  const iteratorAdjustments = iterators.map(iterator => {
    const children = layoutedNodes.filter(node => node.parentId === iterator.id)
    return adjustIteratorToChildren(iterator, children)
  })

  const adjustedIterators = iteratorAdjustments.map(a => a.iterator)
  const adjustedChildren = iteratorAdjustments.flatMap(a => a.children)
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A connection is refused with a circular-connection message | the target already connects back to the source | connect in the other direction or remove the back edge | `src/utils/workflowEditor/helpers/connections/connectionValidator.ts::isValidConnection` |
| Iterator children jump after layout | layout runs on absolute positions | let `applyLayout` convert child positions back to relative | `src/utils/workflowEditor/build/layout.ts::applyLayout` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Change what is written to the backend | edit `serialize` and its tests | `src/utils/workflowEditor/serialization/serializer.ts::serialize` |