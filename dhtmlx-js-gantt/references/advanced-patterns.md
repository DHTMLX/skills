# Advanced Patterns

Use this file when implementing plugins, row reorder, resource panels, baselines, critical path, undo/redo, working calendars, zoom, multi-instance pages, or designing a backend schema for Gantt data.

## Contents
- Plugins And Extensions Loader
- Row Reorder With Sortorder Persistence
- Resource Panel And Workload Visualization (**PRO only**)
- Live Updates (`gantt.ext.liveUpdates`) (all editions)
- Working Calendar (**not in MIT Community**)
- Zoom
- Baselines (**PRO only**)
- Critical Path (**PRO only**)
- Multiple Gantt Instances On One Page (**MIT Community v10+ and PRO**)
- Undo/Redo With State Management (built-in undo **not in MIT Community**)
- Backend Schema Template

## Plugins And Extensions Loader

Many `gantt.ext.*` surfaces are gated behind `gantt.plugins({...})`. Enable the extensions a feature needs before `gantt.init`:

```ts
gantt.plugins({
  critical_path: true,    // PRO only — enables gantt.config.highlight_critical_path
  marker: true,           // gantt.ext.marker (timeline markers / today line) — Standard/GPL + PRO; no-op in MIT Community
  tooltip: true,          // gantt.ext.tooltips — all editions
  keyboard_navigation: true,
  undo: true,             // gantt.ext.undo — Standard/GPL + PRO; no-op in MIT Community
  multiselect: true,      // gantt.ext.multiselect — Standard/GPL + PRO; no-op in MIT Community
  export_api: true,       // gantt.ext.export_api (PDF / PNG / Excel / MS Project) — all editions
  auto_scheduling: true,  // PRO only
});
```

Built-in zoom (`gantt.ext.zoom`) does not require a plugin enable in current versions. Verify with MCP if the extension is unfamiliar.

In the **MIT Community edition (v10+)** the `undo`, `marker`, and `multiselect` extensions are not bundled — `gantt.plugins({...})` accepts the keys silently but the features are no-ops, and the matching `gantt.ext.undo` / `gantt.ext.marker` / `gantt.ext.multiselect` surfaces (and `gantt.undo` / `gantt.addMarker` / `gantt.eachSelectedTask`) are `undefined`. They are available in the legacy GPL Standard build (v9.x) and in PRO. See [editions.md](editions.md).

## Row Reorder With Sortorder Persistence

- enable row ordering through documented Gantt config
- documented reorder events: `onRowDragStart`, `onBeforeRowDragMove`, `onBeforeRowDragEnd`, `onRowDragEnd` — verify exact signatures with MCP before wiring handlers
- treat reorder as a dedicated flow
- do not route reorder through the normal single-task update path
- when a row moves, rebuild the full ordered task list in memory
- recompute `sortorder` for all affected tasks, not only the moved task
- persist the full reordered set
- if hierarchy changes during the move, persist `parent_id` alongside `sortorder`
- normal task updates must preserve current `sortorder` unless a dedicated reorder flow changes it
- load tasks with `ORDER BY sortorder` from the backend
- keep the reorder implementation targeted; do not refactor unrelated CRUD paths during this change
- if DataProcessor also marks the moved task as updated, suppress that single update when the batch reorder already persists sortorder and parent
- ignore reorder persistence for temporary task ids

## Resource Panel And Workload Visualization

**PRO only.** Resource management (resource grid, resource timeline, resource histogram, assignments, workload) is unavailable in both free editions of `dhtmlx-gantt` (MIT Community and GPL Standard). On a free install, warn the user before scaffolding and still produce the requested code — see [editions.md](editions.md).

- Gantt does not enforce a fixed task field for resource binding
- the relation is defined by `gantt.config.resource_property`, and resource features use `task[gantt.config.resource_property]`

Guardrails:

- use one consistent resource model across the feature
- do not mix task fields and assignment datasets
- ensure UI, Gantt config, and persistence use the same field or assignment structure

Product integration:

- build the resource list from real project data
- include an "Unassigned" option only if needed
- ensure lightbox selection maps to the same field or assignment structure
- if using assignments, drive both UI and persistence from the assignments dataset
- format workload values according to product rules (e.g. hours)

Layout:

- include `resourceGrid` and `resourceTimeline` or `resourceHistogram` when needed
- use shared horizontal scrolling for combined views

Notes:

- resource datastore may require initialization after `gantt.init`
- the resource and assignment datasets live in dedicated datastores reachable as `gantt.$data.resourcesStore` and `gantt.$data.assignmentsStore`. Use them for wholesale or incremental updates from external sources:

