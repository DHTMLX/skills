# Live Updates

Use this file when Gantt must reflect changes coming from outside the page — multi-user collaboration, server push, real-time database subscriptions, or any non-local source of mutations.

## Overview

`gantt.ext.liveUpdates` is a built-in extension exposing two helpers:

- `RemoteEvents` — opens a WebSocket connection against the documented DHTMLX multi-user backend protocol, handles handshake, subscribe/unsubscribe, and routes incoming messages.
- `remoteUpdates` — applies incoming task/link change messages to Gantt **without notifying the DataProcessor**. Use it as the canonical channel for any external change stream, including ones that have nothing to do with the native protocol.

Both helpers ship with **every edition** — MIT Community (v10+), GPL Standard (v9.x), and Professional. Live updates is available in all of them (it is not an edition-gated feature in [editions.md](editions.md)).

```ts
const { RemoteEvents, remoteUpdates } = gantt.ext.liveUpdates;
```

## Two Integration Modes

1. **Custom change source** — your own subscription (Firestore, app socket, polling) calls `remoteUpdates.tasks(...)` / `remoteUpdates.links(...)` directly. The dedicated equivalent of `gantt.silent(() => gantt.addTask(...))` for streamed changes.
2. **Native multi-user backend** — `new RemoteEvents(url, token).on(remoteUpdates)` connects to a backend that speaks the documented WS protocol; updates flow in automatically.

Both modes coexist: local writes go out through `gantt.createDataProcessor(...)`, external writes come in through `remoteUpdates`.

## Custom Change Source

Common pattern: subscribe to any change source, translate each change into a `remoteUpdates` call.

```ts
import { gantt } from "dhtmlx-gantt";
import "dhtmlx-gantt/codebase/dhtmlxgantt.css";
import { onSnapshot, addDoc, updateDoc, deleteDoc, doc, query, getDocs } from "firebase/firestore";
import { tasksCollection, linksCollection, db } from "./firebase";

const { remoteUpdates } = gantt.ext.liveUpdates;

gantt.init("gantt_here");

// initial load
const tasksSnap = await getDocs(query(tasksCollection));
const linksSnap = await getDocs(query(linksCollection));
gantt.parse({
  tasks: tasksSnap.docs.map(processTask),
  links: linksSnap.docs.map(processLink),
});

// subscribe — skip the first snapshot (already loaded) and self-originated writes
let primed = false;
const unsubscribe = onSnapshot(query(tasksCollection), (snap) => {
  if (!primed) { primed = true; return; }
  snap.docChanges().forEach((change) => {
    if (change.doc.metadata.hasPendingWrites) return;  // echo guard
    const task = processTask(change.doc);
    if (change.type === "added")    remoteUpdates.tasks({ type: "add-task",    task });
    if (change.type === "modified") remoteUpdates.tasks({ type: "update-task", task });
    if (change.type === "removed")  remoteUpdates.tasks({ type: "delete-task", task });
  });
});

// cleanup
gantt.attachEvent("onDestroy", () => unsubscribe());

// local writes flow out through the DataProcessor as usual
gantt.createDataProcessor(async (entity, action, data, id) => {
  const ref = doc(db, entity === "task" ? "tasks" : "links", id.toString());
  if (action === "create") { const added = await addDoc(entity === "task" ? tasksCollection : linksCollection, data); return { tid: added.id }; }
  if (action === "update") { await updateDoc(ref, data); return {}; }
  if (action === "delete") { await deleteDoc(ref); return {}; }
});
```

Key points:
- `remoteUpdates.tasks(...)` / `.links(...)` apply the change without triggering the DataProcessor — the change came from the source of truth and must not be echoed back.
- Skip self-originated writes (`hasPendingWrites` here, or the equivalent flag from your transport). Without that guard, every local mutation bounces back through the subscription and stacks.
- Skip the first snapshot if the source replays existing state — otherwise `add-task` will be called for items already loaded by `gantt.parse(...)`.
- Use `gantt.attachEvent("onDestroy", unsubscribe)` for tear-down. `onDestroy` fires before the instance is destroyed and is the right place to release subscriptions, timers, and pending requests.

## Native Multi-User Backend

