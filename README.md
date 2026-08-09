<!-- Explains how to install and use the MockFlow model-authoring agent skill. -->
# MockFlow Model Authoring Skill

An agent skill for inspecting, authoring, validating, simulating, and explaining distributed-system models through the MockFlow MCP.

The skill teaches coding agents the safe MockFlow authoring workflow: live discovery, graph and contract operations, reference bindings, compare-and-swap tokens, executable scenarios, deployment views, validation, and failure recovery.

## Install

Install into the current project for Codex:

```bash
npx skills add mockflow-app/mockflow \
  --skill author-mockflow-models \
  --agent codex
```

Install globally:

```bash
npx skills add mockflow-app/mockflow \
  --skill author-mockflow-models \
  --agent codex \
  --global
```

List the skills discoverable in this repository without installing:

```bash
npx skills add mockflow-app/mockflow --list
```

Update an existing installation:

```bash
npx skills update author-mockflow-models
```

## Requirements

- A coding agent supported by the `skills` CLI.
- A configured MockFlow MCP connection with access to `/mcp/v2`.
- A MockFlow Agent Access Key whose grant covers the diagrams and tools needed for the task.

Installing this repository adds the agent instructions only. It does not configure the MCP endpoint or credentials.

## Skill contents

```text
.agents/skills/author-mockflow-models/
├── SKILL.md
├── agents/
│   └── openai.yaml
└── references/
    ├── contracts.md
    ├── deployment.md
    ├── diagnostics.md
    ├── graph-authoring.md
    ├── journeys.md
    └── workflow.md
```

The main skill contains the mandatory workflow and safety rules. Reference files are loaded only when their authoring surface is relevant.
