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

| React Flow type string | Component | Backend state type |
|------------------------|-----------|-------------------|
| `aiNode` | `NodeTypes/AiNode.tsx` | `ai` |
| `decisionNode` | `NodeTypes/DecisionNode.tsx` | `decision` |
| `iteratorNode` | `NodeTypes/IteratorNode.tsx` | `iterator` |
| `startNode` | `NodeTypes/StartNode.tsx` | `start` |
| `endNode` | `NodeTypes/EndNode.tsx` | `end` |

Node type string convention: `${backendType}Node`. Register all types in the `nodeTypes` map passed to `<ReactFlow nodeTypes={nodeTypes} />`.

Custom node components receive `data: { state: WorkflowState }` and render UI only — they call action utilities for mutations.

---

## State Management

### Store structure

```typescript
// src/store/workflows.ts
export const workflowsStore = proxy({
  currentWorkflow: null as Workflow | null,
  workflows: [] as Workflow[],
  editorState: {
    nodes: [] as Node[],
    edges: [] as Edge[],
    selectedNode: null as Node | null,
  },
})
```

Keep `editorState` (React Flow format) separate from `currentWorkflow` (backend format). Never store React Flow nodes inside `currentWorkflow`.

### Load and save

- `loadWorkflow(id)` — fetch from API, call `deserializeWorkflow()`, populate both `currentWorkflow` and `editorState`.
- `saveWorkflow()` — call `serializeWorkflow(nodes, edges)`, PUT to API, update `currentWorkflow`.

### DO / DON'T

| DON'T | DO |
|-------|----|
| Store React Flow nodes in `currentWorkflow` | Keep `editorState` separate |
| Call API directly from node components | Use store actions |
| Mutate node `data` without updating `state` | Update both `state` and `node.data.state` |
| Recalculate edges on every render | `useMemo(() => buildEdges(...), [workflow.states])` |

---

## Node Operations

### Creating a node

`src/utils/workflowEditor/actions/states/createState.ts`

- Generate a unique `stateId`.
- Build `WorkflowState` (backend format) with `getDefaultConfig(type)`.
- Build `Node` (React Flow format) with `type: '${type}Node'`, `position`, and `data: { state }`.
- Push both to store: `workflowsStore.currentWorkflow.states` and `workflowsStore.editorState.nodes`.

### Updating a node

`src/utils/workflowEditor/actions/states/updateState.ts`

- `Object.assign(state, updates)` on the backend state object.
- Rebuild `node.data = { ...node.data, state }` — React Flow requires a new data reference to detect changes.

### Deleting a node

`src/utils/workflowEditor/actions/states/removeState.ts` — exports `removeStateAction`

Always remove connected edges before removing the node:
1. `getConnectedEdges(stateId)` → `deleteConnection(edge.id)` for each.
2. Filter node from `editorState.nodes`.
3. Filter state from `currentWorkflow.states`.

---

## Connection Operations

### Creating a connection

`src/utils/workflowEditor/actions/connections/createConnection.ts`

1. Validate (see validation rules below) — show `toaster.error` and return `null` if invalid.
2. Build `Edge` with `id: '${sourceId}-${targetId}'`, `source`, `target`, `sourceHandle`.
3. Set `sourceState.transitions[sourceHandle || 'default'] = targetId`.
4. Push edge to `editorState.edges`.

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

Test `validateConnection`, `serializeWorkflow`, `deserializeWorkflow`, and `autoLayout` as pure functions with no DOM dependency.

## Examples

### Laying out the graph

Source: `src/utils/workflowEditor/build/layout.ts::applyLayout`

```ts
export const applyLayout = (
  nodes: WorkflowNode[],
  edges: WorkflowEdge[]
): WorkflowNode[] => {
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