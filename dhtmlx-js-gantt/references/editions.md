# Editions

Use this file to identify which Gantt edition is installed, pick the right install/import block, and decide whether a requested feature is available before scaffolding code.

## Editions And Packages

DHTMLX Gantt ships as **two public npm packages** plus a private PRO package. The catch: the free `dhtmlx-gantt` package **changed license and feature set at v10.0** — the package name alone no longer tells you the edition; you must also read the **version**.

| Package | Version | Edition | License | Registry |
| --- | --- | --- | --- | --- |
| `dhtmlx-gantt` | **v10.0+** | **Community (MIT)** | MIT | public npm |
| `dhtmlx-gantt` | **v9.x and earlier** | Standard (GPL) | GPL-2.0 | public npm |
| `@dhx/trial-gantt` | any | Professional Evaluation | trial (30 days) | public npm |
| `@dhx/gantt` | any | Professional | commercial | private `https://npm.dhtmlx.com` |

Starting with **v10.0** the free public distribution — npm `dhtmlx-gantt` and the repo <https://github.com/DHTMLX/gantt> — is the **MIT Community edition** and **replaces** the former GPL build. The GPL "Standard" build is the **v9.x-and-earlier** line of the same package; it is still live in existing projects, so this skill keeps supporting it.

> **The Community (MIT) edition is _not_ a strict superset of the old Standard/GPL build.** It *adds* projects, milestones, custom task types, and multiple instances, but it *drops* several features GPL had (undo/redo, markers, multiselect, unscheduled tasks, the new-task placeholder, working-time calendars, WBS). Always consult the matrix below — never assume "MIT ⊇ GPL".

The `@dhtmlx/trial-*` namespace on public npm is reserved for framework wrappers (`@dhtmlx/trial-react-gantt`, `@dhtmlx/trial-vue-gantt`). The vanilla JS package is never `@dhtmlx/trial-gantt`.

## Detecting The Edition

Check in this order; stop at the first signal that resolves.

1. **npm-based setup** — read `package.json` dependencies, then resolve the version:
   - `@dhx/trial-gantt` → Trial PRO. `@dhx/gantt` → PRO. Done.
   - `dhtmlx-gantt` → resolve the **installed** version (the lockfile, or `node_modules/dhtmlx-gantt/package.json` — not just the semver range in `package.json`):
     - major **≥ 10** → **Community (MIT)**
     - major **≤ 9** → **Standard (GPL)**
2. **Script-tag / CDN / vendored build** — there is no `package.json` entry, so fall back to the JS file or the runtime object:
   - The `@license` banner near the top of the served `dhtmlxgantt.js` names version + edition:
     - `dhtmlxGantt v.10.0.0 (MIT edition)` → Community
     - `dhtmlxGantt v.9.1.0 Standard` → GPL
     - `... Professional Edition` → PRO
   - If the banner was stripped (rare — only by deliberate post-processing), the runtime object still carries the metadata:
     - `gantt.license` — `"mit"` → Community; `"gpl"` → Standard; `"individual" | "commercial" | "enterprise" | "ultimate"` → PRO.
     - `gantt.version` — version string; confirms the **v10 MIT vs v9 GPL** split for the shared `dhtmlx-gantt` package.

## Install + Import

### Community (MIT) — v10+, free, public npm

```bash
npm install dhtmlx-gantt
```

```ts
import { gantt } from "dhtmlx-gantt";
import "dhtmlx-gantt/codebase/dhtmlxgantt.css";
```

The MIT entry also exports the `Gantt` factory, so multiple instances on one page are supported (no longer PRO-only):

```ts
import { gantt, Gantt } from "dhtmlx-gantt";
const chart = Gantt.getGanttInstance();
```

### Standard (GPL) — legacy v9.x

Same package, pinned to the v9 line:

```bash
npm install "dhtmlx-gantt@^9"
```

```ts
import { gantt } from "dhtmlx-gantt";
import "dhtmlx-gantt/codebase/dhtmlxgantt.css";
```

GPL v9 has **no** projects/milestones and **no** multiple-instance factory (`Gantt` is not exported) — see the matrix.

### Professional Evaluation (Trial)

```bash
npm install @dhx/trial-gantt
```

```ts
import { gantt } from "@dhx/trial-gantt";
import "@dhx/trial-gantt/codebase/dhtmlxgantt.css";
```

### Professional (Commercial)

