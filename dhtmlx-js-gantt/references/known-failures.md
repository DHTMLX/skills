# Known Failures

Use this file when debugging incorrect or flaky DHTMLX JavaScript Gantt integrations.

## 1. Missing Height

Problem:
- Gantt is mounted into a container without real height
- grid or timeline appears blank, collapsed, or partially rendered

Fix:
- give the Gantt container explicit height
- verify parent containers also provide height

## 2. Missing CSS Import

Problem:
- Gantt logic runs, but UI is broken or unstyled

Fix:
- import the CSS that matches the installed package

## 3. Duplicated Startup Code

Problem:
- the full startup sequence — applying configs, attaching event handlers, creating a DataProcessor — runs more than once against the same Gantt instance without cleanup in between
- event listeners and DataProcessors stack up: every change fires N times, persistence requests are sent multiple times, memory grows, and undo/redo behaves erratically

Not the cause:
- calling `gantt.init(...)` more than once is **not** itself a problem
- `gantt.init` is the documented way to rebuild the chart after changing `gantt.config.layout`, since the layout is constructed at init time and is not picked up by `gantt.render()` alone
- repeated `gantt.init` calls on the same instance are safe as long as they are not paired with re-running event/DataProcessor setup

Fix:
- separate one-time startup code (event handlers, DataProcessor creation, lookup lists) from re-runnable code (config tweaks + `gantt.init`)
- before re-running startup code, clear the instance:
  - factory instance: `instance.destructor()`
  - singleton: `gantt.clearAll()`, detach handlers via stored `handlerId`s (never `detachAllEvents()`), and `dataProcessor.destructor()` on the DataProcessor reference
- see the Cleanup section in [setup.md](setup.md) for the full pattern

`onGanttReady` trap:
- `gantt.init` fires `onGanttReady` every time it runs, not only on the first call
- any startup code hooked through `onGanttReady` (event attachment, DataProcessor creation, lookup list registration) will re-run on every re-init and stack up exactly as if it had been called directly
- fix by making the handler self-detach after the first run, or by registering it with the `{once: true}` option:

```ts
gantt.attachEvent("onGanttReady", () => {
  attachAppEventHandlers();
  initDataProcessor();
}, { once: true });
```

## 4. Wrong Data Shape

Problem:
- Gantt receives raw backend rows or mismatched field names
- hierarchy, dates, or links do not render correctly

Fix:
- map backend rows explicitly to Gantt task/link fields
- use `id`, `text`, `start_date`, `duration`, `parent`, `source`, `target`, and `type` consistently

## 5. Date Serialization Bugs

Problem:
- persistence assumes every date is a `Date`
- date values are sent in inconsistent formats

Fix:
- normalize dates before persistence
- handle both `Date` and string inputs

## 6. Temporary ID Bugs

Problem:
- backend assigns a real id, but Gantt keeps a temporary id
- links are created or persisted using temporary task ids

Fix:
- return `{ id }` or `{ tid }` after create so Gantt replaces the temporary id
- persist links only after both endpoints have real ids
- use the documented id-change method if automatic replacement does not occur

## 7. Stale State In Persistence

Problem:
- persistence handlers use outdated task or link data
- updates overwrite newer changes

Fix:
- always read the latest data when building persistence payloads
- avoid using stale snapshots captured at initialization

## 8. Sortorder Drift

Problem:
- task updates overwrite row order
- reload restores a different order

Fix:
- treat reorder as a dedicated flow
- persist `sortorder`
- load tasks ordered by `sortorder`

## 9. Private Field Persistence

Problem:
- backend payloads or exports include Gantt runtime fields (`$...`, `_...`)

Fix:
- build payloads from normalized data models
- exclude private and runtime fields explicitly

## 10. Template Signature Guessing

Problem:
- a template callback uses guessed parameter order or return type

Fix:
- verify template signatures with DHTMLX MCP or official docs before implementing

## 11. Resource Field Drift

Problem:
- different fields or structures are used for resource binding across UI, Gantt config, and persistence

Fix:
- use one consistent resource field or assignment structure across the feature

## 12. Calendar Drift

Problem:
- Gantt and the application use different working time rules
- durations and non-working time are calculated inconsistently

Fix:
- keep working time rules in one source of truth
- apply them to Gantt configuration
- call `gantt.render()` after changes

