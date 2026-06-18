---
name: dhtmlx-vue-gantt
description: >
  Builds and integrates DHTMLX Vue Gantt into Vue 3 applications. Covers setup,
  props, templates, themes, events, @ready, refs, data.save, data.batchSave, resources, calendars, undo/redo, row reorder,
  sortorder, and Vue integration patterns for @dhtmlx/trial-vue-gantt and
  @dhx/vue-gantt. Applies when working with gantt charts, project timelines,
  task dependencies, task CRUD, resource panels, working calendars, lightboxes,
  inline editors, or weekend highlighting in Vue - regardless of whether
  "DHTMLX" is mentioned by name. Provides verified API guidance rather than
  guessing Vue Gantt APIs.
paths: "**/*.vue,**/*.ts,**/*.js,package.json"
allowed-tools: Read Grep
metadata:
  tags: dhtmlx, gantt, vue, dependencies, baselines, milestones, resources
---

## Source Of Truth

Use only:
1. The current project's files, structure, and established patterns
2. DHTMLX MCP for Vue Gantt API details: https://docs.dhtmlx.com/mcp
3. Official DHTMLX Vue Gantt docs as fallback: https://docs.dhtmlx.com/gantt/integrations/vue/

Never invent props, helper names, composables, templates, callback signatures, event names, or backend behavior.

If any Vue Gantt API detail is unclear, resolve it through DHTMLX MCP before writing code.

## Preflight

Before writing code, identify:
- **package**: `@dhtmlx/trial-vue-gantt` or `@dhx/vue-gantt` - check `package.json`, imports, lockfiles
- **runtime**: Vue 3, Nuxt, Vite, or another Vue-based setup
- **ownership model**: Application-managed (default) or Gantt-managed (large datasets, Gantt-centric apps)
- **persistence**: local only, `data.save` (default, single-entity), or `data.batchSave` (bulk sync)

## Workflow

1. Confirm the installed DHTMLX Vue Gantt package and import path.
2. Decide the data ownership model before implementing features.
3. Read only the reference file needed for the task:
   - Vue integration/setup: [references/vue-integration.md](references/vue-integration.md)
   - CRUD, state, and persistence: [references/data-and-crud.md](references/data-and-crud.md)
   - Failure cases and guardrails: [references/known-failures.md](references/known-failures.md)
   - Advanced patterns (reorder, resources, undo/redo, schema): [references/advanced-patterns.md](references/advanced-patterns.md)
   - Styling, theming, CSS variables, selectors, and template-based visual customization: [references/styling-and-theming.md](references/styling-and-theming.md)
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

If MCP is not available, use the official docs at https://docs.dhtmlx.com/gantt/integrations/vue/ as fallback.

## Consult MCP First For

Consult DHTMLX MCP before using or changing:
- template callbacks you have not already verified
- helper factories such as `defineGanttConfig`, `defineGanttTemplates`, `defineGanttEvents`, or `defineInlineEditors`
- composables such as `useWorkTime`, `useGanttDatastore`, `useResourceAssignments`, `useGanttActions`, or `useGanttEvent`
- advanced props such as `batchSave`, `inlineEditors`, `modals`, `customLightbox`, `groupTasks`, or `events`
- theme names beyond the known project usage
- resource panel, calendars, or advanced layout details
- resource timeline templates and cell rendering callbacks
- workload cell templates and overload threshold logic

## Hard Rules

- The Gantt container must have explicit height.
- Use the app theme as the single source of truth.
- Prefer CSS variables and documented templates before selector-heavy overrides.
- Normalize date values before persistence.
- Build backend payloads explicitly from normalized task/link models.
- Do not use undocumented internals when a documented prop, composable, event map, `@ready`, or ref API exists.
- Do not mix Vue-managed props and imperative instance mutations unless synchronization is intentional.

## Quick Checklist

- [ ] Correct package identified
- [ ] Matching CSS import used
- [ ] Explicit height provided
- [ ] Data ownership model chosen
- [ ] Dates normalized before persistence
- [ ] Advanced APIs verified with MCP
