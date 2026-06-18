# Styling And Theming

Use this file when theming Scheduler, matching an app design system, or styling events, scales, lightboxes, blocked time, and advanced views.

## Contents

- Styling Priority
- Skins
- Theme Variables First
- Use Config For Size And Geometry
- Templates
- Data-Driven Styling
- Common Selectors And Escape Hatches
- Practical Guardrails

## Styling Priority

Use this order by default:
1. built-in skins via `scheduler.skin` or `scheduler.setSkin`
2. CSS variables for global theme alignment
3. config for size and geometry
4. templates for semantic or data-driven styling
5. direct CSS selectors only for structural exceptions

## Skins

Built-in skins:
- `"terrace"` (default)
- `"dark"`
- `"material"`
- `"flat"`
- `"contrast-black"`
- `"contrast-white"`

Starting from Scheduler v7.0, all skins are bundled in `dhtmlxscheduler.css`.

Set the skin before calling `scheduler.init()`:

```ts
scheduler.skin = appTheme === "dark" ? "dark" : "terrace";

scheduler.init("scheduler_here", new Date(), "week");
```

Or switch the skin at runtime:

```ts
scheduler.setSkin("dark");
```

Use the application theme as the single source of truth. Avoid introducing a Scheduler-specific theme switch when a global theme already exists.

## Theme Variables First

CSS variables are the preferred customization path.

Define shared variables at `:root` so inheritance works correctly across the whole component:

```css
:root {
  --dhx-scheduler-base-colors-primary: #2563eb;
  --dhx-scheduler-container-background: #ffffff;
  --dhx-scheduler-container-color: #0f172a;
  --dhx-scheduler-event-background: #2563eb;
  --dhx-scheduler-event-color: #ffffff;
}
```

### High-Impact Variables

Typography and surfaces:
- `--dhx-scheduler-font-family`
- `--dhx-scheduler-font-size`
- `--dhx-scheduler-container-background`
- `--dhx-scheduler-container-color`
- `--dhx-scheduler-popup-background`
- `--dhx-scheduler-popup-color`

Primary and semantic colors:
- `--dhx-scheduler-base-colors-primary`
- `--dhx-scheduler-base-colors-background`
- `--dhx-scheduler-base-colors-text-base`
- `--dhx-scheduler-base-colors-border`

Events:
- `--dhx-scheduler-event-background`
- `--dhx-scheduler-event-color`
- `--dhx-scheduler-event-border`

Use CSS variables for application-wide theming and design system alignment.

Per-event `color` and `textColor` override event CSS variables through inline CSS variables. Use them only when colors are true event data; use classes and templates for semantic styling.

## Templates

Templates should be the primary customization mechanism for data-driven content.

```ts
scheduler.templates.event_text = (start, end, event) => {
  return escapeHtml(event.text);
};
```

Common template areas:
- event text and content
- event CSS classes
- hour scale labels
- quick info and tooltip content
- agenda view rendering
- units view rendering
- year view rendering

## Use Config For Size And Geometry

Some visible styling is controlled by Scheduler configuration, not CSS:

```ts
scheduler.xy.scale_width = 60;
scheduler.xy.scale_height = 25;
scheduler.xy.bar_height = 24;
scheduler.xy.scroll_width = 18;

scheduler.config.hour_size_px = 90;
```

Common geometry settings:
- `scheduler.xy.scale_width`
- `scheduler.xy.scale_height`
- `scheduler.xy.bar_height`
- `scheduler.xy.scroll_width`
- `scheduler.xy.month_scale_height`
- `scheduler.xy.min_event_height`
- `scheduler.config.hour_size_px`

Apply geometry-related settings before calling `scheduler.init()`.

## Data-Driven Styling

When styling depends on event data or time cell data, return CSS classes from templates and style those classes with CSS variables.

```ts
scheduler.templates.event_class = (_start, _end, event) => {
  return event.priority === "high" ? "event-priority-high" : "";
};
```

```css
.event-priority-high {
  --dhx-scheduler-event-background: #dc2626;
  --dhx-scheduler-event-color: #fff;
}
```

For background time cells in Day and Week views, use documented templates such as `scheduler.templates.time_slot_class` rather than broad selectors.

### Events

Prefer `event_class` to style groups of events by priority, status, owner, or any other semantic field:

```js
scheduler.templates.event_class = (_start, _end, event) => {
  return event.priority ? `priority_${event.priority}` : "";
};
```

```css
.priority_high {
  --dhx-scheduler-event-background: #dc2626;
  --dhx-scheduler-event-color: #ffffff;
}
```

Supported event color shortcut fields:
- `color`
- `textColor`

Important caveat:
- these fields are applied by Scheduler directly to the event container and text
- they can override class-based event styling depending on CSS precedence

Use shortcut color fields only when the app truly stores per-event visual values as data. For reusable semantic styling, `event_class` is usually safer.

Advanced pattern:
- if colors come from backend-driven entities such as owners, calendars, or stages, generate CSS classes from the loaded dataset and return those classes from `event_class`

## Common Selectors And Escape Hatches

Use these when skins, variables, and templates are not enough.

Scheduler structure:
- `.dhx_cal_container`
- `.dhx_cal_navline`
- `.dhx_cal_header`
- `.dhx_cal_data`
- `.dhx_cal_tab`

Events:
- `.dhx_cal_event`
- `.dhx_cal_event_line`
- `.dhx_cal_event_clear`

Mini calendar:
- `.dhx_month_head`
- `.dhx_calendar_click`
- `.dhx_month_head.dhx_year_event`
- `.dhx_now`

Lightbox:
- `.dhx_cal_light`

Prefer CSS variables and templates over direct selectors whenever possible.

Scope overrides to a wrapper class when multiple Scheduler instances or other DHTMLX widgets exist on the page.

## Practical Guardrails

- Prefer Scheduler CSS variables for theme-wide restyling before touching selectors.
- Prefer templates when styling depends on event or time cell data.
- Prefer scoped selectors over global overrides when changing only one Scheduler area.
- Apply geometry changes through documented `scheduler.xy` and `scheduler.config` settings instead of CSS.
- Keep skin changes wired to the application theme.
- Remember that the Material skin requires Roboto to be imported manually.
- Avoid selector-heavy overrides that depend on internal DOM structure.
