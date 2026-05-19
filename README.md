# cc-agents

`cc-agents` is a portable `.claude/` toolkit of specialized Claude Code
subagents and pattern docs for building **full-stack Spring Boot + React**
projects with consistent architecture, testing, and code quality.

> A fully agentic multi-agent dev suite — backend, frontend, UX/UI design, and
> architecture. Work is done by deploying subagents, not interactive commands.

## What it is

Drop `cc-agents` into any Spring Boot (and/or React) project and you get:

- **Four specialized subagents** that build code following strict, documented
  conventions.
- **An orchestrator (`CLAUDE.md`)** that routes your task to the right subagent
  and enforces a consistent task-handling flow.
- **Pattern docs** that are the single source of truth for how tests are
  written and how SonarQube issues are resolved.

## Subagents

| Subagent | Owns | Output |
|---|---|---|
| `architect` | System design, ADRs, module boundaries, cross-cutting concerns (security, observability, caching, error handling) | ADRs + design docs in `docs/`; delegates implementation |
| `backend-developer` | Feature-based Spring Boot REST APIs **and their tests + SonarQube** | Production code + unit/integration/controller tests per `.claude/docs/*` |
| `ux-ui-designer` | UX flows, wireframes, component states, design-token system, accessibility, UX copy | Markdown design specs in `docs/design/` (no code) |
| `frontend-developer` | React + TypeScript implementation of design specs | Feature code, hooks, services, Tailwind/`cva`/Radix UI |

UI work chains: `ux-ui-designer` writes the spec → `frontend-developer`
implements it. Complex/cross-cutting work enters via `architect` first.

## Pattern docs (single source of truth)

| File | Purpose |
|---|---|
| `.claude/docs/INITIAL_TEST_PREQUISITES.md` | Test infrastructure setup |
| `.claude/docs/UNIT_TESTING.md` | Service-isolation unit-test patterns |
| `.claude/docs/INTEGRATION_TESTING.md` | HTTP + DB + WireMock integration patterns |
| `.claude/docs/CONTROLLER_TESTING.md` | Web-layer validation-test patterns |
| `.claude/docs/SONARQUBE.md` | SonarQube resolution standards |

`backend-developer` reads these directly. They are never duplicated into agent
files.

## How to use

1. Copy the `.claude/` folder **and** `CLAUDE.md` into your project root.
2. Open the project in Claude Code.
3. Describe your task normally — `CLAUDE.md` routes it to the right subagent,
   or invoke a subagent explicitly.
4. For new UI: ask for a design first (`ux-ui-designer`), then implementation
   (`frontend-developer`).
5. For anything cross-cutting: start with `architect`.

## Repository structure

```
CLAUDE.md                         orchestrator: task handling + routing
README.md                         this file
.claude/
├── agents/
│   ├── architect.md
│   ├── backend-developer.md
│   ├── frontend-developer.md
│   └── ux-ui-designer.md
└── docs/
    ├── INITIAL_TEST_PREQUISITES.md
    ├── UNIT_TESTING.md
    ├── INTEGRATION_TESTING.md
    ├── CONTROLLER_TESTING.md
    └── SONARQUBE.md
```

## Renaming the repository

The project identity is now `cc-agents`. Renaming the **GitHub remote** and the
**on-disk directory** is a manual step (it can't be done safely from inside a
git worktree):

```
# on GitHub: Settings → rename repo to cc-agents, then locally:
git remote set-url origin <new-cc-agents-url>
# optionally rename the working directory to cc-agents
```

All in-repo references already use the `cc-agents` name.
