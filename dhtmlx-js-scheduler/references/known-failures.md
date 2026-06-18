# Known Failures

Use this file when debugging incorrect or flaky DHTMLX JavaScript Scheduler integrations.

## 1. Collapsed Scheduler Container

Problem:
- Scheduler is initialized into a container with no resolved height.

Fix:
- give the Scheduler host and necessary parents explicit height
- use `height: 100vh` or a fixed height when parent layout is uncertain

## 2. Wrong Package Or CSS Import

Problem:
- code imports Scheduler JS from one package but CSS from another
- old skin-specific CSS files are used with current v7+ skins

Fix:
- match JS and CSS package paths
- verify package registry configuration before changing imports
- inspect the installed package contents before assuming CSS file paths
- use `codebase/dhtmlxscheduler.css`

## 3. PRO Feature In Standard Build

Problem:
- edition-gated features such as Timeline, Units, Grid, Week Agenda, or multiple instances are used without verifying edition capabilities.

Fix:
- detect edition before implementation
- warn that the feature is PRO-only
- use trial/PRO package if the user wants to run the feature

## 4. Singleton Destroyed Accidentally

Problem:
- calling `scheduler.destructor()` on a Standard/GPL singleton makes Scheduler unusable until page reload.

Fix:
- for singleton cleanup, detach handlers, destroy DataProcessor, and clear data as needed
- verify available Scheduler exports and instance capabilities before using `destructor()`
- use singleton-safe cleanup when factory instances are not available

## 5. Date Format Mismatch

Problem:
- backend date strings do not match `scheduler.config.date_format`, so events parse incorrectly or disappear.

Fix:
- set `scheduler.config.date_format` before loading
- normalize backend rows into Scheduler event objects
- prefer stable ISO/timestamp serialization at persistence boundaries

## 6. Temporary ID Not Replaced

Problem:
- backend assigns real ids but Scheduler keeps temporary ids, breaking later updates/deletes.

Fix:
- return `{ id: realId }` or `{ tid: realId }` from DataProcessor create
- when saving manually, call `scheduler.changeEventId(tempId, realId)`
- ensure application state and caches also replace temporary ids after create succeeds

## 7. XSS Through Templates Or Lightbox Input

Problem:
- Scheduler templates inject return values as raw HTML; user event text executes as markup/script.

Fix:
- sanitize on backend
- HTML-escape user strings before `scheduler.parse`, `scheduler.load`, remote updates, or template returns
- validate lightbox input server-side

## 8. Readonly Treated As Security

Problem:
- client-side readonly mode is used as the only authorization control.

Fix:
- set `scheduler.config.readonly` for UX
- enforce permissions in DataProcessor/API endpoints

## 9. Backend Write Policy Missing

Problem:
- Scheduler can read data successfully, but create, update, or delete operations fail due to backend authorization or row-level security rules.

Fix:
- verify insert, update, and delete permissions separately from read access
- test Scheduler create/update/delete flows end-to-end
- enforce backend authorization for every write operation

## 10. Plugin API Used Before Plugin Activation

Problem:
- calls like `createTimelineView`, recurring lightbox sections, or readonly event fields fail.

Fix:
- call `scheduler.plugins({...})` before plugin-dependent config/API usage
- verify PRO-only plugin availability

## 11. Re-initializing On Every Data Or State Change

Problem:
- the app recreates the Scheduler (e.g. `getSchedulerInstance()` + `init()` + `createDataProcessor()`) whenever data, filters, or props change — commonly by putting all of it inside one reactive effect keyed on data.
- symptoms: visible flicker/rebuild on every update, the view snapping back to the initial date/mode, duplicate DataProcessors firing requests twice, leaked event handlers, and growing memory.

Fix:
- build the instance **once** per mount; tear it down **once** on unmount.
- reflect data changes with `scheduler.clearAll()` + `scheduler.parse(...)`, or incremental `addEvent`/`updateEvent`/`deleteEvent` (`batchUpdate` for many).
- refresh the view or re-apply a filter with `scheduler.setCurrentView()` / `scheduler.updateView()` — not by re-initializing.
- in React, separate "create once" (mount effect) from "feed data" (effect keyed on the data); read mutable values (filters, current user) through refs/getters in handlers registered once.
