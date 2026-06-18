# Data And CRUD

Use this file when events, state ownership, loading, recurring events, or persistence are involved.

## Data Flow

The canonical JavaScript Scheduler data flow:

1. Load events with `scheduler.parse(events)` or `scheduler.load(url)`.
2. Wire a DataProcessor via `scheduler.createDataProcessor(...)` to send changes to the backend.
3. User interactions (drag, resize, lightbox) and programmatic CRUD methods all notify the DataProcessor automatically. The standard mutators are `scheduler.addEvent`, `scheduler.updateEvent`, `scheduler.deleteEvent`, plus recurrence and custom datastore-related APIs where documented.

If a feature does not need server persistence, skip step 2.

## Applying External Changes

When changes arrive from outside Scheduler (app store update, socket push, polling refresh, AI agent action), pick the channel that matches the change shape.

**Hard reload — for first load, page resets, or when most of the dataset turned over at once:**

```ts
scheduler.clearAll();
scheduler.parse(events);
```

Hard reload replaces the loaded Scheduler data. Use it when losing transient Scheduler UI state is acceptable.

**Soft update — for incremental external changes:**

```ts
scheduler.batchUpdate(() => {
  for (const event of added) {
    scheduler.addEvent(event);
  }

  for (const event of updated) {
    if (scheduler.getEvent(event.id)) {
      scheduler.getEvent(event.id).text = event.text;
      scheduler.getEvent(event.id).start_date = event.start_date;
      scheduler.getEvent(event.id).end_date = event.end_date;

      scheduler.updateEvent(event.id);
    }
  }

  for (const id of removed) {
    if (scheduler.getEvent(id)) {
      scheduler.deleteEvent(id);
    }
  }
});

- `scheduler.batchUpdate(fn)` updates multiple events with a single re-rendering.

For streamed event changes (multi-user, server push, real-time DB subscription), `scheduler.ext.liveUpdates.remoteUpdates` is the dedicated built-in equivalent of applying incoming event changes silently:

```ts
const { RemoteEvents, remoteUpdates } = scheduler.ext.liveUpdates;
const remoteEvents = new RemoteEvents("/api/v1", authToken);

remoteEvents.on(remoteUpdates);
```

`remoteUpdates` handles documented `add-event`, `update-event`, and `delete-event` messages and applies incoming changes without sending them back through the CRUD backend. See [live-updates.md](live-updates.md) for the multi-user backend protocol, subscription flow, and echo-loop guard.

The manual synchronization pattern using the Scheduler API remains the right tool for custom entities or custom streams that are not handled by `remoteUpdates`.

## Data Model Essentials

Use the event shape expected by Scheduler.

Example:

```ts
const eventData = [
  {
    id: "1",
    text: "Website Relaunch",
    start_date: "2026-05-04 09:00",
    end_date: "2026-05-04 12:00",
  },
];

scheduler.parse(eventData);
```

Required event fields:
- `id`
- `text`
- `start_date`
- `end_date`

The default date format is defined by `scheduler.config.date_format` (default: `%Y-%m-%d %H:%i`). If backend strings use another format, set `scheduler.config.date_format` before parsing.

When loading from a backend, map rows explicitly into Scheduler event objects. Do not pass raw database rows straight through.

For server-driven setups, `scheduler.load(url)` fetches and parses data in one call. Scheduler automatically detects supported input formats (JSON, XML, and iCalendar), so use `scheduler.load(...)` as an alternative to `scheduler.parse(...)` when the backend already returns Scheduler-shaped data.

## Loading Data

Inline:

```ts
scheduler.init("scheduler_here", new Date(2026, 4, 1), "month");

scheduler.parse([
  {
    id: 1,
    text: "Meeting",
    start_date: "2026-05-11 14:00",
    end_date: "2026-05-11 17:00",
  },
]);
```

Backend:

```ts
scheduler.init("scheduler_here", new Date(2026, 4, 1), "month");

