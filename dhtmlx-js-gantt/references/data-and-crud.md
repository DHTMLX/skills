# Data And CRUD

Use this file when tasks, links, state ownership, or persistence are involved.

## Data Flow

The canonical JavaScript Gantt data flow:

1. Load data with `gantt.parse({ tasks, links, resources?, assignments?, baselines? })` or `gantt.load(url)`.
2. Wire a DataProcessor via `gantt.createDataProcessor(...)` to send changes to the backend.
3. User interactions (drag, resize, inline edit, lightbox) and programmatic CRUD methods all notify the DataProcessor automatically. The standard mutators are `gantt.addTask`, `gantt.updateTask`, `gantt.deleteTask`, `gantt.addLink`, `gantt.updateLink`, `gantt.deleteLink`, plus the datastore equivalents (see Datastores below).

If a feature does not need server persistence, skip step 2.

## Applying External Changes

When changes arrive from outside Gantt (app store update, socket push, polling refresh, AI agent action), pick the channel that matches the change shape.

**Hard reload — for first load, page resets, or when most of the dataset turned over at once:**

```ts
gantt.clearAll();
gantt.parse({ tasks, links });
```

Hard reload discards scroll position, selection, branch expansion state, and the undo stack. Use it when those losses are acceptable.

**Soft update — for incremental external changes that must not bounce back to the backend:**

```ts
gantt.batchUpdate(() => {
  gantt.silent(() => {
    for (const task of added)   gantt.addTask(task, task.parent);
    for (const task of updated) gantt.updateTask(task.id, task);
    for (const id   of removed) if (gantt.isTaskExists(id)) gantt.deleteTask(id);

    for (const link of addedLinks)   gantt.addLink(link);
    for (const link of updatedLinks) gantt.updateLink(link.id, link);
    for (const id   of removedLinks) if (gantt.isLinkExists(id)) gantt.deleteLink(id);
  });
});
```

- `gantt.silent(fn)` runs the wrapped mutations without notifying the DataProcessor. Use it when the change already came from the source of truth and must not be echoed back as a write request.
- `gantt.batchUpdate(fn)` defers rendering until the batch finishes; combine with `silent` to keep the chart fluid during bulk syncs.
- Always pre-check with `gantt.isTaskExists(id)` / `gantt.isLinkExists(id)` before update or delete to keep the routine idempotent.

For streamed task/link changes (multi-user, server push, real-time DB subscription), `gantt.ext.liveUpdates.remoteUpdates` is the dedicated built-in equivalent of `gantt.silent(() => gantt.addTask(...))`:

```ts
const { remoteUpdates } = gantt.ext.liveUpdates;
remoteUpdates.tasks({ type: "update-task", task });
remoteUpdates.links({ type: "delete-link", link: { id } });
```

`remoteUpdates` is idempotent against the DataProcessor and ships with every edition (MIT Community, GPL Standard, and PRO). See [live-updates.md](live-updates.md) for both integration modes (custom change source and native multi-user backend) and the echo-loop guard. The manual `silent`+`batchUpdate`+API pattern above remains the right tool for non-task/link entities (resources, assignments, baselines via datastores) and for bulk syncs where you want explicit batch control.

## Datastores

Tasks and links are exposed through datastores reachable as `gantt.getDatastore("task")` and `gantt.getDatastore("link")`. Resources, assignments, and baselines (PRO only) live in their own datastores accessed the same way: `gantt.getDatastore("resource")`, `gantt.getDatastore("resourceAssignments")`, and `gantt.getDatastore("baselines")`. The resource and assignment store names are configurable via `gantt.config.resource_store` and `gantt.config.resource_assignment_store`; the baseline store name comes from `gantt.config.baselines.datastore`. Internal shortcuts `gantt.$data.resourcesStore`, `gantt.$data.assignmentsStore`, and `gantt.$data.baselineStore` resolve to the same instances and are commonly seen in wrapper code, but prefer `getDatastore(name)` in new code.

Each datastore exposes `.parse([...])`, `.clearAll()`, `.getItems()`, `.count()`, `.exists(id)`, `.getItem(id)`, `.addItem(item)`, `.updateItem(id, item)`, and `.removeItem(id)`. For wholesale updates to resources or assignments:

```ts
gantt.$data.resourcesStore.clearAll();
gantt.$data.resourcesStore.parse(resources);

gantt.$data.assignmentsStore.clearAll();
gantt.$data.assignmentsStore.parse(assignments);
```