MIT Community edition caveat:
- the **MIT Community edition (v10+)** has no working-time calendars: `gantt.config.work_time` and `gantt.setWorkTime(...)` are **hard no-ops** (every hour is working time, durations never skip weekends). The calls do not throw, so the bug is silent — bars just span weekends and durations don't match expectations.
- if non-working time must be respected on a MIT install, do the date math in app code and use templates (`timeline_cell_class` / `scale_cell_class`) for styling. Working calendars are available in the legacy GPL Standard build (v9.x) and in PRO — see [editions.md](editions.md).

## 13. Destructor Called On The Singleton

Problem:
- cleanup code calls `gantt.destructor()` on the singleton imported via `import { gantt } from "..."`
- after the call, the module-level `gantt` is the only instance the page had, and it is now non-functional
- subsequent `gantt.init` / `gantt.parse` calls do nothing or throw, and the chart cannot be recovered without a full page reload

Fix:
- only call `destructor()` on factory instances created with `Gantt.getGanttInstance()`
- for the singleton, clean up by calling `gantt.clearAll()` and detaching the specific handlers attached during init using the ids returned by `gantt.attachEvent`
- never use `gantt.detachAllEvents()` for cleanup — it removes Gantt's own internal listeners and leaves the chart non-functional
- see the Cleanup section in [setup.md](setup.md) for the full pattern

## 14. Echo Loop From External Updates

Problem:
- a subscription to an external change source (multi-user backend, Firestore, app socket) applies incoming changes through plain `gantt.addTask` / `gantt.updateTask` / `gantt.deleteTask` (or the link equivalents)
- those calls fire the DataProcessor, which sends a write back to the backend
- the backend re-broadcasts the write, the subscription receives it, and Gantt applies it again
- under bursty edits, the loop stacks: the same task is re-saved and re-applied multiple times, updates can overwrite each other, and optimistic UI on the writer's side flickers

Fix:
- for streamed task/link changes use `gantt.ext.liveUpdates.remoteUpdates.tasks(...)` / `.links(...)` — it applies the change without notifying the DataProcessor
- for non-task/link entities (resources, assignments, baselines via datastores), wrap mutations in `gantt.silent(...)` so the DataProcessor is not triggered
- always add an echo guard at the subscription boundary so self-originated writes are dropped before they are re-applied:
  - native multi-user protocol — drop events whose origin matches your own `connectionId` (sent on the first WS frame)
  - Firestore — skip changes where `change.doc.metadata.hasPendingWrites` is true
  - custom transports — tag outgoing writes with a per-client id and ignore matching tags on the way back
- see [live-updates.md](live-updates.md) for full integration patterns

## 15. Date Parse Year Drift

Problem:
- loaded tasks render with dates in 1920, 1900, or similarly far-off years
- the input data clearly contained current dates, but the chart shows them shifted by decades

Cause:
- Gantt parsed incoming date strings with a format that does not match the actual string format
- the default parser interprets unrecognized tokens as zero, producing wildly wrong timestamps (e.g. `"05/13/2026"` parsed as `"DD-MM-YY"` becomes year 13 → adjusted to 1913)

Fix:
- pass `Date` objects to Gantt instead of strings whenever possible
- if strings are required, ensure they match the format Gantt expects (ISO `YYYY-MM-DD` or `YYYY-MM-DD HH:mm:ss` are the safest defaults)
- otherwise, override `gantt.templates.parse_date` and `gantt.templates.format_date` so they convert between the project's date format and `Date` objects correctly
- do not change the format on one side only; `parse_date` and `format_date` must round-trip the same representation

## 16. XSS Through Template Output

Problem:
- a task or other entity carries unsanitized user-supplied text (e.g. `task.text = "<img src=x onerror=alert(1)>"`)
- Gantt renders the task bar / tooltip / grid cell, calls the corresponding template (`task_text`, `tooltip_text`, column `template`), and inserts the return value as HTML
- the injected markup executes in the user's browser

Cause:
- Gantt templates inject return values as raw HTML, not text. The default templates simply return `task.text` (or other fields) untouched. Any HTML or script in the data renders as live DOM.
- the same applies to columns with a custom `template`, scale formatters that include data fields, tooltip text, and any other template that interpolates data

Fix:
- treat every Gantt template output as an HTML sink
- HTML-escape (or sanitize via a vetted library such as DOMPurify) every user-supplied substring before it reaches the template — either at the data boundary (before `gantt.parse` / `gantt.load` / `remoteUpdates` / datastore writes) or inside the template return value
- sanitize on the server as well, so persisted data is safe regardless of which client loads it
- if a template intentionally returns markup, escape only the data substrings, not the static markup
- see the Templates section in [setup.md](setup.md) for an escape helper example