scheduler.load("/api/events");
```

If the GET request needs auth headers, fetch the data manually and call `scheduler.parse(payload)`.

## DataProcessor

`scheduler.createDataProcessor()` creates a DataProcessor instance and attaches it to Scheduler.

It accepts:
- a URL string
- a predefined request config object
- a router function
- a router config object

REST-JSON mode:

```ts
const dp = scheduler.createDataProcessor({
  url: "/api/events",
  mode: "REST-JSON",
});
```

Router function:

```ts
scheduler.createDataProcessor((entity, action, data, id) => {
  // entity: "event"
  // action: "create" | "update" | "delete"
  // data: processed event object
  // id: processed event id
});
```

Router object:

```ts
scheduler.createDataProcessor({
  event: {
    create: (data) => Promise.resolve({}),
    update: (data, id) => Promise.resolve({}),
    delete: (id) => Promise.resolve({}),
  },
});
```

All router handlers should return either a Promise or a data response object.

If the backend assigns a new id during create, return `id` or `tid` so Scheduler can apply the database id:

```ts
scheduler.createDataProcessor(async (entity, action, data, id) => {
  if (entity === "event" && action === "create") {
    const created = await api.createEvent(data);
    return { tid: created.id };
  }

  return {};
});
```

If you manage persistence without DataProcessor and the backend assigns a new id, call the documented id-change method:

```ts
const created = await api.createEvent(event);

if (created.id !== event.id) {
  scheduler.changeEventId(event.id, created.id);
}
```
### Auth Headers And Payload Shaping

The DataProcessor handles the **write** side (create/update/delete), so auth for writes is configured on the DataProcessor, not on `scheduler.load`. Attach headers with `setTransactionMode`, and shape or cancel the outgoing request with `onBeforeUpdate`:

```ts
const dp = scheduler.createDataProcessor({ url: "/api/events", mode: "REST-JSON" });

dp.setTransactionMode({ headers: { Authorization: `Bearer ${token}` } });

dp.attachEvent("onBeforeUpdate", (id, state, data) => {
  data.owner_id = currentUserId; // inject server-required fields not held on the event
  return true;                   // return false to cancel this request
});
```

Notes:
- A token captured at init can go stale; read it at request time (or recreate the DataProcessor) if it can rotate during the session.
- Server-side authorization is still mandatory — `onBeforeUpdate` is for payload shaping, not security.
- Verify the exact `setTransactionMode` option names and event signatures with MCP.

## Manual Persistence Without DataProcessor

If not using DataProcessor, listen to Scheduler events and persist changes manually:

```ts
scheduler.attachEvent("onEventAdded", (id, event) => {
  api.createEvent(event).then((result) => {
    scheduler.changeEventId(id, result.id);
  });
});

scheduler.attachEvent("onEventChanged", (id, event) => {
  api.updateEvent(id, event);
});

scheduler.attachEvent("onEventDeleted", (id) => {
  api.deleteEvent(id);
});
```

Prefer DataProcessor unless the app architecture explicitly needs manual control.

## Dates

Use `Date` objects or date strings that match `scheduler.config.date_format` consistently.

Scheduler places events by the **browser's local timezone**. If the backend stores another zone (e.g. UTC), convert to local before `parse`/`addEvent` and back before persisting — otherwise events render shifted by the offset and drift further on every save round-trip. Keep the conversion at the persistence boundary (load/save), not scattered across templates, and do it consistently for both reads and writes.

## Recurring Events

For the rrule-based recurring engine, enable recurring support:

```ts
scheduler.plugins({ recurring: true });
```

Default recurring lightbox section:

```ts
scheduler.config.lightbox.sections = [
  { name: "description", map_to: "text", type: "textarea", focus: true },
  { name: "recurring", type: "recurring", map_to: "rrule" },
  { name: "time", height: 72, type: "time", map_to: "auto" },
];
```

Recurring event fields:
- `rrule` - RFC-5545 recurrence rule on the series row
- `duration` - duration of each occurrence in seconds
- `recurring_event_id` - parent series id for modified/deleted occurrences
- `original_start` - original date/time of the occurrence being replaced
- `deleted` - true for deleted occurrence tombstones

Rules:
- `rrule` belongs on the series row, not exception rows
- modified/deleted occurrences are separate rows
- deleted occurrences should be updated to `deleted: true`, not physically removed
- if a series is modified/deleted, remove or recalculate its exception rows according to product behavior
- DHTMLX stores `start_date` and `end_date` separately; do not embed DTSTART/DTEND into the `rrule` string

Legacy recurring mode uses `rec_type`, `event_pid`, and `event_length` instead.

## Persistence Guardrails

- Build backend payloads explicitly from normalized event models.
- Do not persist raw Scheduler callback objects.
- Normalize dates before sending them to the backend.
- Keep app-level custom fields explicit in mapping (`resource_id`, `location_id`, `calendar_id`, etc.).
- If the backend assigns ids, make id replacement explicit and deterministic.
- Validate permissions and constraints on the backend; client-side validation is not security.
- Sanitize or HTML-escape user-supplied strings at write and read/template boundaries.
- If recurring events are enabled, implement recurring-specific server logic before production use.
