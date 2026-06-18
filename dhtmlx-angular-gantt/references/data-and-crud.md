# Data And CRUD

Use this file when tasks, links, state ownership, or persistence are involved.

## Data Model Essentials

In Angular wrapper integrations, keep a normalized app model and map it explicitly to Gantt task/link payloads.

Minimum task shape:
```ts
interface Task {
  id: string | number;
  text: string;
  start_date: Date;
  duration: number;
  progress: number;
  parent: string | number;
  type?: "task" | "project" | "milestone";
  open?: boolean;
  end_date?: Date;
}
```

Minimum link shape:
```ts
interface Link {
  id: string | number;
  source: string | number;
  target: string | number;
  type: "0" | "1" | "2" | "3";
}
```

When loading from backend, map rows into app task/link models explicitly. Do not pass raw DB rows straight through.

### Wrapper types: `TaskInput`, `Task`, `SerializedTask`

`@dhx/angular-gantt` exports three task types — use the right one for the context:

- **`TaskInput`** — data you *provide* to Gantt (the `[tasks]` input, `gantt.addTask()`, your own store/state). Date fields accept `Date` **or** `string`, and every field including `id` is optional. **Type your store/state as `TaskInput[]`** — it accepts either date form and avoids `Date`-vs-`string` type errors.
- **`Task`** — the runtime object Gantt returns (`gantt.getTask()`): `Date` dates plus computed `$`-prefixed fields.
- **`SerializedTask`** — the JSON/wire form: `string` dates only. Use it for server payloads, not for in-memory state seeded with `Date` objects.

Common mistake (breaks on v10): typing store state as `SerializedTask[]` but seeding it with `new Date(...)` values. Since v10 `SerializedTask` date fields are `string`-only, that no longer compiles — use `TaskInput[]` (or `Task[]`). Links have no date fields, so `Link[]` or `SerializedLink[]` both work.

## Choose Ownership Model First

Choose one model per page/feature area and keep it consistent.

Angular state/store as source of truth (recommended for most apps):
- store/component state owns `tasks` and `links`
- wrapper receives arrays through inputs
- edits are captured in `data.save` or `data.batchSave`
- callbacks update state/store and arrays flow back into `<dhx-gantt>`

Gantt as source of truth (specialized/performance-heavy pages):
- chart and backend own most runtime lifecycle
- Angular mirrors only when required
- requires a clear reconciliation strategy

Do not mix both models casually.

## CRUD Default

For regular CRUD, prefer `data.save`.

Pattern:
```ts
dataConfig: AngularGanttDataConfig = {
  save: async (entity, action, item, id) => {
    // call store action
    // store action normalizes payload
    // store action persists backend write
    // store action updates state on confirmed success
  },
};
```

Strict rules:
- route persistence through store/service actions, not inline component mutation spaghetti
- normalize callback payloads before backend writes
- for create with backend-assigned IDs, return remap response (`{ id }` or `{ tid }`) when using per-change save flow
- update state from latest snapshot, not stale captured arrays
- keep links persistence guarded by real persisted task IDs
- for `update`, persist only normalized values
- for `delete`, remove from state and persistence layer
- normalize date values before serialization
- if using temporary IDs, replace them deterministically after persistence
- persist a link only when both `source` and `target` already refer to real persisted task IDs
- if a task still has a temporary client-side ID, defer or reject link persistence until the real task ID replacement flow completes
- do not send links that reference temporary IDs to the backend
- build backend payloads explicitly from normalized task/link models, not from raw callback objects

## Date Normalization

`data.save` and `data.batchSave` callbacks can hand back either `Date` instances or formatted strings depending on how Gantt renders and parses dates. Pin both directions with `parse_date` and `format_date` templates so persistence always sees ISO strings, regardless of input shape.

```ts
import type { AngularGanttTemplates } from "@dhtmlx/trial-angular-gantt";

templates: AngularGanttTemplates = {
  parse_date: (value: string | Date) => this.parseDate(value),
  format_date: (date: string | Date) => this.formatDate(date),
};

private parseDate(value: string | Date): Date {
  return value instanceof Date ? value : new Date(value);
}

private formatDate(value: string | Date): string {
  return this.parseDate(value).toISOString();
}
```

Pass these templates to `<dhx-gantt>` once at the page/feature root. With both templates in place, dates round-trip as ISO strings through `data.save`/`data.batchSave` and through any state/store layer in between, even when Gantt internally serializes them.

## When To Use `batchSave`

Use `data.batchSave` when:
- heavy operations can emit many changes
- bulk backend sync is more appropriate than single-entity persistence
- you need grouped change handling rather than one callback per edit

Pattern:
```ts
dataConfig: AngularGanttDataConfig = {
  batchSave: (changes) => {
    // apply grouped task/link/resource changes
    // persist grouped payloads
    // update store in one coherent transition
  },
};
```