For incremental external updates wrap datastore mutations in `gantt.silent(...)` the same way as task/link updates, so the DataProcessor is not triggered for changes that already came from the backend.

## Data Model Essentials

Use the `tasks` and `links` shape expected by Gantt.

Example:

```ts
const projectData = {
  tasks: [
    {
      id: "1",
      text: "Website Relaunch",
      type: "project",
      start_date: "2026-05-04T00:00:00",
      duration: 20,
      progress: 0.35,
      open: true,
      parent: 0,
    },
    {
      id: "2",
      text: "Discovery",
      start_date: "2026-05-04T00:00:00",
      duration: 4,
      progress: 0.8,
      parent: "1",
    },
  ],
  links: [{ id: "1", source: "2", target: "3", type: "0" }],
};

gantt.parse(projectData);
```

Common task fields:
- `id`
- `text`
- `start_date`
- `duration`
- `end_date`
- `progress`
- `parent`
- `type`
- `open`

Common link fields:
- `id`
- `source`
- `target`
- `type`

`link.type` values are strings `"0"` (finish-to-start), `"1"` (start-to-start), `"2"` (finish-to-finish), `"3"` (start-to-finish). The symbolic constants `gantt.config.links.finish_to_start`, `start_to_start`, `finish_to_finish`, `start_to_finish` resolve to the same strings and are accepted everywhere `link.type` is read.

When loading from a backend, map rows explicitly into Gantt task/link objects. Do not pass raw database rows straight through.

For server-driven setups, `gantt.load(url, type?)` fetches and parses in one call (XML or JSON depending on `type`); use it as an alternative to `gantt.parse(...)` when the backend already returns Gantt-shaped JSON.

## Dates

Use ISO date strings or `Date` objects consistently.

## Duration Unit

`gantt.config.duration_unit` (`"minute" | "hour" | "day" | "week" | "month" | "year"`) decides the unit of stored task durations. Set it at project start — changing it later usually requires converting persisted durations. Decimal duration values (e.g. a 4.5-hour task stored as `duration: 4.5`) are a PRO-only feature; on a free edition (MIT Community or GPL Standard), store durations as integers in the smallest unit the product needs.

## DataProcessor

`gantt.createDataProcessor()` creates a DataProcessor instance and attaches it to Gantt.

It accepts:
- a predefined request config object
- a router function
- a router config object

Router function:

```ts
gantt.createDataProcessor((entity, action, data, id) => {
  // entity: "task" | "link" | "resource" | "assignment"
  // action: "create" | "update" | "delete"
  // data: processed object
  // id: processed object id
});
```

Router object:

```ts
gantt.createDataProcessor({
  task: {
    create: (data) => Promise.resolve({}),
    update: (data, id) => Promise.resolve({}),
    delete: (id) => Promise.resolve({}),
  },
  link: {
    create: (data) => Promise.resolve({}),
    update: (data, id) => Promise.resolve({}),
    delete: (id) => Promise.resolve({}),
  },
});
```

All router handlers should return either a Promise or a data response object.

If the backend assigns a new id during create, return `id` or `tid` so Gantt can apply the database id:

```ts
gantt.createDataProcessor(async (entity, action, data, id) => {
  if (entity === "task" && action === "create") {
    const created = await api.createTask(data);
    return { tid: created.id };
  }

  return {};
});
```

If the installed version does not replace the id reliably from the DataProcessor response alone, call the documented id-change method for the entity as part of the create flow and still return `{ id }` or `{ tid }`.

```ts
if (entity === "task" && action === "create") {
  const created = await store.createTask(payload);
  if (created.id !== id) gantt.changeTaskId(id, created.id);
  return { tid: created.id };
}
```

## Persistence Guardrails

- Build backend payloads explicitly from normalized task/link models.
- Do not persist raw Gantt callback objects.
- Normalize date values before sending them to the backend.
- Keep `duration` aligned with the configured `duration_unit`.
- If temporary ids are used, make id replacement explicit and deterministic.
- Persist links only when both endpoints refer to persisted task ids.
- Sanitize or HTML-escape user-supplied text fields (task `text`, descriptions, resource names, custom string fields) at the persistence boundary. Gantt templates inject return values as raw HTML, so any unsanitized text becomes an XSS vector when rendered. Sanitize defensively at both write time (backend) and load time (before `gantt.parse` / `gantt.load` / `remoteUpdates`).
