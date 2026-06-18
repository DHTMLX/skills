# Advanced Patterns

Use this file when implementing row reorder, resource panels, custom editors, undo/redo, or designing a backend schema for Gantt data.

## Contents
- Row Reorder With Sortorder Persistence
- Resource Panel And Workload Visualization
- Vue Customization Hooks
- Undo/Redo With State Management
- Backend Schema Template

## Row Reorder With Sortorder Persistence

- enable row ordering via Gantt config (`order_branch`)
- enable cross-level moves only when needed (`order_branch_free`)
- treat reorder as a dedicated flow - do not route it through normal single-task CRUD
- use DHTMLX reorder target semantics:
  - `targetId` means insert before `targetId`
  - `next:targetId` means insert after `targetId`
- prefer `events.onRowDragEnd(id, target)` as the reorder boundary when available
- if using DataProcessor/`data.save`, only use `target` as the reorder signal after verifying the wrapper payload with MCP
- do not combine `getVisibleTaskCount()` with `getTaskByIndex()` to build persisted order
- rebuild canonical order from application-managed task state
- recompute `sortorder` for all affected tasks
- persist only changed rows, scoped by project/schedule
- persist `parent_id` when cross-level moves are enabled
- load tasks with `ORDER BY sortorder`
- guard local state from refetch/realtime overwrites while reorder persistence is in flight
- use an atomic backend/RPC flow for large or server-owned schedules when possible

## Resource Panel And Workload Visualization

- Gantt does not enforce one standard task field for resource binding
- the relation is defined by `config.resource_property`, and built-in resource features watch `task[gantt.config.resource_property]`
- for a single-resource model, choose any task field that fits the app schema, for example `owner_id` or `user_id`, then set `config.resource_property` to match it
- for a multi-resource model, the field named by `config.resource_property` can store either an array of resource IDs or an array of assignment objects
- for advanced flows, resource assignments can also be loaded separately through the `resourceAssignments` prop
- build the resource list dynamically from real project data, not from a hardcoded array
- include an explicit "Unassigned" option only when the product actually needs one
- if using a single-resource model, resource grid display, timeline display, and lightbox editing should all read and write the same task field
- if using multi-resource assignments, keep the UI and persistence model aligned to the same assignment structure
- if using separate assignments, build task/resource views and save flows from the assignments dataset rather than inventing a display-only task field
- display workload in the resource grid as formatted values such as `"8h"`, `"16h"`, `"24h"` when the product uses hours
- wire lightbox resource selection so it maps back to the same field or assignment structure used by persistence
- layout must include both main Gantt rows (grid + timeline) and resource panel rows (resourceGrid + resourceTimeline or resourceHistogram) with shared horizontal scrolling when the product needs a combined view
- verify `resource_cell_value`, `resource_cell_class`, histogram templates, and resource timeline callback signatures with MCP before implementing

### Supported Resource Binding Models

Single-resource field on task:

```ts
config.resource_property = "user_id";
// task.user_id <-> resource.id
```

Multiple resource IDs on task:

```ts
config.resource_property = "users";
// task.users = [2, 3]
```

Assignment objects on task:

```ts
config.resource_property = "users";
// task.users = [{ resource_id: 2, value: 8 }, { resource_id: 3, value: 4 }]
```

Separate assignments list:

```ts
{
  tasks: [...],
  links: [...],
  resources: [...],
  resourceAssignments: [{ id: 1, task_id: 5, resource_id: 2, value: 8 }]
}
```

## Vue Customization Hooks

Use the lowest-cost customization layer that solves the task:

1. `config` and `templates` for visual output and built-in behavior changes
2. `theme` and `locale` for chart-wide skin and language switches
3. Vue VNodes from `h()` inside template functions when template output needs interactive or composable Vue UI
4. `events` and `@ready` for interaction rules and startup orchestration
5. `modals` for delete-confirmation control
6. `customLightbox` for an application-specific task editor in a modal
7. `events.onBeforeLightbox` returning `false` plus Vue Router for full-page task editors
8. `inlineEditors` for custom grid cell editors

### Replace The Task Form

Use `customLightbox` when the built-in lightbox is not enough and you need a Vue component for task editing.

Your component receives:
- `data`
- `onSave(updatedTask)`
- `onCancel()`
- `onDelete()`
- `ganttInstance`

Save and delete flows should update the same source of truth used by the chart.

### Custom Grid Inline Editors

Use `inlineEditors` when grid cells need custom editing widgets.

The `config.columns[].editor.type` string must match the key in the `inlineEditors` map.

### Custom Delete Confirmation Modals

Use `modals` to replace built-in task/link deletion confirmations.

The handler receives a `callback`. Deletion only proceeds when you invoke `callback()`. Capture the callback in Vue state when using an async custom dialog.

### Replace The Lightbox With A Route

Use `onBeforeLightbox(taskId)` returning `false` when the task editor belongs in a separate route. For newly created tasks, use `onTaskCreated(task)`, stage the task in the same store the chart reads from, navigate, and return `false`.

## Undo/Redo With State Management

- use current state references inside `data.save` callbacks - do not rely on stale closures
- if using an external store for Gantt history, normalize all date values before writing to the store
- do not enable Gantt's undo plugin and a separate store-level undo model for the same records unless synchronization is explicitly designed
- undo and redo actions must also update persistence - client-only history creates data gaps on reload
- persist history snapshots when rehydration across sessions is required
- save or refetch operations must not wipe the undo/redo history stack
- keep history state and application state in sync after every mutation

## Backend Schema Template

Recommended practical schema for a writable Gantt-backed app with tasks, links, and optional resources. This is not a DHTMLX-mandated contract. Adjust the field names and tables to fit the app and backend.

Tasks table:
- `id` - UUID, primary key
- `project_id` - strongly recommended foreign key to a `projects` table for multi-project apps
- `text` - task name
- `start_date` - timestamp (nullable for projects that derive dates from children)
- `duration` - integer in the configured project duration unit (nullable, >= 0 when set)
- `progress` - numeric, constrained between 0 and 1
- `parent_id` - self-referencing foreign key to tasks (must not equal `id`), cascade on delete
- `sortorder` - strongly recommended integer field for persistent row ordering
- `type` - one of `task`, `project`, `milestone`
- resource relation:
  - either a task field named to match `config.resource_property`
  - or no task field at all when the app uses separate resource assignments

Duration storage guardrail:
- treat `duration_unit` as a project-level storage decision, not just a display preference
- choose the smallest unit the product must support at project start, because changing from `day` to `hour` or `minute` later usually requires conversion of persisted task durations
- stored durations are integers, so `duration_unit="day"` supports whole-day tasks only and cannot represent a plain 4-hour task as `duration: 4`
- use `duration_unit="hour"` when tasks like 4 hours must be stored directly, and `duration_unit="minute"` when the product needs minute-level precision

Links table:
- `id` - UUID, primary key
- `project_id` - strongly recommended foreign key to a `projects` table for multi-project apps
- `source` - foreign key to task, cascade on delete
- `target` - foreign key to task, cascade on delete (must not equal `source`)
- `type` - one of `0`, `1`, `2`, `3`

Recommended constraints:
- unique constraint on `(project_id, source, target, type)` for links
- cascade on delete for project foreign keys in both tables

Recommended indexes:
- `project_id` on both tables
- `parent_id` on tasks
- `source` and `target` on links
