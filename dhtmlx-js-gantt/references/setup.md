# JavaScript Setup

Use this file when initializing or configuring a core DHTMLX JavaScript Gantt instance.

## Installation

Two public packages plus a private PRO package are published. The free `dhtmlx-gantt` is the **MIT Community edition at v10+** and the legacy **GPL Standard build at v9.x** — same package name, different edition by version. See [editions.md](editions.md) for the detection procedure and the per-edition feature matrix.

Community (MIT, free, public npm — v10+; v9.x of the same package is GPL Standard):
```bash
npm install dhtmlx-gantt
```

Professional Evaluation (Trial, full PRO surface, 30 days):
```bash
npm config set @dhx:registry=https://npm.dhtmlx.com
npm install @dhx/trial-gantt
```

Professional (Commercial, private registry):
```bash
npm config set @dhx:registry=https://npm.dhtmlx.com
npm login --registry=https://npm.dhtmlx.com --scope=@dhx
npm install @dhx/gantt
```
The user must generate credentials in the [Client's Area](https://dhtmlx.com/clients/) and run the registry config and login commands themselves before the Professional package can be installed. See [installation docs](https://docs.dhtmlx.com/gantt/guides/installation/#npmevaluationandproversions) for details.

CSS must match the installed package and is imported on its own line. For all three packages the canonical CSS path is `<package>/codebase/dhtmlxgantt.css`:

```ts
import "dhtmlx-gantt/codebase/dhtmlxgantt.css";
// or
import "@dhx/trial-gantt/codebase/dhtmlxgantt.css";
// or
import "@dhx/gantt/codebase/dhtmlxgantt.css";
```

## Imports

Use the import style that matches the installed package. Check `package.json`, existing imports, and lockfiles before changing package names.

Community (MIT, v10+) / Standard (GPL, v9.x) package — same import, edition determined by version:

```ts
import { gantt } from "dhtmlx-gantt";
import "dhtmlx-gantt/codebase/dhtmlxgantt.css";
```

In the MIT Community edition (v10+) the `Gantt` factory is also exported for multiple instances: `import { gantt, Gantt } from "dhtmlx-gantt"`.

Trial package (default singleton usage):

```ts
import { gantt } from "@dhx/trial-gantt";
import "@dhx/trial-gantt/codebase/dhtmlxgantt.css";
```

Commercial package:

```ts
import { gantt } from "@dhx/gantt";
import "@dhx/gantt/codebase/dhtmlxgantt.css";
```

Instance factory pattern (used in some examples):

```ts
import { Gantt } from "@dhx/trial-gantt";
import "@dhx/trial-gantt/codebase/dhtmlxgantt.css";

const gantt = Gantt.getGanttInstance();
```

Running more than one Gantt on the same page through `Gantt.getGanttInstance()` is available in the **MIT Community edition (v10+)** and PRO (Commercial since Oct 6 2021, Enterprise, Ultimate). It is **not** available in the legacy GPL Standard build (v9.x), where `Gantt` is not exported. See [editions.md](editions.md).

Do not mix singleton and factory-instance patterns in the same feature unless the feature explicitly manages multiple instances.

## Script Tag / CDN

Gantt can also be loaded without a bundler — drop the JS file and matching CSS into the page:

```html
<link rel="stylesheet" href="https://cdn.dhtmlx.com/gantt/edge/dhtmlxgantt.css">
<script src="https://cdn.dhtmlx.com/gantt/edge/dhtmlxgantt.js"></script>
<div id="gantt_here" style="width:100%; height:600px;"></div>
<script>
  gantt.init("gantt_here");
  gantt.parse({ tasks: [...], links: [...] });
</script>
```

Globals exposed in this mode:
- `window.gantt` — the singleton, present in every edition.
- `window.Gantt` — the factory class, present in the MIT Community edition (v10+) and in PRO editions that support multiple instances (Commercial since Oct 6 2021, Enterprise, Ultimate). When `window.Gantt` is `undefined`, the page is running the legacy GPL Standard build (v9.x) or a single-instance PRO build, and `Gantt.getGanttInstance()` is not available.

The same singleton-vs-factory rules apply: do not call `gantt.destructor()` on the global singleton (see Cleanup), and use `new Gantt()` / `Gantt.getGanttInstance()` only when the global is present and the edition supports it.

See [editions.md](editions.md) for how to detect the edition when there is no `package.json` to inspect.

## Vite

Include the installed Gantt package in `optimizeDeps` when Vite has trouble pre-bundling or the project already follows this pattern. Use the package name that matches the installed edition:

```ts
// vite.config.ts
export default {
  optimizeDeps: {
    include: ["dhtmlx-gantt"],          // Standard
    // include: ["@dhx/trial-gantt"],   // Trial
    // include: ["@dhx/gantt"],         // Commercial
  },
};
```

## Height Rule

The Gantt container must have a defined height:

```html
<div id="gantt_here" style="height: 600px;"></div>
```

If using `height: 100%`, all parent elements must also have a defined height. Otherwise, the container will collapse.

## Initialization

Initialize Gantt after the container exists:

```ts
gantt.init("gantt_here");
```

or:

```ts
gantt.init(container);
```

Load client-side data with:

```ts
gantt.parse({ tasks, links });
```

`gantt.init` is safe to call more than once on the same instance and is in fact the documented way to apply a new `gantt.config.layout` — the layout is built at init time and is not picked up by `gantt.render()` alone. Re-init reuses existing event handlers and the existing DataProcessor; do not re-run event attachment or `gantt.createDataProcessor(...)` alongside it, or handlers and DataProcessors will stack up. Split the startup code into a one-time setup phase (events, DataProcessor, lookup lists) and a re-runnable configure-and-init phase (configs + `gantt.init`).

Keep all Gantt-related logic (initialization, configuration, data loading, events, DataProcessor, cleanup) in a single module. Avoid scattering direct Gantt calls across the app.

## Cleanup

Cleanup depends on whether the instance was created via the singleton or the factory. The two paths are not interchangeable.

**Factory instance (`Gantt.getGanttInstance()`):** call `destructor()` when the container is removed. This clears data, removes event handlers, detaches the instance from the DOM, and disposes the attached DataProcessor. Safe because the factory created a dedicated instance.

```ts
const ganttA = Gantt.getGanttInstance();
// ...
ganttA.destructor();
```

**Singleton (`import { gantt }`):** **do not** call `gantt.destructor()` for cleanup-and-reuse. The singleton is the only Gantt instance available to the page, and destroying it leaves the module in a non-functional state until a full page reload. For reuse, clear data and detach the handlers that the init code attached:

```ts
// during init — capture handler ids
const handlerIds = [
  gantt.attachEvent("onAfterTaskUpdate", onTaskUpdate),
  gantt.attachEvent("onAfterTaskDelete", onTaskDelete),
];

// on cleanup
gantt.clearAll();
for (const id of handlerIds) gantt.detachEvent(id);
```

Do not use `gantt.detachAllEvents()` for singleton cleanup. It is a legacy method that also removes Gantt's own internal listeners, leaving the chart non-functional.

The DataProcessor attached to the singleton, however, is safe to destroy and recreate. Keep a reference to the DataProcessor returned by `gantt.createDataProcessor(...)` and call `dataProcessor.destructor()` on cleanup; create a fresh DataProcessor when Gantt is activated again:

```ts
// during init
const dataProcessor = gantt.createDataProcessor(routerFn);

// on cleanup
dataProcessor.destructor();

// on reactivation
const dataProcessor = gantt.createDataProcessor(routerFn);
```

Clearing `container.innerHTML` afterwards is rarely needed and only useful when the host element survives across re-mounts.

## Config And Templates

Common configuration areas:

- `gantt.config.columns` — grid columns and rendering
- `gantt.config.scales` — timeline scales and headers
- `gantt.config.scale_height`, `row_height`, `bar_height` — sizing
- `gantt.config.layout` — layout and view structure
- `gantt.config.lightbox` — task editing UI
- interaction settings (`readonly`, etc.)

Templates:

- `gantt.templates.*` — formatting of task bars, grid cells, scale labels, tooltips, and CSS classes

Example:

```ts
gantt.config.columns = [
  { name: "text", label: "Task name", tree: true, width: "*" },
  { name: "start_date", label: "Start", align: "center", width: 100 },
  { name: "duration", label: "Duration", align: "center", width: 80 },
  { name: "add", label: "", width: 44 },
];

gantt.config.scales = [
  { unit: "month", step: 1, format: "%F %Y" },
  { unit: "day", step: 1, format: "%d %M" },
];

// Task bar text
gantt.templates.task_text = (start, end, task) => {
  return task.text;
};

// Highlight timeline cells (e.g. weekends)
gantt.templates.timeline_cell_class = (task, date) => {
  const day = date.getDay();
  return day === 0 || day === 6 ? "weekend" : "";
};

// Scale styling
gantt.templates.scale_cell_class = (date) => {
  const day = date.getDay();
  return day === 0 || day === 6 ? "weekend" : "";
};

// Grid column template
gantt.config.columns[0].template = (task) => {
  return task.text.toUpperCase();
};
```

Do not guess template signatures. Verify unfamiliar templates with MCP.

**Security — templates return raw HTML.** Whatever a template returns is inserted into the DOM as HTML, not as text. Any user-supplied field (`task.text`, descriptions, resource names, comments, free-form custom fields) that reaches a template unsanitized is an XSS vector — e.g. a task named `<img src=x onerror=alert(1)>` will execute on render.

Mitigate at both ends:

- Sanitize / escape user-supplied text on save in the backend (defense in depth).
- Escape — or sanitize via a vetted library like DOMPurify — before the data reaches Gantt (`parse`, `addTask`, `updateTask`, `remoteUpdates`, datastore writes), or escape inside the template before returning:

```ts
const escapeHtml = (s) => String(s ?? "").replace(/[&<>"']/g, (c) => ({
  "&": "&amp;", "<": "&lt;", ">": "&gt;", "\"": "&quot;", "'": "&#39;",
}[c]));

gantt.templates.task_text = (_s, _e, task) => escapeHtml(task.text);
gantt.templates.tooltip_text = (_s, _e, task) => `<b>${escapeHtml(task.text)}</b>`;
```

If the template intentionally returns markup (icons, badges, a `<span class="...">` wrapper), escape only the user-supplied substrings, not the static markup.

`gantt.templates.parse_date` and `gantt.templates.format_date` can be overridden to accept or emit non-default date formats (useful when the backend returns timestamps or compact strings):

```ts
gantt.templates.parse_date = (str) => new Date(str);
gantt.templates.format_date = (date) => date.toISOString();
```

## Events

Wire app logic through documented events rather than direct mutation. Attach with `gantt.attachEvent` and detach (or destroy via `gantt.destructor`) on unmount:

```ts
const handlerId = gantt.attachEvent("onAfterTaskUpdate", (id, task) => {
  // app-side reactions, persistence triggers, store updates
});

// later
gantt.detachEvent(handlerId);
```

For one-shot subscriptions (e.g. running first-init startup code through `onGanttReady`), pass `{ once: true }` so Gantt auto-detaches the handler after the first invocation:

```ts
gantt.attachEvent("onGanttReady", () => {
  // runs once, even if gantt.init is called again later
}, { once: true });
```

Event names and argument lists are stable but numerous. Verify unfamiliar events with MCP before wiring them.

## Localization

Switch the active locale at runtime with `gantt.i18n.setLocale("xx")` (32 locales ship with every edition). Per-label overrides go through `gantt.locale.labels.*`:

```ts
gantt.i18n.setLocale("ru");
gantt.locale.labels.section_description = "Description";
gantt.locale.labels.column_text = "Task name";
```

Apply locale changes before `gantt.init` when possible. If changed after init, call `gantt.render()`.

## Read-Only Mode

To make the chart non-editable:

```ts
gantt.config.readonly = true;
```

This disables built-in editing (drag, resize, inline edit, lightbox).

Note that API methods still work. If needed, block mutations in your own event handlers or UI.

Also remove or hide app-level controls that mutate data (e.g. add buttons, undo/redo).
