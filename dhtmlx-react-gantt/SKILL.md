---
name: dhtmlx-react-gantt
description: >
  Builds and integrates DHTMLX React Gantt into React applications. Covers setup, props,
  templates, themes, data.save, data.batchSave, resources, calendars, undo/redo, row reorder,
  sortorder, and React integration patterns for @dhtmlx/trial-react-gantt and @dhx/react-gantt.
  Applies when working with gantt charts, project timelines, task dependencies, task CRUD,
  resource panels, working calendars, lightboxes, or weekend highlighting in React — regardless
  of whether "DHTMLX" is mentioned by name. Provides verified API guidance rather than guessing
  React Gantt APIs.
paths: "**/*.tsx,**/*.jsx,**/*.ts,**/*.js,package.json"
allowed-tools: Read Grep
metadata:
  tags: dhtmlx, gantt, dependencies, baselines, milestones, resources
---

## Source Of Truth

Use only:
1. The current project's files, structure, and established patterns
2. DHTMLX MCP for React Gantt API details: https://docs.dhtmlx.com/mcp
3. Official DHTMLX React Gantt docs as fallback: https://docs.dhtmlx.com/gantt/integrations/react/

Never invent props, hooks, templates, callback signatures, event names, or backend behavior.

If any React Gantt API detail is unclear, resolve it through DHTMLX MCP before writing code.

## Preflight

Before writing code, identify:
- **package**: `@dhtmlx/trial-react-gantt` or `@dhx/react-gantt` — check `package.json`, imports, lockfiles
- **runtime**: React, Next.js (needs `"use client"`), Remix, or other React-based setup
- **ownership model**: React-managed (default) or Gantt-managed (large datasets, Gantt-centric app)
- **persistence**: local only, `data.save` (default, single-entity), or `data.batchSave` (bulk sync)

## Workflow

1. Confirm the installed DHTMLX React Gantt package and import path.
2. Decide the data ownership model before implementing features.
3. Read only the reference file needed for the task:
   - React integration/setup: [references/react-integration.md](references/react-integration.md)
   - CRUD, state, persistence, wrapper task types, and v9 -> v10 migration: [references/data-and-crud.md](references/data-and-crud.md)
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

If MCP is not available, use the official docs at https://docs.dhtmlx.com/gantt/integrations/react/ as fallback.

## Consult MCP First For

Consult DHTMLX MCP before using or changing:
- template callbacks you have not already verified
- wrapper hooks such as `useWorkTime`, `useGanttDatastore`, or `useResourceAssignments`
- advanced props such as `batchSave`, `inlineEditors`, `modals`, `customLightbox`, or `groupTasks`
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
- Do not use undocumented internals when a documented prop, hook, or ref API exists.
- Do not mix React-managed props and imperative instance mutations unless synchronization is intentional.

## Quick Checklist

- [ ] Correct package identified
- [ ] Matching CSS import used
- [ ] Explicit height provided
- [ ] Data ownership model chosen
- [ ] Dates normalized before persistence
- [ ] Advanced APIs verified with MCP