```ts
// wholesale replace
gantt.$data.resourcesStore.clearAll();
gantt.$data.resourcesStore.parse(resources);

// incremental sync — wrap in gantt.silent to avoid notifying the DataProcessor
gantt.silent(() => {
  for (const a of addedAssignments)   gantt.$data.assignmentsStore.addItem(a);
  for (const a of updatedAssignments) gantt.$data.assignmentsStore.updateItem(a.id, a);
  for (const id of removedIds)        if (gantt.$data.assignmentsStore.exists(id)) gantt.$data.assignmentsStore.removeItem(id);
});
```

- `gantt.serverList(name, list)` is the canonical way to register named lookup lists used by lightbox `select` sections and resource dropdowns. Register lists *before* `gantt.init` and reference them in the lightbox config:

```ts
gantt.serverList("staff", [
  { key: 1, label: "John" },
  { key: 2, label: "Mike" },
]);

gantt.config.lightbox.sections = [
  { name: "description", height: 70, map_to: "text", type: "textarea" },
  { name: "owner", map_to: "owner_id", type: "select", options: gantt.serverList("staff") },
  { name: "time", type: "duration", map_to: "auto" },
];

gantt.init("gantt_here");
```

`gantt.serverList(name)` returns a live reference to the same array, so mutations propagate to any UI that reads it (after `gantt.refreshData()` or `gantt.render()`).

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
  assignments: [{ id: 1, task_id: 5, resource_id: 2, value: 8 }]
}
```

## Live Updates (`gantt.ext.liveUpdates`)

For real-time collaboration, server push, or any external change stream, use `gantt.ext.liveUpdates`. The extension exposes `remoteUpdates` (apply task/link change messages without notifying the DataProcessor) and `RemoteEvents` (WebSocket client speaking the documented multi-user backend protocol). Works in Standard and PRO.

Both integration modes — native multi-user backend and custom change source like Firestore or an app socket — and the echo-loop guard are covered in [live-updates.md](live-updates.md).

## Working Calendar

**Not available in the MIT Community edition (v10+).** Working-time calendars are excluded from MIT: `gantt.config.work_time` and `gantt.setWorkTime(...)` are **hard no-ops** there — every hour counts as working time regardless of config, durations never skip weekends/non-working time, and per-task/multiple calendars do nothing. The APIs still exist (so calls don't throw), but they have no effect. Working calendars are available in the legacy GPL Standard build (v9.x) and in PRO (calendar-assignment to project/resource is PRO-only). On a MIT install, warn the user before scaffolding calendar logic, then either scaffold it anyway (no effect) or implement weekend/non-working styling and date math in app code via templates. See [editions.md](editions.md).

- keep one source of truth for working and non-working time rules
- configure working time through documented Gantt APIs
- call `gantt.setWorkTime(...)` only with verified arguments for the installed version
- use templates for calendar-driven cell styling when appropriate
- re-render Gantt after changing calendar rules

Common pattern:

```ts
gantt.config.work_time = true;
gantt.config.correct_work_time = true;
gantt.setWorkTime({ day: 0, hours: false });
gantt.setWorkTime({ day: 6, hours: false });
```

Calendar-driven styling should use templates rather than global date math in CSS:

```ts
gantt.templates.timeline_cell_class = (_task, date) =>
  isNonWorking(date) ? "weekend-cell" : "";
gantt.templates.scale_cell_class = (date) =>
  isNonWorking(date) ? "weekend-cell" : "";
```

## Zoom

Use the built-in zoom extension for multi-level zoom. Each level is a full configuration block — `name`, `scale_height`, `min_column_width`, and a `scales` array of `{ unit, step, format }`:

```ts
gantt.ext.zoom.init({
  levels: [
    {
      name: "day",
      scale_height: 50,
      min_column_width: 80,
      scales: [
        { unit: "week", step: 1, format: "Week #%W" },
        { unit: "day",  step: 1, format: "%d %M" },
      ],
    },
    {
      name: "month",
      scale_height: 50,
      min_column_width: 120,
      scales: [
        { unit: "year",  step: 1, format: "%Y" },
        { unit: "month", step: 1, format: "%F" },
      ],
    },
  ],
});

