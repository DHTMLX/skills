# Known Failures

Use this file when debugging incorrect or flaky DHTMLX Vue Gantt integrations.

## 1. Theme Drift

Problem:
- the shell and Gantt use different dark-mode sources

Fix:
- use one shared app theme source
- map that source to the Gantt `theme` prop
- do not create separate Gantt-only theme state

## 2. Date Serialization Bugs

Problem:
- `data.save` and `data.batchSave` handlers can emit formatted date strings rather than real `Date` instances
- calling `toISOString()` directly can fail or produce unstable persistence behavior

Fix:
- normalize date inputs before persistence
- handle both `Date` and string inputs in the normalization layer
- define `format_date` / `parse_date` templates to keep date round-tripping consistent

## 3. Stale Callback State

Problem:
- `data.save` handlers capture old task/link arrays or old store snapshots

Fix:
- read the current Vue state inside the callback (`ref.value`, current store state, or fresh query/cache snapshot)
- do not build persistence logic from stale setup-time closures

## 4. Undo/Redo Persistence Gaps

Problem:
- local undo/redo changes state but not persistence, so reload restores the wrong data

Fix:
- if the app has local history, undo/redo must also update backend persistence or trigger correct rehydration behavior

## 5. Wrong Template Callback Signature

Problem:
- a template callback is wired with the wrong parameter order or wrong argument assumptions

Fix:
- verify the exact documented signature with MCP
- do not infer template arguments from memory

## 6. Implicit Height Containers

Problem:
- Gantt is mounted into a container without real height, so rendering is broken or collapsed

Fix:
- give the Gantt host and needed parents explicit height

## 7. Mixed Ownership Bugs

Problem:
- Vue props and imperative Gantt instance mutations both modify the same data without synchronization

Fix:
- choose the ownership model first
- if imperative mutations are required, keep Vue refs synchronized deliberately
