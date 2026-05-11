# React Integration

Use this file when setting up or modifying the React wrapper integration.

## Installation

Trial:
```bash
npm install @dhtmlx/trial-react-gantt
```

Commercial (requires private registry setup and login):
```bash
npm config set @dhx:registry=https://npm.dhtmlx.com
npm login --registry=https://npm.dhtmlx.com --scope=@dhx
npm install @dhx/react-gantt
```
The user must generate credentials in the [Client's Area](https://dhtmlx.com/clients/) and run the registry config and login commands themselves before the package can be installed. See [installation docs](https://docs.dhtmlx.com/gantt/integrations/react/installation/) for details.

CSS import must match the installed package and must be a separate import line:

```ts
import "@dhtmlx/trial-react-gantt/dist/react-gantt.css";
```

or

```ts
import "@dhx/react-gantt/dist/react-gantt.css";
```

## Next.js

In Next.js (App Router), add `"use client"` at the top of the Gantt component file. Next.js uses Server Components by default, but React Gantt must render as a Client Component.

## Height Rule

The Gantt container must have explicit height.

Valid:
```tsx
<div style={{ height: "600px" }}>
  <Gantt ... />
</div>
```

## Documented Props

Use documented props only:

- `tasks`
- `links`
- `config`
- `templates`
- `theme`
- `plugins`
- `resources`
- `baselines`
- `markers`
- `calendars`
- `locale`
- `data`
- `customLightbox`
- `inlineEditors`
- `groupTasks`
- `filter`
- `resourceFilter`
- `modals`
- `ref`

## Ref Access

Use `ref` with `ReactGanttRef` when direct instance access is required:

```tsx
const ganttRef = useRef<ReactGanttRef>(null);
const gantt = ganttRef.current?.instance;
```

If you mutate data through the instance while also passing `tasks` and `links` as props, keep them synchronized. Otherwise React can overwrite those changes on re-render.

## Theme Rule

Use the app theme as the single source of truth.

Preferred built-in themes:
- `"terrace"`
- `"dark"`

Typical mapping:
```tsx
const ganttTheme = appTheme === "dark" ? "dark" : "terrace";
<Gantt theme={ganttTheme} />
```

Do not introduce separate Gantt-only theme state if the app already has a global theme source.

## Config And Templates

Wrap `config` and `templates` in `useMemo` when they contain functions.

Typical:
```tsx
const config: GanttConfig = useMemo(() => ({ ... }), [deps]);
const templates: GanttTemplates = useMemo(() => ({ ... }), [deps]);
```

Common documented config areas:
- `config.columns`
- `config.scales`
- `config.readonly`
- `config.drag_move`
- `config.drag_resize`
- `config.resource_store`
- `config.resource_property`
- `config.lightbox`
- `config.layout`

Do not guess template callback signatures. Verify with MCP if there is any doubt.

Example:
```tsx
timeline_cell_class: (task, date) => ...
```

## Resources And Calendars

When implementing resources:
- keep resource IDs and task assignment fields consistent
- derive display labels from the same persisted field used for editing
- include an explicit unassigned option when needed
- keep resource grid and timeline logic tied to the same source of truth

When implementing working time:
- keep one source of truth for working and non-working time rules
- pass calendars through the `calendars` prop
- use documented helpers or templates for non-working-day styling
- if worktime logic is complex, prefer wrapper hooks such as `useWorkTime(ganttRef)` over guessing engine behavior
