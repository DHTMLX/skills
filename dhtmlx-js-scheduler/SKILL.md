---
name: dhtmlx-js-scheduler
description: >
  Builds and integrates core DHTMLX JavaScript Scheduler into JavaScript or
  TypeScript applications. Covers setup, initialization, lifecycle,
  scheduler.parse/scheduler.load, templates, events, plugins, DataProcessor,
  event CRUD, recurring events, timeline views, units views, locale/i18n,
  styling, and theming for dhtmlx-scheduler (Standard/GPL),
  @dhx/trial-scheduler, and @dhx/scheduler. Applies when working with booking
  calendars, event scheduling, event CRUD, resource scheduling, recurring
  events, timeline views, or units views, regardless of whether "DHTMLX" is
  mentioned by name. Provides verified API guidance rather than guessing core
  Scheduler APIs and surfaces edition limits before scaffolding.
---

## Source Of Truth

Use only:
1. The current project's files, structure, and established patterns
2. DHTMLX MCP for core Scheduler API details: https://docs.dhtmlx.com/mcp
3. Official DHTMLX Scheduler docs as fallback: https://docs.dhtmlx.com/scheduler/

Never invent config names, template callbacks, event names, method signatures, data shapes, plugin names, or backend behavior.

If any core Scheduler API detail is unclear, resolve it through DHTMLX MCP before writing code.

## Preflight

Before writing code, identify:
- **delivery**: npm package (`dhtmlx-scheduler`, `@dhx/trial-scheduler`, `@dhx/scheduler`) — check `package.json`, imports, lockfiles. Or script tag / CDN — check `<script>` tags in HTML, vendored `dhtmlxscheduler.js`, the `@license` banner inside that file.
- **edition**: `dhtmlx-scheduler` means Standard/GPL (PRO-only features unavailable); everything else is Professional. See [references/editions.md](references/editions.md) for the full detection procedure and the PRO-only feature list.
- **runtime**: plain JavaScript, TypeScript, Vite, browser-only `<script>`, or another app setup
- **outbound changes**: no persistence, `scheduler.createDataProcessor`, or custom backend/API clients
- **inbound changes**: none, hard reload (`scheduler.clearAll` + `scheduler.parse`), dynamic loading, or incremental external updates
- **inbound changes**: none, hard reload (`scheduler.clearAll` + `scheduler.parse`), dynamic loading via `scheduler.setLoadMode`, or incremental updates via scheduler API calls — see [references/data-and-crud.md](references/data-and-crud.md). For real-time streams use `scheduler.ext.liveUpdates` — see [references/live-updates.md](references/live-updates.md).

## Workflow

1. Confirm the installed DHTMLX Scheduler package and import path.
2. Decide the outbound and inbound change strategies before implementing CRUD or external sync.
3. Read only the reference file needed for the task:
   - Editions, package detection, and PRO-only feature list: [references/editions.md](references/editions.md)
   - Setup: [references/setup.md](references/setup.md)
   - CRUD, state, and persistence: [references/data-and-crud.md](references/data-and-crud.md)
   - Failure cases and guardrails: [references/known-failures.md](references/known-failures.md)
   - Advanced patterns (plugins, views, resources, readonly, validation, undo/redo, schema): [references/advanced-patterns.md](references/advanced-patterns.md)
   - Real-time / multi-user / external change streams (`scheduler.ext.liveUpdates`): [references/live-updates.md](references/live-updates.md)
   - Styling, skins, CSS variables, selectors, and template-based visual customization: [references/styling-and-theming.md](references/styling-and-theming.md)
4. Use DHTMLX MCP before relying on advanced or unfamiliar APIs.
5. Implement with documented APIs only.

## MCP Server

This skill relies on DHTMLX MCP for API verification. If the `dhtmlx-mcp` tool is not available, ask the user to add it:

Claude Code:
```bash
claude mcp add --transport http dhtmlx-mcp https://docs.dhtmlx.com/mcp
```

Codex:
```bash
codex mcp add dhtmlx-mcp --url https://docs.dhtmlx.com/mcp
```

If MCP is not available, use the official docs at https://docs.dhtmlx.com/scheduler/ as fallback.

## Consult MCP First For

Consult DHTMLX MCP before using or changing:
- template callbacks you have not already verified
- lifecycle patterns around multiple instances or repeated mount/unmount
- DataProcessor modes, router signatures, delete confirmation, id replacement, and error handling
- recurring event payload shape (`rrule`, `duration`, `recurring_event_id`, `original_start`, `deleted`)
- lightbox section types and `map_to` conventions
- custom lightbox controls via `scheduler.form_blocks` (`render`/`set_value`/`get_value`/`focus`)
- marked timespans (`addMarkedTimespan`/`deleteMarkedTimespan`), `blockTime`, and the `limit` plugin
- per-view event filters (`scheduler.filter_<view>`) and view refresh via `setCurrentView`/`updateView`
- DataProcessor transaction mode, request headers, and `onBeforeUpdate` payload shaping
- `scheduler.plugins({...})` extension loading and plugin-specific APIs
- timeline/units view configuration (`property`, `y_property`, `x_unit`, `x_step`, `x_size`, section lists)
- dynamic loading via `scheduler.setLoadMode`
- theme/skin values and CSS variable names
- locale and i18n customization
- `scheduler.ext.liveUpdates` (`RemoteEvents`, `remoteUpdates`) and the multi-user backend WebSocket protocol

## Hard Rules

- The Scheduler container must have explicit height.
- Import the matching Scheduler CSS.
- Use the app theme as the single source of truth.
- Prefer documented config, templates, events, and methods over undocumented internals.
- Keep Scheduler init/parse/events/DataProcessor cleanup in one lifecycle boundary.
- Build the Scheduler instance once per mount; reflect data changes with `clearAll`/`parse` or incremental CRUD and refresh views with `setCurrentView`/`updateView` — do not recreate the instance to apply new data, filters, or props.
- Activate extensions with `scheduler.plugins({...})` before using extension APIs.
- Normalize `start_date` and `end_date` before persistence.
- Scheduler renders events in the browser's local timezone; if the backend stores another zone (e.g. UTC), convert at the persistence boundary on the way in and out.
- Build backend payloads explicitly from normalized event models.
- Return `{ id: databaseId }` or `{ tid: databaseId }` after creates when the backend assigns a real id.
- If not using DataProcessor, call `scheduler.changeEventId(tempId, databaseId)` after create persistence.
- Scheduler template return values are injected as raw HTML. Sanitize or HTML-escape every user-supplied string before it enters Scheduler or before returning it from templates.
- Enforce permissions on the backend/DataProcessor boundary. Scheduler readonly mode controls UI behavior and should not be treated as a security mechanism.
- When the installed package is Standard/GPL and the requested feature appears in the PRO-only list in [references/editions.md](references/editions.md), warn the user before scaffolding, then still scaffold the requested code.

## Quick Checklist

- [ ] Correct package identified
- [ ] Edition determined and PRO-only usage flagged for Standard installs
- [ ] Matching CSS import used
- [ ] Explicit height provided
- [ ] Singleton vs multi-instance model chosen
- [ ] Plugins enabled before dependent APIs
- [ ] Outbound and inbound change strategies decided
- [ ] Dates normalized before persistence
- [ ] DataProcessor return values handled
- [ ] Advanced APIs verified with MCP