gantt.ext.zoom.setLevel("day");
```

Common zoom level names: `hour`, `day`, `week`, `month`, `year`.

For manual zoom (without the extension):
- update `gantt.config.scales`
- call `gantt.render()` after changes


## Baselines

**PRO only.** Baselines, deadlines, and similar custom timeline elements are PRO features — unavailable in both free editions (MIT Community and GPL Standard). On a free install, warn the user before scaffolding and still produce the requested code — see [editions.md](editions.md).

Inbuilt baselines (added in v9.0) render planned-vs-actual bars alongside each task automatically once enabled and loaded — no custom render template is required.

**Data shape.** Baselines are loaded as a **top-level** array in `gantt.parse(...)`, alongside `tasks` and `links`. Each entry has its own `id` and references the task via `task_id`:

```ts
gantt.parse({
  tasks: [
    { id: 1, text: "Task #1", start_date: "2026-05-04", duration: 5, parent: 0 },
    { id: 2, text: "Task #2", start_date: "2026-05-10", duration: 3, parent: 0 },
  ],
  links: [],
  baselines: [
    { id: 1, task_id: 1, start_date: "2026-05-03", duration: 4, end_date: "2026-05-07" },
    { id: 2, task_id: 2, start_date: "2026-05-09", duration: 4, end_date: "2026-05-13" },
  ],
});
```

Baselines live in their own datastore named `"baselines"` and are reachable via `gantt.getDatastore("baselines")`.

**Config:**

```ts
gantt.config.baselines = {
  datastore: "baselines",      // name of the baseline datastore
  render_mode: "separateRow",  // "taskRow" | "separateRow" | "individualRow" | false (disable)
  dataprocessor_baselines: false, // true → baseline writes flow through the DataProcessor as separate entries
  row_height: 16,              // height of the baseline subrow (used by separateRow / individualRow)
  bar_height: 8,               // height of the baseline bar
};
```

The three render modes:
- `"taskRow"` — baseline drawn in the same row as the task bar.
- `"separateRow"` — all baselines for a task drawn together in a subrow below the task, expanding the row height.
- `"individualRow"` — each baseline in its own subrow.

`gantt.config.baselines = false` disables the feature entirely.

For the full surface (lightbox `{type: "baselines"}` section, `gantt.getTaskBaselines(id)`, `gantt.adjustTaskHeightForBaselines(task)` after dynamic config changes, the `baseline_text` template, and the `addTaskLayer` custom-rendering alternative) see <https://docs.dhtmlx.com/gantt/guides/inbuilt-baselines/>.

## Critical Path

**PRO only.** Critical path calculation (and total/free slack) is a PRO feature, gated through `gantt.plugins({ critical_path: true })` — unavailable in both free editions (MIT Community and GPL Standard). On a free install, warn the user and still produce code.

```ts
gantt.plugins({ critical_path: true });
gantt.config.highlight_critical_path = true;
```

Critical bars and links pick up `gantt_critical_task` / `gantt_critical_link` classes. Avoid combining critical highlighting with inline color shortcut fields (`task.color`, `link.color`) — inline styles will override the critical-path class.

## Multiple Gantt Instances On One Page

Available in the **MIT Community edition (v10+)** and **PRO** (Commercial since Oct 6 2021, Enterprise, Ultimate). Not available in the legacy GPL Standard build (v9.x). Use the factory pattern, importing from whichever package is installed:

```ts
import { Gantt } from "dhtmlx-gantt"; // MIT Community v10+ (or "@dhx/gantt" / "@dhx/trial-gantt" for PRO)
const ganttA = Gantt.getGanttInstance();
const ganttB = Gantt.getGanttInstance();

ganttA.init("ganttA_here");
ganttB.init("ganttB_here");
```

Each instance has its own `config`, `templates`, events, and DataProcessor. Do not import the singleton `gantt` and a factory instance in the same module — wire each feature against a single chosen reference.

On the legacy GPL Standard build (v9.x), `Gantt` is not exported — fall back to a single instance. See [editions.md](editions.md).

## Undo/Redo With State Management

The built-in `gantt.ext.undo` extension (`gantt.undo()` / `gantt.redo()`, `gantt.plugins({ undo: true })`) is available in the **legacy GPL Standard build (v9.x)** and in **PRO**, but **not in the MIT Community edition (v10+)** — there the plugin is a no-op and `gantt.undo` is `undefined`. On a MIT install, either warn the user that built-in undo/redo was dropped from the Community edition (PRO has it), or implement app-managed undo/redo as below. See [editions.md](editions.md).

- decide whether undo/redo is Gantt-managed, app-managed, or both
- if using an app store, snapshot normalized task and link data only
- ensure undo/redo updates persistence if state must survive reload
- avoid wiping the undo/redo stack during save or refetch unless intended
- after app-managed undo/redo, reparse Gantt from the current store snapshot

## Backend Schema Template

Recommended practical schema for a writable Gantt-backed app with tasks, links, and optional resources. This is not a DHTMLX-mandated contract. Adjust the field names and tables to fit the app and backend.

Tasks table:
- `id` - UUID, primary key
- ID type must be consistent across tasks and links
- `project_id` - strongly recommended foreign key to a `projects` table for multi-project apps
- `text` - task name
- `start_date` - timestamp (nullable for projects that derive dates from children)
- `duration` - integer in the configured project duration unit (nullable, >= 0 when set)
- `progress` - numeric, constrained between 0 and 1
- `parent_id` - self-referencing foreign key to tasks (must not equal `id`), cascade on delete
- use a consistent root value for top-level tasks (e.g. `0` or `null`)
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
