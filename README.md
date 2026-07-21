# DHTMLX Skills

AI coding assistants such as Claude Code, Cursor, or Codex can generate DHTMLX code, but they often make mistakes with specialized APIs - wrong prop names, missing CSS imports, incorrect callback signatures, or mixing incompatible data patterns. The skills in this repository teach the assistant the correct integration patterns, decision points, and known pitfalls before it writes code.

Skills pair with the DHTMLX MCP server, which provides real-time API reference. Skills focus on patterns and failure prevention; MCP covers method signatures, properties, and configuration options.

- Official site: [dhtmlx.com](https://dhtmlx.com/)
- MCP server docs: [docs.dhtmlx.com/suite/guides/ai](https://docs.dhtmlx.com/suite/guides/ai/)
- MCP server endpoint (for your agent's MCP config): `https://docs.dhtmlx.com/mcp`

## Skills

### `dhtmlx-js-gantt`

Building and integrating the core DHTMLX JavaScript Gantt in JavaScript and TypeScript applications. Recognises all delivery channels (`dhtmlx-gantt` Standard/GPL, `@dhx/trial-gantt`, `@dhx/gantt`, `<script>`/CDN) and adapts setup, data, and theming guidance to each.

### `dhtmlx-react-gantt`

Building and integrating DHTMLX React Gantt (`@dhtmlx/trial-react-gantt` and `@dhx/react-gantt`) - wrapper-specific setup, data ownership and persistence patterns, and theming approach, with known pitfalls extracted from real projects.

### `dhtmlx-angular-gantt`

Building and integrating DHTMLX Angular Gantt (`@dhtmlx/trial-angular-gantt` and `@dhx/angular-gantt`) - wrapper-specific setup, data ownership and persistence patterns (`data.save` / `data.batchSave`), and theming approach, with known failure modes and concrete fixes.

### `dhtmlx-js-scheduler`

Building and integrating the core DHTMLX JavaScript Scheduler in JavaScript and TypeScript applications. Recognises all delivery channels (`dhtmlx-scheduler` Standard/GPL, `@dhx/trial-scheduler`, `@dhx/scheduler`, `<script>`/CDN) and adapts setup, data, and theming guidance to each.

### `dhtmlx-react-scheduler`

Building and integrating DHTMLX React Scheduler (`@dhtmlx/trial-react-scheduler` and `@dhx/react-scheduler`) - wrapper setup, event state and persistence, scheduler views, resources, lightboxes, and conflict checks.

## Installation

```bash
npx skills add DHTMLX/skills --skill dhtmlx-js-gantt
npx skills add DHTMLX/skills --skill dhtmlx-react-gantt
npx skills add DHTMLX/skills --skill dhtmlx-angular-gantt
npx skills add DHTMLX/skills --skill dhtmlx-js-scheduler
npx skills add DHTMLX/skills --skill dhtmlx-react-scheduler
```

Or copy manually (adjust the target directory for your agent - for example `.cursor/skills/` for Cursor):

```bash
git clone https://github.com/DHTMLX/skills
cp -r skills/dhtmlx-js-gantt ~/.claude/skills/
cp -r skills/dhtmlx-react-gantt ~/.claude/skills/
cp -r skills/dhtmlx-angular-gantt ~/.claude/skills/
cp -r skills/dhtmlx-js-scheduler ~/.claude/skills/
cp -r skills/dhtmlx-react-scheduler ~/.claude/skills/
```

## Requirements

- An AI coding agent that supports the Agent Skills standard (Claude Code, Cursor, Copilot, Codex, Gemini CLI, and others)
