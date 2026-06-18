# JavaScript Setup

Use this file when initializing or configuring a core DHTMLX JavaScript Scheduler instance.

## Installation

Three packages are published. See [editions.md](editions.md) for the full Standard vs PRO feature matrix.

Standard (GPL, free, public npm):
```bash
npm install dhtmlx-scheduler
```

Professional Evaluation (Trial, full PRO surface, 30 days):
```bash
npm config set @dhx:registry=https://npm.dhtmlx.com
npm install @dhx/trial-scheduler
```

Professional (Commercial, private registry):
```bash
npm config set @dhx:registry=https://npm.dhtmlx.com
npm login --registry=https://npm.dhtmlx.com --scope=@dhx
npm install @dhx/scheduler
```

The user must generate credentials in the [Client's Area](https://dhtmlx.com/clients/) and run the registry config and login commands themselves before the Professional package can be installed. See [installation docs](https://docs.dhtmlx.com/scheduler/guides/installation/#npmevaluationandproversions) for details.

CSS must match the installed package and is imported on its own line. For all three packages the canonical CSS path is `<package>/codebase/dhtmlxscheduler.css`:

```ts
import "dhtmlx-scheduler/codebase/dhtmlxscheduler.css";
// or
import "@dhx/trial-scheduler/codebase/dhtmlxscheduler.css";
// or
import "@dhx/scheduler/codebase/dhtmlxscheduler.css";
```
## Imports

Use the import style that matches the installed package. Check `package.json`, existing imports, and lockfiles before changing package names.

Standard (GPL) package:

```ts
import { scheduler } from "dhtmlx-scheduler";
import "dhtmlx-scheduler/codebase/dhtmlxscheduler.css";
```

Trial package (default singleton usage):

```ts
import { scheduler } from "@dhx/trial-scheduler";
import "@dhx/trial-scheduler/codebase/dhtmlxscheduler.css";
```

Commercial package:

```ts
import { scheduler } from "@dhx/scheduler";
import "@dhx/scheduler/codebase/dhtmlxscheduler.css";
```

Instance factory pattern (used in some examples):

```ts
import { Scheduler } from "@dhx/trial-scheduler";
import "@dhx/trial-scheduler/codebase/dhtmlxscheduler.css";

const scheduler = Scheduler.getSchedulerInstance();
```

Running more than one Scheduler on the same page through `Scheduler.getSchedulerInstance()` is a PRO-only capability. See [editions.md](editions.md).

Do not mix singleton and factory-instance patterns in the same feature unless the feature explicitly manages multiple instances.

## Script Tag / CDN

Scheduler can also be loaded without a bundler — drop the JS file and matching CSS into the page:

```html
<link rel="stylesheet" href="https://cdn.dhtmlx.com/scheduler/edge/dhtmlxscheduler.css">
<script src="https://cdn.dhtmlx.com/scheduler/edge/dhtmlxscheduler.js"></script>
<div id="scheduler_here" style="width:100%; height:600px;"></div>
<script>
  scheduler.init("scheduler_here", new Date(), "week");
  scheduler.parse([
    {
      id: 1,
      text: "Meeting",
      start_date: "2026-05-11 14:00",
      end_date: "2026-05-11 15:00"
    }
  ]);
</script>
```

Globals exposed in this mode:
- `window.scheduler` — the singleton, present in every edition.
- `window.Scheduler` — the factory class, present only in PRO editions that support multiple instances. When `window.Scheduler` is `undefined`, the page is running Standard or a single-instance PRO build and `Scheduler.getSchedulerInstance()` is not available.

The same singleton-vs-factory rules apply: do not call `scheduler.destructor()` on the global singleton in Standard/single-instance builds unless the page will be reloaded (see Cleanup), and use `Scheduler.getSchedulerInstance()` only when the global `Scheduler` is present and the edition supports it.

See [editions.md](editions.md) for how to detect the edition when there is no `package.json` to inspect.

## Vite

Include the installed Scheduler package in `optimizeDeps` when Vite has trouble pre-bundling or the project already follows this pattern. Use the package name that matches the installed edition:

```ts
// vite.config.ts
export default {
  optimizeDeps: {
    include: ["dhtmlx-scheduler"],          // Standard
    // include: ["@dhx/trial-scheduler"],   // Trial
    // include: ["@dhx/scheduler"],         // Commercial
  },
};
```
## Height Rule

The Scheduler container must have a defined height:

```html
<div id="scheduler_here" style="height: 600px;"></div>
```

If using `height: 100%`, all parent elements must also have a defined height. Otherwise, the container will collapse.

## Initialization

Initialize Scheduler after the container exists:

```ts
scheduler.init("scheduler_here", new Date(), "week");
```

or:

```ts
scheduler.init(container, new Date(), "week");
```

Load client-side data with:

```ts
scheduler.parse(events);
```

Keep all Scheduler-related logic (initialization, configuration, data loading, events, DataProcessor, cleanup) in a single module. Avoid scattering direct Scheduler calls across the app.

## Plugins

Enable plugins before calling APIs that depend on them:

```ts
scheduler.plugins({
  recurring: true,
  tooltip: true,
  readonly: true,
});
```

In v6.0+, extensions are bundled in `dhtmlxscheduler.js` and activated through `scheduler.plugins({...})`; do not import old `ext/dhtmlxscheduler_*.js` files unless the project is on an older version.

## Markup vs Header Config

You can initialize with Scheduler markup:

```html
<div id="scheduler_here" class="dhx_cal_container">
  <div class="dhx_cal_navline">
    <div class="dhx_cal_prev_button">&nbsp;</div>
    <div class="dhx_cal_next_button">&nbsp;</div>
    <div class="dhx_cal_today_button"></div>
    <div class="dhx_cal_date"></div>
    <div class="dhx_cal_tab" data-tab="day"></div>
    <div class="dhx_cal_tab" data-tab="week"></div>
    <div class="dhx_cal_tab" data-tab="month"></div>
  </div>
  <div class="dhx_cal_header"></div>
  <div class="dhx_cal_data"></div>
</div>
```

For app layouts, prefer `scheduler.config.header` plus an empty container because it is easier to make responsive:

```ts
scheduler.config.header = ["day", "week", "month", "date", "prev", "today", "next"];
```

## Cleanup

Cleanup depends on whether the instance was created via the singleton or the factory. The two paths are not interchangeable.

**Factory instance (`Scheduler.getSchedulerInstance()`):** call `destructor()` when the container is removed. This clears data, removes event handlers, detaches the instance from the DOM, and disposes the attached DataProcessor. Safe because the factory created a dedicated instance.

```ts
const schedulerA = Scheduler.getSchedulerInstance();
// ...
schedulerA.destructor();
```

**Singleton (`import { scheduler }`):** **do not** call `scheduler.destructor()` for cleanup-and-reuse. The singleton is the only Scheduler instance available to the page, and destroying it leaves the module in a non-functional state until a full page reload. For reuse, clear data and detach the handlers that the init code attached:

```ts
// during init — capture handler ids
const handlerIds = [
  scheduler.attachEvent("onEventChanged", onEventChanged),
  scheduler.attachEvent("onEventDeleted", onEventDeleted),
];

// on cleanup
scheduler.clearAll();
for (const id of handlerIds) scheduler.detachEvent(id);
```

The DataProcessor attached to the singleton, however, is safe to destroy and recreate. Keep a reference to the DataProcessor returned by `scheduler.createDataProcessor(...)` and call `dataProcessor.destructor()` on cleanup; create a fresh DataProcessor when Scheduler is activated again:

```ts
// during init
const dataProcessor = scheduler.createDataProcessor(routerFn);

// on cleanup
dataProcessor.destructor();

// on reactivation
const dataProcessor = scheduler.createDataProcessor(routerFn);
```

Clearing `container.innerHTML` afterwards is rarely needed and only useful when the host element survives across re-mounts.

## Config And Templates

Common configuration areas:

- `scheduler.config.header` — navigation controls and visible views
- `scheduler.config.date_format` — event date parsing format
- `scheduler.config.first_hour`, `last_hour`, `time_step` — time scale and time resolution
- `scheduler.config.readonly`, `readonly_form` — interaction settings
- `scheduler.config.lightbox.sections` — event editing UI
- `scheduler.config.show_loading` — loading indicator behavior
- `scheduler.xy.*` — sizing values
- view-specific configuration for Timeline, Units, Grid, Agenda, Year, and other Scheduler views

Templates:

- `scheduler.templates.*` — formatting of event text, CSS classes, scale labels, tooltips, month cells, Timeline cells, Timeline rows, Units output, and view-specific rendering

Example:

```ts
// Event text
scheduler.templates.event_text = (start, end, event) => {
  return event.text;
};

// Event CSS class
scheduler.templates.event_class = (start, end, event) => {
  return event.status ? `status-${event.status}` : "";
};

// Tooltip text
scheduler.templates.tooltip_text = (start, end, event) => {
  return event.text;
};

// Highlight month cells, e.g. weekends
scheduler.templates.month_date_class = (date) => {
  const day = date.getDay();
  return day === 0 || day === 6 ? "weekend" : "";
};
```

Do not guess template signatures. Verify unfamiliar templates with MCP.

**Security — templates return raw HTML.** Whatever a template returns is inserted into the DOM as HTML, not as text. Any user-supplied field (`event.text`, descriptions, resource names, notes, free-form custom fields) that reaches a template unsanitized is an XSS vector — e.g. an event named `<img src=x onerror=alert(1)>` will execute on render.

Mitigate at both ends:

- Sanitize / escape user-supplied text on save in the backend (defense in depth).
- Escape — or sanitize via a vetted library like DOMPurify — before the data reaches Scheduler (`parse`, `addEvent`, `updateEvent`, `remoteUpdates`, DataProcessor responses), or escape inside the template before returning:

```ts
const escapeHtml = (value) => String(value ?? "").replace(/[&<>"']/g, (character) => ({
  "&": "&amp;",
  "<": "&lt;",
  ">": "&gt;",
  "\"": "&quot;",
  "'": "&#39;",
})[character]);

scheduler.templates.event_text = (_start, _end, event) => escapeHtml(event.text);
scheduler.templates.tooltip_text = (_start, _end, event) => `<b>${escapeHtml(event.text)}</b>`;
```

If the template intentionally returns markup, escape only the user-supplied substrings, not the static markup.

`scheduler.templates.parse_date` and `scheduler.templates.format_date` can be overridden to accept or emit non-default date formats (useful when the backend returns timestamps or compact strings):

```ts
scheduler.config.date_format = "%Y-%m-%d %H:%i";

const parseDate = scheduler.date.str_to_date(scheduler.config.date_format);
const formatDate = scheduler.date.date_to_str(scheduler.config.date_format);

scheduler.templates.parse_date = (value) => parseDate(value);
scheduler.templates.format_date = (date) => formatDate(date);
```

## Events

Wire app logic through documented events rather than direct mutation. Attach with `scheduler.attachEvent` and detach (or destroy via `scheduler.destructor`) on unmount:

```ts
const handlerId = scheduler.attachEvent("onEventChanged", (id, event) => {
  // app-side reactions, persistence triggers, store updates
});

// later
scheduler.detachEvent(handlerId);
```

For one-shot subscriptions (e.g. running first-init startup code through `onSchedulerReady`), pass `{ once: true }` so Scheduler auto-detaches the handler after the first invocation:

```ts
scheduler.attachEvent("onSchedulerReady", () => {
  // runs once, even if scheduler.init is called again later
}, { once: true });
```

Event names and argument lists are stable but numerous. Verify unfamiliar events with MCP before wiring them.

## Localization

Switch the active locale at runtime with `scheduler.i18n.setLocale("xx")`. Per-label overrides go through `scheduler.locale.labels.*` or a partial locale object passed to `scheduler.i18n.setLocale(...)`:

```ts
scheduler.i18n.setLocale("ru");
scheduler.locale.labels.day_tab = "Day";
scheduler.locale.labels.week_tab = "Week";
scheduler.locale.labels.section_description = "Description";
```

Or:

```ts
scheduler.i18n.setLocale({
  labels: {
    day_tab: "Day",
    week_tab: "Week",
    section_description: "Description",
  },
});
```

Apply locale changes before `scheduler.init` when possible. If changed after init, call `scheduler.render()`.

## Read-Only Mode

To make the Scheduler non-editable:

```ts
scheduler.config.readonly = true;
```

This makes the Scheduler non-editable and prevents users from opening the lightbox.

To allow opening the lightbox while preventing edits inside it:

```ts
scheduler.config.readonly_form = true;
```

This requires the `readonly` extension.

Read-only behavior can also be applied to individual events:

```ts
const event = scheduler.getEvent(id);
event.readonly = true;
```

Note that read-only mode is a UI restriction, not a security boundary. Also remove or hide app-level controls that mutate data (e.g. add buttons, undo/redo) and enforce permissions in DataProcessor routes and on the backend.
