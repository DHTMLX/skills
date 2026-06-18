# Editions

Use this file to identify which Scheduler edition is installed and decide whether a requested feature is available before scaffolding code.

## Packages And Delivery

Detect the delivery by reading project files; do not guess.

| Package / Delivery | Edition | Registry / Source | Notes |
| --- | --- | --- | --- |
| `dhtmlx-scheduler` | Standard (GPL) | public npm | Free Standard edition. Public npm contains only the Standard version. |
| `@dhx/trial-scheduler` | Professional Evaluation | public npm | 30-day trial. Full PRO surface. |
| `@dhx/scheduler` | Professional | private registry `https://npm.dhtmlx.com` | Requires Client's Area credentials + `npm login`. Full PRO surface. |

The `@dhtmlx/trial-*` namespace on public npm is reserved for framework wrappers (`@dhtmlx/trial-react-scheduler`, `@dhtmlx/trial-vue-scheduler`, `@dhtmlx/trial-angular-scheduler`). The vanilla JS package is never `@dhtmlx/trial-scheduler`.

## Detecting The Edition

Different setups expose the edition through different signals — check in this order:

1. **npm-based setup** — read `package.json` dependencies. `dhtmlx-scheduler` → Standard, `@dhx/trial-scheduler` → Trial PRO, private/local Scheduler package → PRO. Stop here when one matches.
2. **Script-tag / CDN / vendored build** — there is no `package.json` entry, so fall back to the JS file or the runtime object:
   - Open the served `dhtmlxscheduler.js` (CDN url or local copy) and look for the `@license` banner near the top. The banner names the version and the edition.
   - If the banner has been stripped (rare — only by deliberate post-processing), inspect the runtime object and consult DHTMLX MCP before determining the edition.

## PRO-Only Features

The following are unavailable in Standard (`dhtmlx-scheduler`). Source: https://docs.dhtmlx.com/scheduler/editions.html

- Timeline view
- Units view
- Week Agenda view
- Grid view
- Multisection events
- Multiple Scheduler instances on one page (`Scheduler.getSchedulerInstance()`)
- Drag-and-drop between Schedulers
- Day Timeline extension
- Tree Timeline extension

Standard supports day, week, month, year, and agenda views, event CRUD, recurring events, DataProcessor integration, dynamic loading, localization, accessibility, collision control, limits, tooltips, export, map view, touch support, keyboard navigation, and custom templates.

## Detect-And-Warn Behavior

Before scaffolding any feature from the PRO-only list above:

1. Inspect `package.json` to determine the edition.
2. If the package is `dhtmlx-scheduler`, surface a warning to the user that the requested feature is PRO-only and points to upgrade options (`@dhx/trial-scheduler` for evaluation, `@dhx/scheduler` for production).
3. Scaffold the requested code anyway. Do not silently substitute, omit, or rewrite the feature.
4. If the package is `@dhx/trial-scheduler` or `@dhx/scheduler`, no warning is needed.

This rule lets the user make an informed upgrade decision without losing the requested implementation.
