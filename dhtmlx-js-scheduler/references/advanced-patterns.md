# Advanced Patterns

Use this file when implementing Scheduler plugins, advanced views, resources, validation, undo/redo, or database schema.

## Contents
- Plugins
- Views And Resources
- Marked Timespans And Blocked Time
- Client-Side Event Filtering
- Custom Lightbox Sections
- Multiple Scheduler Instances On One Page (**PRO only**)
- Live Updates
- Validation And Collision Checks
- Undo/Redo With State Management
- Backend Schema Template
- Dynamic Loading

## Plugins

Many Scheduler features are gated behind `scheduler.plugins({...})`. Enable the extensions a feature needs before using the related APIs:

```ts
scheduler.plugins({
  recurring: true,
  tooltip: true,
  collision: true,
});
```

Enable only the plugins required by the feature being implemented. Verify plugin requirements with MCP when working with unfamiliar Scheduler APIs or views.

## Views And Resources

Built-in calendar views:
- `day`
- `week`
- `month`

Additional views:
- `year`
- `agenda`
- `week_agenda` (PRO)
- `timeline` (PRO)
- `units` (PRO)
- `grid` (PRO)
- `map`

Resource-based scheduling is typically implemented with Timeline and Units views.

### Timeline View (PRO)

Timeline is PRO-only.

```ts
scheduler.plugins({ timeline: true });

scheduler.createTimelineView({
  name: "timeline",
  x_unit: "minute",
  x_date: "%H:%i",
  x_step: 30,
  x_size: 24,
  x_start: 16,
  x_length: 48,
  y_unit: [
    { key: 1, label: "Room A" },
    { key: 2, label: "Room B" },
  ],
  y_property: "section_id",
  render: "bar",
});
```

Rules:
- event objects must include the mapped `y_property` field
- tree timeline requires the Tree Timeline extension
- days timeline requires the Days Timeline extension
- timeline-specific templates are created or overwritten by `createTimelineView`, so configure them after the view is created
- for dynamic sections loaded from a backend, use `scheduler.serverList(...)` and `scheduler.updateCollection(...)`

### Units View (PRO)

Units is PRO-only.

```ts
scheduler.createUnitsView({
  name: "unit",
  property: "unit_id",
  list: [
    { key: 1, label: "Room A" },
    { key: 2, label: "Room B" },
  ],
});
```

Rules:
- events must include the configured `property`, such as `unit_id`
- `list.key` values must match event property values
- for multi-section events, enable `multisection` (PRO) and align persistence with the selected data shape

## Marked Timespans And Blocked Time

Marked timespans paint background zones — working hours, day-off, holidays, breaks. They are **not events** and Scheduler does not recompute them when the view or date changes.

```ts
scheduler.addMarkedTimespan({
  days: 1,              // weekday 0-6 or a Date
  zones: "fullday",     // or flattened [startMinute, endMinute] pairs
  invert_zones: false,  // true marks everything EXCEPT the zones
  type: "dhx_time_block",
  css: "gray_section",
  html: "Day off",
  sections: { my_units_view: 1 }, // optional: limit to one Units/Timeline section
});

scheduler.deleteMarkedTimespan();             // remove all
scheduler.deleteMarkedTimespan({ days: 1 });  // remove filtered
```

Visual marking alone does not prevent edits. To actually block creating or moving events into a zone, use `scheduler.blockTime(...)` or the `limit` plugin (verify signatures with MCP):

```ts
scheduler.plugins({ limit: true });
```

Rules:
- Recompute and redraw timespans on `onViewChange` (and whenever availability data changes); call `deleteMarkedTimespan()` first so zones do not stack across renders.
- `invert_zones: true` is the idiom for "closed outside business hours".
- In Units/Timeline, scope a zone to one section via `sections: { <viewName>: sectionId }`.
- Verify exact option names with MCP before relying on them.

## Client-Side Event Filtering

Filter visible events per view by assigning a predicate to `scheduler.filter_<viewName>`:

```ts
scheduler.filter_week = (id, event) => event.resource_id === selectedResource;
scheduler.filter_month = scheduler.filter_week;
```

Scheduler dispatches to `scheduler["filter_" + currentView]`, so the suffix is the view name (`week`, `month`, a Units/Timeline view name, etc.).

Rules:
- After the filter state changes, re-apply with `scheduler.setCurrentView()` (or `scheduler.updateView()`). **Do not re-init or recreate the instance to refresh a filter** — see [known-failures.md](known-failures.md).
- The predicate is registered once and reads external state by closure; keep that state current (or read it through a ref/getter) so the filter sees fresh values.
- Confirm the exact `filter_<view>` suffix for custom/PRO views with MCP.

## Custom Lightbox Sections

Beyond the built-in section types (`textarea`, `select`, `time`, `recurring`, ...), register a custom control as a form block, then reference its name in `lightbox.sections`:

