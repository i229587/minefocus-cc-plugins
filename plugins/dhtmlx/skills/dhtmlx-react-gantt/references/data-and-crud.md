# Data And CRUD

Use this file when tasks, links, state ownership, or persistence are involved.

## Data Model Essentials

Prefer JavaScript `Date` objects for task date fields in React-managed mode.

Minimum task shape:
```ts
interface Task {
  id: string | number;
  text: string;
  start_date: Date;
  duration: number;
  progress: number;
  parent: string | number;
  type?: "task" | "project" | "milestone";
  open?: boolean;
  end_date?: Date;
}
```

Minimum link shape:
```ts
interface Link {
  id: string | number;
  source: string | number;
  target: string | number;
  type: "0" | "1" | "2" | "3";
}
```

When loading from a backend, map rows explicitly into Gantt task/link objects. Do not pass raw DB rows straight through.

## Choose The Ownership Model First

For most React apps, use React-managed data:
- keep `tasks` and `links` in React state or a state manager
- pass them into `<Gantt />`
- handle updates with `data.save` or `data.batchSave`

Use Gantt-managed data only when:
- datasets are very large
- mass updates are frequent
- React does not need to react to every edit
- the page is primarily Gantt-centric

## CRUD Default

For normal CRUD, prefer `data.save`.

Pattern:
```tsx
<Gantt
  tasks={tasks}
  links={links}
  data={{
    save: async (entity, action, item, id) => {
      // update local state
      // sync backend
      // return { id } after creates when backend assigns a real ID
    },
  }}
/>
```

Strict rules:
- update state from the latest current snapshot, not stale closure state
- for `create`, if the backend assigns a real ID, return `{ id: realId }`
- for `update`, persist only normalized values
- for `delete`, remove from local state and persistence layer
- normalize date values before serialization
- if using temporary IDs, replace them deterministically after persistence
- persist a link only when both `source` and `target` already refer to real persisted task IDs
- if a task still has a temporary client-side ID, defer or reject link persistence until the real task ID replacement flow completes
- do not send links that reference temporary IDs to the backend
- build backend payloads explicitly from normalized task/link models, not from raw callback objects

## Date Formatting

Define `format_date` and `parse_date` templates to ensure dates always round-trip as ISO strings in `data.save` and `data.batchSave` callbacks:

```tsx
const templates: ReactGanttProps['templates'] = useMemo(
  () => ({
    format_date: (d) => d.toISOString(),
    parse_date: (s) => new Date(s),
  }),
  []
);
```

Pass these templates to the `<Gantt>` component. This guarantees that date values in persistence callbacks are always ISO-formatted strings regardless of input shape.

## When To Use batchSave

Use `data.batchSave` when:
- heavy operations can emit many changes
- bulk backend sync is more appropriate than single-entity persistence
- you need grouped change handling rather than one callback per edit

Do not switch to `batchSave` casually. Verify the exact expected behavior with MCP first.

## Persistence Guardrails

- Do not assume task updates always contain real `Date` instances.
- Normalize strings and dates before serializing to the backend.
- Keep frontend `duration_unit`, backend payloads, and DB schema aligned: `duration` should be treated as an integer count of the chosen storage unit.
- Do not persist raw Gantt callback objects directly.
- If IDs are temporary on create, make the ID replacement flow explicit and deterministic.
- If the app introduces undo/redo, persistence rules must remain correct after history actions.
