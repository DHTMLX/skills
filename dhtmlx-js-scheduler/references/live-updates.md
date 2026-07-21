# Live Updates

Use this file when Scheduler must reflect changes coming from outside the local UI — multi-user collaboration, server push, WebSocket subscriptions, realtime databases, polling, or another browser tab.

## Overview

`scheduler.ext.liveUpdates` is a built-in extension exposing two helpers:

- `RemoteEvents` — opens a WebSocket connection against the documented DHTMLX multi-user backend protocol, handles handshake, subscribe/unsubscribe messages, and routes incoming messages.
- `remoteUpdates` — parses incoming WebSocket messages and applies the appropriate changes to Scheduler.

```ts
const { RemoteEvents, remoteUpdates } = scheduler.ext.liveUpdates;
```

`remoteUpdates` handles the documented `add-event`, `update-event`, and `delete-event` message types and applies the corresponding changes to Scheduler data.

The current live updates API is separate from the legacy `DataProcessor.live_updates(...)` implementation. The legacy implementation is deprecated and is not included in the main package.

The standard integration pattern is:

```ts
const remoteEvents = new RemoteEvents("/api/v1", authToken);

remoteEvents.on(remoteUpdates);
```

## Custom Change Source

Common pattern: subscribe to a custom realtime source and translate incoming changes into the message format expected by `remoteUpdates`.

```ts
remoteUpdates.events({ type: "add-event", event });
remoteUpdates.events({ type: "update-event", event });
remoteUpdates.events({ type: "delete-event", event: { id } });
```

Skip self-originated writes and replayed initial snapshots according to the rules of the underlying transport.

## Native Multi-User Backend

When the backend implements the documented DHTMLX live-updates protocol, wire both halves of the synchronization loop with the built-ins:

```ts
const AUTH_TOKEN = "<per-user token>";

scheduler.init("scheduler_here", new Date(2026, 4, 1), "week");
scheduler.load("/events");

scheduler.createDataProcessor({
  url: "/events",
  mode: "REST-JSON",
  headers: {
    "Remote-Token": AUTH_TOKEN,
  },
});

const { RemoteEvents, remoteUpdates } = scheduler.ext.liveUpdates;
const remoteEvents = new RemoteEvents("/api/v1", AUTH_TOKEN);

remoteEvents.on(remoteUpdates);
```

Protocol shape (illustrative — see the guide for the full specification):

- **Handshake** — `RemoteEvents` issues `GET /api/v1` with `Remote-Token`; the backend replies:

  ```json
  {"api":{},"data":{},"websocket":true}
  ```

  and accepts a WebSocket connection. First WebSocket frame from the server:

  ```json
  {"action":"start","body":"<connectionId>"}
  ```

- **Subscribe** — the client subscribes to Scheduler event updates:

  ```json
  {"action":"subscribe","name":"events"}
  ```

  Unsubscribe:

  ```json
  {"action":"unsubscribe","name":"events"}
  ```

- **Event** — the server broadcasts Scheduler changes:

  ```json
  {
    "action":"event",
    "body":{
      "name":"events",
      "value":{
        "type":"update-event",
        "event":{}
      }
    }
  }
  ```

Supported event actions:

```text
add-event
update-event
delete-event
```

Local changes are sent to the backend through DataProcessor. The backend broadcasts confirmed changes over WebSocket; `RemoteEvents` receives those messages and `remoteUpdates` applies the corresponding changes to Scheduler.

## Custom Message Handlers And Custom Entities

`RemoteEvents.on(...)` accepts the default `remoteUpdates` helper or an object of per-entity handlers. The documented pattern is to attach `remoteUpdates` first, then add custom handlers when needed.

```ts
const { RemoteEvents, remoteUpdates } = scheduler.ext.liveUpdates;
const remoteEvents = new RemoteEvents("/api/v1", authToken);

remoteEvents.on(remoteUpdates);

remoteEvents.on({
  events: (message) => {
    if (message.type === "custom-action") {
      handleCustomAction(message.event);
    }
  },
});
```

Subscribe to a custom entity (sends `{"action":"subscribe","name":"calendars"}` to the server):

```ts
remoteEvents.on({
  calendars: (message) => {
    const { type, value } = message;

    if (type === "custom-action") {
      updateCalendars(value);
    }
  },
});
```

The server-side event shape for a custom entity follows the same envelope:

```json
{"action":"event","body":{"name":"calendars","value":{"type":"custom-action","value":{}}}}
```

## Echo Loop Guard

When the same client both writes (DataProcessor → backend) and subscribes to live updates (backend → `remoteUpdates`), guard against re-applying the same local write as a remote update.

Known strategies:

- **Connection identity** — the native protocol gives the client a `connectionId` in the first WebSocket frame:
  ```json
  {"action":"start","body":"<connectionId>"}
  ```
  If you control the backend, use this identity to avoid broadcasting a confirmed change back to the originating client, or include an origin marker and let the client ignore its own updates.

- **Source flag** — for custom transports, use whatever the transport exposes: Firestore `hasPendingWrites`, an `originatedBy` field on the payload, operation ids, pending local ids, or another per-write marker.