```ts
scheduler.form_blocks.my_control = {
  render: (section) => `<div class="dhx_cal_ltext">...</div>`,
  set_value: (node, value, ev) => { /* populate inputs from ev */ },
  get_value: (node, ev) => { /* write values back onto ev; return display value */ },
  focus: (node) => {},
};

scheduler.config.lightbox.sections = [
  { name: "client", type: "my_control", map_to: "client_id" },
  { name: "time", type: "time", map_to: "auto" },
];
```

Rules:
- `render` output is injected as **raw HTML** — escape any user-supplied data (same XSS rule as templates).
- `set_value` runs **every time the lightbox opens**, not once. Do not attach un-removed `document`/global listeners there — they accumulate per open. Remove the previous listener before re-adding, or scope listeners to the section node so they are discarded with the lightbox.
- Verify the `render`/`set_value`/`get_value`/`focus` contract and `map_to` behavior with MCP.

## Multiple Scheduler Instances On One Page

**PRO only** (Commercial since Oct 6 2021, Enterprise, Ultimate). Use the factory pattern:

```ts
const schedulerA = Scheduler.getSchedulerInstance();
const schedulerB = Scheduler.getSchedulerInstance();

schedulerA.init("schedulerA_here", new Date(2026, 4, 1), "week");
schedulerB.init("schedulerB_here", new Date(2026, 4, 1), "month");
```

Each instance has its own configuration, templates, events, data, and DataProcessor. Do not mix the global `scheduler` singleton and factory instances in the same module — wire each feature against a single chosen reference.

On non-PRO packages, fall back to a single instance.

## Live Updates (`scheduler.ext.liveUpdates`)

For real-time collaboration, server push, or any external change stream, use `scheduler.ext.liveUpdates`. The extension exposes `remoteUpdates` (apply incoming event change messages) and `RemoteEvents` (client for the documented multi-user backend protocol).

Both integration modes — the native multi-user backend protocol and custom change sources — are covered in [live-updates.md](live-updates.md).

`RemoteEvents` supports subscriptions and custom handlers, while `remoteUpdates` applies the documented `add-event`, `update-event`, and `delete-event` messages automatically.

## Validation And Collision Checks

Client-side validation is UX only. Backend validation is authoritative.

Blockable validation hooks:
- `onBeforeEventChanged`
- `onEventSave`
- `onBeforeEventCreated`

Post-change notification events:
- `onEventAdded`
- `onEventChanged`
- `onEventDeleted`

Collision control:

```ts
scheduler.plugins({ collision: true });
```

Use collision checks for immediate feedback, but still enforce no-overlap constraints in the backend when the product requires them.

## Undo/Redo With State Management

Scheduler has delete-specific undo UI behavior via `scheduler.config.undo_deleted`, but full undo/redo should be designed at the application state level.

Rules:
- decide whether undo/redo is Scheduler-delete-only, app-managed, or both
- if using app-level history, store serializable event snapshots, not live Scheduler objects
- normalize date values before storing snapshots
- ensure undo/redo updates persistence if state must survive reload
- avoid wiping app-level undo/redo history during save or refetch unless intended
- after app-managed undo/redo, reparse Scheduler from the current store snapshot

## Backend Schema Template

Recommended practical schema for a writable Scheduler-backed app with events and optional resources. This is not a DHTMLX-mandated contract. Adjust the field names and tables to fit the app and backend.

Events table:
- `id` - UUID or integer primary key
- `calendar_id` or `project_id` - recommended foreign key for multi-calendar apps
- `text` - event title/description
- `start_date` - timestamp, not null
- `end_date` - timestamp, not null
- custom resource fields such as `resource_id`, `room_id`, `unit_id`, or `section_id`
- `created_at`
- `updated_at`

Recurring fields when recurring is enabled:
- `rrule` - text, nullable
- `duration` - duration of each occurrence in seconds, nullable
- `recurring_event_id` - nullable parent series id
- `original_start` - timestamp of the original occurrence, nullable
- `deleted` - boolean, default false

Recommended constraints:
- `end_date > start_date` for normal events
- foreign keys for calendar/project/resource references
- self-reference from `recurring_event_id` to events
- application-specific no-overlap constraints when required

Recommended indexes:
- calendar/project id
- `start_date`
- `end_date`
- resource field(s)
- `recurring_event_id`

Recurring server logic:
- if a deleted occurrence is inserted, respond with `deleted` status
- when a series is modified, remove its modified/deleted exception rows
- when a series is deleted, delete its exception rows according to product behavior
- when an exception row is deleted, update it to `deleted = true` rather than physically removing it

## Dynamic Loading

For large datasets, enable dynamic loading after `scheduler.init(...)` and before `scheduler.load(...)`:

```ts
scheduler.config.show_loading = true;

scheduler.init("scheduler_here", new Date(2026, 4, 1), "month");
scheduler.setLoadMode("month");
scheduler.load("/api/events");
```

Valid load modes: `"day"`, `"week"`, `"month"`, `"year"`.

Scheduler will request URLs with `from` and `to` query parameters:

```text
/api/events?from=...&to=...
```

Use `scheduler.config.load_date` when the backend expects a specific date format for generated `from` and `to` values.