When the backend implements the documented WS protocol (see `gantt-docs/docs/guides/multiuser-live-updates.md` and the reference repo at <https://github.com/DHTMLX/gantt-multiuser-backend-demo>), wire both halves of the loop with the built-ins:

```ts
const AUTH_TOKEN = "<per-user token>";

gantt.init("gantt_here");
gantt.load("/data");                       // initial bulk load

gantt.createDataProcessor({
  url: "/data",
  mode: "REST-JSON",
  headers: { "Remote-Token": AUTH_TOKEN },  // sent on every write
});

const { RemoteEvents, remoteUpdates } = gantt.ext.liveUpdates;
const remoteEvents = new RemoteEvents("/api/v1", AUTH_TOKEN);
remoteEvents.on(remoteUpdates);             // apply incoming events to Gantt
```

Protocol shape (illustrative — see the guide for the full spec):

- **Handshake** — `RemoteEvents` issues `GET /api/v1` with `Remote-Token`; backend replies `{api:{}, data:{}, websocket:true}` and accepts a WS connection. First WS frame from the server:
  ```json
  {"action":"start","body":"<connectionId>"}
  ```
- **Subscribe** — client tells the server which entities it cares about:
  ```json
  {"action":"subscribe","name":"tasks"}
  {"action":"subscribe","name":"links"}
  ```
- **Event** — server broadcasts a change after handling a REST write:
  ```json
  {"action":"event","body":{"name":"tasks","value":{"type":"update-task","task":{...}}}}
  ```

`remoteUpdates` consumes events whose `body.name` is `tasks` or `links` and whose `value.type` is one of the six documented actions (see the API summary below). Anything else is passed to the custom handlers.

## Custom Message Handlers And Custom Entities

`RemoteEvents.on` accepts either the default `remoteUpdates` helper, an object of per-entity handlers, or both — call `.on` more than once to layer them.

Override the built-in handler for `tasks`:

```ts
remoteEvents.on({
  tasks: (message) => {
    const { type, task } = message;
    if (type === "highlight-task") highlightTask(task.id);
    // fall back for built-in types
    if (type.startsWith("add-") || type.startsWith("update-") || type.startsWith("delete-")) {
      remoteUpdates.tasks(message);
    }
  },
});
```

Subscribe to a custom entity (sends `{"action":"subscribe","name":"presence"}` to the server):

```ts
remoteEvents.on({
  presence: (message) => {
    const { type, value } = message;        // any shape you choose on the server
    if (type === "user-joined") showPresence(value);
  },
});
```

The server-side event shape for a custom entity follows the same envelope: `{"action":"event","body":{"name":"presence","value":{"type":"user-joined", ...}}}`.

## `remoteUpdates` API Summary

Six documented action shapes. Each call is idempotent against the DataProcessor — Gantt applies the change without firing a write hook.

```ts
remoteUpdates.tasks({ type: "add-task",    task });               // task: full task object
remoteUpdates.tasks({ type: "update-task", task });
remoteUpdates.tasks({ type: "delete-task", task: { id } });       // only id is required

remoteUpdates.links({ type: "add-link",    link });
remoteUpdates.links({ type: "update-link", link });
remoteUpdates.links({ type: "delete-link", link: { id } });
```

For resources, assignments, baselines, or other datastore entities, `remoteUpdates` is not the channel — apply changes through `gantt.silent(() => gantt.$data.<store>.{addItem|updateItem|removeItem}(...))` instead (see [data-and-crud.md](data-and-crud.md)).

## Echo Loop Guard

When the same client both writes (DataProcessor → backend) and subscribes (backend → `remoteUpdates`), guard against re-applying your own writes. Two strategies:

- **Connection identity** — the native protocol gives the client a `connectionId` in the first WS frame. The backend can tag broadcasts with the origin and clients drop events that match their own `connectionId`. The reference backend demonstrates this pattern.
- **Source flag** — for custom transports (Firestore, app sockets), use whatever the transport exposes: `change.doc.metadata.hasPendingWrites` for Firestore, an `originatedBy` field on the payload, a per-write monotonic tag, etc.

Without a guard, every DataProcessor write produces a bounce that re-enters Gantt as a `remoteUpdates.tasks(...)` call, which is harmless on its own but stacks under bursty edits and confuses optimistic UI on the writer's side.
