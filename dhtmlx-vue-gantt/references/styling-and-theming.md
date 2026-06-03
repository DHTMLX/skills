# Styling And Theming

Use this file when theming Gantt, matching an app design system, or styling tasks, links, rows, and resource views.

## Contents
- Styling Priority
- Theme Variables First
- Use Config For Size And Geometry
- Data-Driven Styling
- Vue Components In Templates
- Common Selectors And Escape Hatches
- Practical Guardrails

## Styling Priority

Use this order by default:
1. CSS variables for global theme alignment
2. templates and config for semantic or data-driven styling
3. Vue VNodes from `h()` when templates need interactive or composable Vue UI
4. direct CSS selectors only for structural exceptions

## Theme Variables First

CSS variables are the preferred customization path.

Define shared variables at `:root` so inheritance works correctly across the whole component:

```css
:root {
  --dhx-gantt-base-colors-primary: #2563eb;
  --dhx-gantt-container-background: #ffffff;
  --dhx-gantt-container-color: #0f172a;
  --dhx-gantt-task-background: #2563eb;
  --dhx-gantt-task-color: #ffffff;
}
```

### High-Impact Variables

Typography and surfaces:
- `--dhx-gantt-font-family`
- `--dhx-gantt-font-size`
- `--dhx-gantt-container-background`
- `--dhx-gantt-container-color`
- `--dhx-gantt-scale-background`
- `--dhx-gantt-scale-color`
- `--dhx-gantt-offtime-background`
- `--dhx-gantt-popup-background`
- `--dhx-gantt-popup-color`
- `--dhx-gantt-popup-border`

Primary and semantic colors:
- `--dhx-gantt-base-colors-primary`
- `--dhx-gantt-base-colors-warning`
- `--dhx-gantt-base-colors-error`
- `--dhx-gantt-base-colors-success`
- `--dhx-gantt-base-colors-text-base`
- `--dhx-gantt-base-colors-text-on-fill`
- `--dhx-gantt-base-colors-border`
- `--dhx-gantt-base-colors-border-light`
- `--dhx-gantt-base-colors-select`
- `--dhx-gantt-base-colors-hover-color`

Task and project bars:
- `--dhx-gantt-task-background`
- `--dhx-gantt-task-color`
- `--dhx-gantt-project-background`
- `--dhx-gantt-project-color`
- `--dhx-gantt-milestone-background`

Rows, links:
- `--dhx-gantt-task-row-border`
- `--dhx-gantt-task-row-background`
- `--dhx-gantt-task-row-background--odd`
- `--dhx-gantt-link-background`

### Map App Tokens To Gantt Tokens

Use the app theme as the source of truth instead of inventing Gantt-only colors:
- app accent -> `--dhx-gantt-base-colors-primary`
- app surface -> `--dhx-gantt-container-background`
- app body text -> `--dhx-gantt-container-color` and `--dhx-gantt-base-colors-text-base`
- app border -> `--dhx-gantt-base-colors-border` and `--dhx-gantt-base-colors-border-light`
- app selected state -> `--dhx-gantt-base-colors-select`
- app hover state -> `--dhx-gantt-base-colors-hover-color`
- app on-accent text -> `--dhx-gantt-task-color` and `--dhx-gantt-project-color`

## Use Config For Size And Geometry

Some visible styling is controlled by Gantt config, not by CSS:
- `config.row_height`
- `config.scale_height`
- `config.link_line_width`
- `config.link_radius`
- `config.link_arrow_size`

## Data-Driven Styling

When styling depends on task, link, resource, or calendar data, prefer templates.

### Tasks

Prefer `task_class` to style groups of tasks by priority, status, owner, or any other semantic field:

```ts
const templates = defineGanttTemplates({
  task_class: (_start, _end, task) => {
    return task.priority ? `priority_${task.priority}` : "";
  }
});
```

```css
.gantt_task_line.priority_high {
  --dhx-gantt-task-background: #dc2626;
  --dhx-gantt-task-color: #ffffff;
}
```

Supported task color shortcut fields:
- `color`
- `textColor`
- `progressColor`

Important caveat:
- these fields are applied as inline styles
- inline styles have the highest priority
- they can suppress critical-path styling or other CSS overrides

Use inline color fields only when the app truly stores per-task visual values as data. For reusable semantic styling, `task_class` is usually safer.

Advanced pattern:
- if colors come from backend-driven entities such as owners or stages, generate CSS classes from the loaded dataset and return those classes from `task_class`

### Links

Prefer `link_class` to style links by dependency type or link metadata:

```ts
const templates = defineGanttTemplates({
  link_class: link => {
    return link.type === "0" ? "finish_to_start" : "";
  }
});
```

```css
.gantt_task_link.finish_to_start {
  --dhx-gantt-link-background: #2563eb;
}
```

Supported link color shortcut field:
- `color`

The same caveat applies:
- link `color` becomes inline style
- inline style overrides other CSS
- critical-link styling and custom classes may not show as expected when inline color is present

### Rows And Timeline Cells

Use templates for semantic row or cell styling:
- `grid_row_class` for specific grid rows
- `timeline_cell_class` for timeline cells such as weekends or non-working time

Use selectors alone only when the styling is purely structural and not data-driven.

## Vue Components In Templates

The Vue wrapper can mount Vue VNodes returned from template functions. Use this when a grid cell, task label, tooltip, or scale cell needs a Vue component.

```ts
import { h } from "vue";

const templates = {
  task_text: (_start, _end, task) =>
    h(TaskTextBadge, { task, onToggle: () => toggleCompleted(task.id) })
} as any;
```

The native `GanttTemplates` type may require `as any` or a per-template cast because many template return types are declared as `string | number | void`, even though the Vue wrapper accepts VNodes at runtime.

## Common Selectors And Escape Hatches

Use these when variables or templates are not enough:

Task bars:
- `.gantt_task_line`
- `.gantt_task_progress`
- `.gantt_task_line.gantt_bar_project`
- `.gantt_task_line.gantt_selected`

Grid rows and cells:
- `.gantt_row`
- `.gantt_row.odd`
- `.gantt_row.gantt_selected`
- `.gantt_row .gantt_cell[data-column-name="start_date"]`
- `.gantt_grid_head_cell[data-column-id="start_date"]`

Timeline cells:
- `.gantt_task_row`
- `.gantt_task_row.odd`
- `.gantt_task_cell`
- `.gantt_scale_cell`

Resource panel scoping:
- `.resourceGrid_cell`
- `.resourceTimeline_cell`
- `.resourceHistogram_cell`

Resource views reuse many of the same selectors as the main grid and timeline. Scope overrides with the resource layout cell class when the resource panel should look different from the main chart.

## Practical Guardrails

- Prefer CSS variables for theme-wide restyling before touching selectors.
- Prefer templates when styling depends on task, link, resource, or calendar data.
- Prefer scoped selectors over global overrides when changing only resource views or only one structural area.
- In Vue SFCs, `scoped` styles may require `:deep(...)` to target Gantt-generated DOM elements.
- Do not change `row_height` or `scale_height` in CSS.
- Use link config for line thickness, arrow size, and corner radius.