```bash
npm config set @dhx:registry=https://npm.dhtmlx.com
npm login --registry=https://npm.dhtmlx.com --scope=@dhx
npm install @dhx/gantt
```

```ts
import { gantt } from "@dhx/gantt";
import "@dhx/gantt/codebase/dhtmlxgantt.css";
```

Credentials are generated in the [Client's Area](https://dhtmlx.com/clients/). The user must run the registry config and login commands before install.

## Feature Availability By Edition

Sources: the Standard-vs-PRO comparison (<https://docs.dhtmlx.com/gantt/guides/editions-comparison/>) and the MIT edition design matrix. Use the *detected edition*, not the package name, to decide.

### Available in Community (MIT) but NOT in Standard/GPL

- Projects (summary tasks) and milestones — task types, rendering, lightbox type selector, timed projects
- Custom task types (`gantt.config.types`) and auto type detection (`gantt.config.auto_types`)
- Multiple Gantt charts on one page (`Gantt.getGanttInstance()`)
- Resizing grid columns, and hiding/showing grid columns, from the UI

### Available in Standard/GPL but NOT in Community (MIT)

- **Undo/redo** (`gantt.ext.undo`) — enabling the plugin in MIT is a silent no-op
- **Markers / today line** (`gantt.ext.marker`, highlighting time slots) — no-op in MIT
- **Multi-task selection** and horizontal multi-drag (`gantt.ext.multiselect`) — no-op in MIT
- **Unscheduled tasks** — `gantt.config.show_unscheduled` is forced `false` in MIT, so date-less tasks render as normal bars instead of staying unscheduled. The code still ships, so setting `gantt.config.show_unscheduled = true` re-enables it.
- **New-task placeholder row** (`gantt.config.placeholder_task`, the "+ New task" row) — absent in MIT (built on unscheduled tasks)
- **WBS codes** (`getWBSCode`, the WBS column) — absent in MIT
- **Working-time calendars** (`gantt.config.work_time`, `gantt.setWorkTime`, per-task/multiple calendars) — **hard no-op** in MIT: all time counts as working time regardless of config (`work_time:true` and `setWorkTime` do nothing). Skip weekends/non-working time via templates and your own date math instead.

### Available in BOTH free editions (Community MIT and Standard/GPL)

- Basic task/link CRUD, tree grid, lightbox, inline editing, sorting, filtering, RTL, accessibility, smart rendering
- Tooltips, quick info, keyboard navigation, fullscreen, click-drag, drag-timeline (scroll timeline by drag)
- Export to PDF/PNG/Excel/MS Project; import from Excel/MS Project
- 32 locales, jQuery integration
- Live updates — `gantt.ext.liveUpdates` (`remoteUpdates`, `RemoteEvents`)

### PRO-only — in neither free edition

The following are unavailable in both `dhtmlx-gantt` editions (MIT and GPL):

- Resource management (resource grid, resource timeline, resource histogram, workload, assignments)
- Baselines / deadlines / custom timeline elements
- Critical path calculation and total/free slack
- Auto-scheduling
- Constraint control (`constraint_type` / `constraint_date`)
- Split tasks
- Rollups
- Tasks grouping
- Dynamic loading (branch loading)
- Calendar assignment to project / resource
- Decimal duration units (decimal hours/days)
- Custom scale — hiding time units
- Link formatter for the Predecessor inline editor
- Custom task/link layers API (`addTaskLayer`)

## Detect-And-Warn Behavior

Before scaffolding any feature, confirm it exists in the **detected edition** — not merely "is it PRO":

1. Determine the edition (steps above); for `dhtmlx-gantt`, resolve MIT vs GPL by **version**.
2. If the requested feature is unavailable in that edition, surface a warning to the user **before** scaffolding. Name the feature, the edition(s) that *do* support it, and the path:
   - PRO feature on a free install → point to `@dhx/trial-gantt` (evaluation) or `@dhx/gantt` (production).
   - A Standard/GPL feature (undo, markers, multiselect, unscheduled tasks, new-task placeholder, working-time calendars, WBS) on a **MIT Community** install → note it was dropped from the Community edition and is available in PRO (or in the legacy GPL v9 build).
3. Scaffold the requested code anyway. Do not silently substitute, omit, or rewrite the feature.
4. No warning is needed when the feature is available in the detected edition.

This rule applies in **both directions** — a PRO feature on a free install, and a Standard/GPL feature on a Community (MIT) install — and lets the user make an informed decision without losing the requested implementation.
