# Vue Integration

Use this file when setting up or modifying the official Vue wrapper integration.

## Installation

Trial:
```bash
npm install @dhtmlx/trial-vue-gantt
```

Commercial (requires private registry setup and login):
```bash
npm config set @dhx:registry=https://npm.dhtmlx.com
npm login --registry=https://npm.dhtmlx.com --scope=@dhx --auth-type=legacy
npm install @dhx/vue-gantt
```

The user must generate credentials in the [Client's Area](https://dhtmlx.com/clients/) and run the registry config and login commands themselves before the package can be installed. See [installation docs](https://docs.dhtmlx.com/gantt/integrations/vue/installation/) for details.

CSS import must match the installed package and must be a separate import line:

```ts
import "@dhtmlx/trial-vue-gantt/dist/vue-gantt.css";
```

or

```ts
import "@dhx/vue-gantt/dist/vue-gantt.css";
```

## Height Rule

Ensure the Gantt rendering area has a resolved height. If the container height resolves to 0, the chart will not be visible.

Valid:
```vue
<template>
  <div style="height: 600px;">
    <VueGantt :tasks="tasks" :links="links" />
  </div>
</template>
```

## Documented Props

Use documented props only:

- `tasks`
- `links`
- `resources`
- `resourceAssignments`
- `baselines`
- `markers`
- `calendars`
- `data`
- `config`
- `plugins`
- `templates`
- `locale`
- `theme`
- `filter`
- `resourceFilter`
- `modals`
- `groupTasks`
- `inlineEditors`
- `customLightbox`
- `events`

## Type Imports

Import wrapper and Gantt engine types from the Vue wrapper package itself.

Common wrapper-owned types:
- `SerializedTask`
- `SerializedLink`
- `VueGanttRef`
- `VueGanttDataConfig`
- `BatchChanges`
- `DataCallbackChange`
- `Marker`
- `WrapperCalendar`
- `GanttModals`
- `CustomLightboxProps`
- `InlineEditorComponentProps`
- `VueGanttEvents`

Common re-exported engine types:
- `Task`
- `Link`
- `GanttStatic`
- `GanttConfigOptions`
- `GanttTemplates`
- `GanttPlugins`
- `CalendarConfig`

Use `SerializedTask` and `SerializedLink` for application-owned state, store state, API responses, and initial literals. Use `Task` and `Link` inside event handlers, templates, and filters where runtime records may include Gantt-owned `$` fields.

## Events

Use the `events` map instead of inventing wrapper props for each event.

```ts
import { defineGanttEvents } from "@dhtmlx/trial-vue-gantt";

const events = defineGanttEvents({
  onTaskCreated: task => {
    console.log(task);
    return true;
  },
  onBeforeLightbox: taskId => {
    console.log(taskId);
    return true;
  }
});
```

Return `false` from cancellable events such as `onBeforeLightbox` or `onTaskCreated` when intentionally suppressing built-in behavior.

## Ready And Ref Access

`@ready` fires once after initialization and first sync.

```vue
<script setup lang="ts">
import { ref } from "vue";
import type { GanttStatic, VueGanttRef } from "@dhtmlx/trial-vue-gantt";

const ganttRef = ref<VueGanttRef | null>(null);

const onReady = (instance: GanttStatic) => {
  instance.showDate(new Date());
};

function showToday() {
  ganttRef.value?.instance?.showDate(new Date());
}
</script>

<template>
  <VueGantt ref="ganttRef" @ready="onReady" />
</template>
```

If you mutate data through the instance while also passing `tasks` and `links` as props, keep them synchronized. Otherwise Vue can overwrite those changes on the next prop sync.

## Helper Factories And Composables

Helper factories are TypeScript identity helpers:
- `defineGanttConfig(config)`
- `defineGanttTemplates(templates)`
- `defineGanttEvents(events)`
- `defineInlineEditors(inlineEditors)`

Composables take `Ref<VueGanttRef | null>`:
- `useGanttActions(ganttRef)`
- `useWorkTime(ganttRef)`
- `useGanttDatastore(ganttRef, storeName)`
- `useResourceAssignments(ganttRef)`
- `useGanttEvent(ganttRef, eventName, handler)`

Verify unfamiliar composable behavior with MCP before using it in production logic.

## Theme Rule

Use the app theme as the single source of truth.

Preferred built-in themes:
- `"terrace"`
- `"dark"`

Typical mapping:
```vue
<VueGantt :theme="appTheme === 'dark' ? 'dark' : 'terrace'" />
```

Do not introduce separate Gantt-only theme state if the app already has a global theme source.

## Config And Templates

Use `config` and `templates` props for day-to-day chart setup.

Common documented config areas:
- `config.columns`
- `config.scales`
- `config.readonly`
- `config.drag_move`
- `config.drag_resize`
- `config.resource_store`
- `config.resource_property`
- `config.lightbox`
- `config.layout`

Do not guess template callback signatures. Verify with MCP if there is any doubt.

Example:
```tsx
task_text: (_start, _end, task) => ...
```

Vue Gantt template callbacks may return either native Gantt output (`string`/HTML) or Vue VNodes created with `h()`. The wrapper accepts VNodes at runtime, but native `GanttTemplates` TypeScript types may require `as any` or a per-template cast.

## Resources And Calendars

When implementing resources:
- keep resource IDs and task assignment fields consistent
- derive display labels from the same persisted field used for editing
- include an explicit unassigned option only when needed
- keep resource grid and timeline logic tied to the same source of truth

When implementing working time:
- keep one source of truth for working and non-working time rules
- pass calendars through the `calendars` prop
- use documented helpers or templates for non-working-day styling
- if worktime logic is complex, prefer wrapper composables such as `useWorkTime(ganttRef)` over guessing engine behavior