Do not switch to `batchSave` casually. Verify the exact expected behavior with MCP first.

Queue behavior (do not assume one DOM gesture = one persisted change):
- changes are batched within a short debounce window
- `create` + `update` for the same entity coalesce into a single `create` with the latest payload
- `create` + `delete` pairs are dropped entirely
- the wrapper strips internal `!nativeeditor_status` from payloads before invoking the callback

Plan persistence around the *post-coalescing* change set, not the user-visible action count.

## ID Remapping And Temporary IDs

Create flows often start with temporary client-side IDs.

Rules:
- backend must return persistent IDs
- in `save` mode, return `{ id: dbId }` or `{ tid: dbId }` so Gantt can remap
- do not persist links that reference temporary task IDs
- replace temporary IDs deterministically in state after create confirmation

## State Service Pattern (RxJS)

When Angular owns the data, expose narrow per-concern streams from a state service rather than one wide state object, then recompose them in the component with `combineLatest` and `AsyncPipe`. The wrapper compares input identities, so narrow streams keep `<dhx-gantt>` re-binding only the inputs that actually changed.

Service shape:
```ts
@Injectable()
export class GanttStateService {
  private readonly stateSubject = new BehaviorSubject<GanttState>(initial);

  readonly state$ = this.stateSubject.asObservable();
  readonly tasks$ = this.state$.pipe(map((s) => s.tasks));
  readonly links$ = this.state$.pipe(map((s) => s.links));
  readonly wrapperConfig$ = this.state$.pipe(map((s) => this.buildWrapperConfig(s.config)));
  readonly zoomLevel$ = this.state$.pipe(
    map((s) => s.config.zoom.current),
    distinctUntilChanged(),
  );
  readonly canUndo$ = this.state$.pipe(map((s) => s.past.length > 0), distinctUntilChanged());
  readonly canRedo$ = this.state$.pipe(map((s) => s.future.length > 0), distinctUntilChanged());
}
```

Component shape:
```ts
protected readonly vm$ = combineLatest({
  tasks: this.ganttState.tasks$,
  links: this.ganttState.links$,
  config: this.ganttState.wrapperConfig$,
  zoomLevel: this.ganttState.zoomLevel$,
  canUndo: this.ganttState.canUndo$,
  canRedo: this.ganttState.canRedo$,
});
```

Rules:
- one `BehaviorSubject` per state shape; do not spread state across many subjects that must stay synchronized
- derive view streams with `map` + `distinctUntilChanged` so identical references are deduplicated
- bind through `AsyncPipe` in the template; do not subscribe manually in the component
- route every mutation through service methods so history, persistence, and `BehaviorSubject` updates stay coherent

## Row Reorder And Sortorder

Treat row reorder as dedicated flow, not normal update.

Rules:
- detect reorder intent from update payload (`target` when available)
- rebuild full ordered task list in memory
- recompute `sortorder` for all affected tasks
- persist full reordered set
- if hierarchy changes during move, persist `parent_id` together with `sortorder`
- load tasks ordered by `sortorder`

## Persistence Guardrails

- Do not assume task updates always contain real `Date` instances.
- Normalize date-like inputs before serialization and persistence.
- Build backend payloads explicitly from normalized task/link models, not from raw callback objects.
- Keep frontend ordering (`sortorder`), backend payloads, and DB schema aligned.
- If IDs are temporary on create, make ID replacement flow explicit and deterministic.
- If the app introduces undo/redo, persistence rules must remain correct after history actions.

## Migration: v9 → v10

When upgrading an Angular Gantt app from 9.x to 10.x:

- **Package / import path may differ** between 9.x and 10.x (and between trial and licensed builds), e.g. `@dhtmlx/trial-angular-gantt` vs `@dhx/angular-gantt`. Verify the installed package in `package.json` and align all imports.
- **`SerializedTask` / `SerializedLink` are now strictly serialized.** Date fields are `string`-only (they were `Date | string` in 9.x) and `SerializedTask.id` is now optional. The most common upgrade break is store/seed data typed as `SerializedTask[]` but populated with `Date` objects — retype it as `TaskInput[]` (accepts `Date` or `string`) or `Task[]` (Date). See [Wrapper types](#wrapper-types-taskinput-task-serializedtask).
- **`NewTask` → `TaskInput`.** `NewTask` is no longer deprecated; it is now an alias of `TaskInput`. Prefer `TaskInput` in new code.
- Core-level changes also apply (auto-scheduling config, GPL → Community/MIT edition split, pure date-interval helpers). Confirm specifics via DHTMLX MCP or the official **Migration from Older Versions → 9.1 → 10.0** guide before relying on changed behavior.
