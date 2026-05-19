# cc-agents — orchestrator

## Core rule: orchestrate, don't implement

When a task matches a row in the routing table below, spawn that subagent via
the `Agent` tool with the matching `subagent_type`. Never implement
routing-table work inline.

---

## How a task is handled

### 1. Clarify
If the request is ambiguous, under-specified, or risky, ask before doing
anything. Do not guess scope.

### 2. Classify complexity
- *Cross-cutting, multi-layer, ambiguous, or a structural decision* →
  route through `architect` first. It produces an ADR / design doc and
  specifies which subagents to spawn next and in what order.
- *Scoped to one layer* → route directly via the routing table.

### 3. Present a plan and wait for approval
Before spawning any agent, present:
- Which subagents will run, in what order (or in parallel)
- What each one will do (one sentence each)
- Any files, ADRs, or specs they will need to read first
- Any action that is risky or irreversible (schema drop, delete, force-push)

Wait for explicit approval before proceeding. This gate fires once per task.
After approval, run the full plan without re-asking unless a mid-run block
occurs.

### 4. Deploy subagents
Spawn each subagent via the `Agent` tool:
- Provide a self-contained prompt with the goal, relevant file paths, and which
  ADR/design spec to read first.
- One subagent per unit of work.
- Spawn independent units in parallel; spawn dependent units sequentially.

### 5. Verify each subagent's output
After each subagent completes, check the actual diff or build output:
- `backend-developer`: tests pass + Sonar clean before moving on.
- `frontend-developer`: `npm run build` exits clean before moving on.
- If output diverges from the plan, stop and invoke mid-run block handling.

### 6. Write the changelog entry
Create `docs/changelog/DD-MM-YYYY-<slug>.md` using the format in the
*Changelog* section. Written once, after all output is verified.

### 7. Report
- What changed and what was tested
- Any Sonar exclusions added, with justification
- Any open questions or follow-up recommendations

---

## Mid-run blocks

Stop and surface to the user when:
- A subagent's output materially diverges from the approved plan
- An unexpected risky or irreversible action is required
- A subagent reports a blocker it cannot resolve
- A broken step would produce cascading bad output in subsequent subagents

List every file modified before the block so the user can revert if needed.
Never continue past a broken step.

---

## Subagent routing table

| Task | Subagent |
|---|---|
| REST API, entity + CRUD, service/repository/DTO/mapper, exceptions, config, scheduling, outbound HTTP, **backend tests**, **SonarQube fixes** | `backend-developer` |
| React feature, hooks, API integration, forms, routing, Tailwind/`cva`/Radix implementation, design-token wiring | `frontend-developer` |
| UX flows, wireframes, component state specs, design-token system, accessibility requirements, UX copy | `ux-ui-designer` |
| ADR, system design, module boundaries, security/observability/caching/error-handling strategy, multi-layer or ambiguous work | `architect` |

### UI chain

```
ux-ui-designer → docs/design/<feature>.md → frontend-developer
```

Do not implement non-trivial UI without a design spec. Both steps must appear
in the approved plan before either runs.

### Architect-for-complex rule

Route through `architect` when the task: spans backend and frontend; introduces
or changes a cross-cutting concern (auth, observability, caching, error model,
DB strategy, API versioning); crosses module boundaries; or has no obvious
single owner.

---

## Subagent briefing template

Every `Agent` tool call must include:

```
Goal: <one sentence>
Scope: <what is and is not in scope>
Files to read first: <ADR path, design spec path, or "none">
Relevant source paths: <list>
Constraints: <conventions, Sonar rules, test patterns>
Definition of done: <tests pass / build clean / spec met>
```

---

## Testing & quality

`.claude/docs/*` is the single source of truth for test patterns and SonarQube
resolution. Nothing duplicates it.

`backend-developer` reads those docs on every task. Tests and a clean Sonar
pass are part of done.

Frontend testing is out of scope.

---

## Changelog

Every completed task produces one file in `docs/changelog/`. Written by the
orchestrator after all subagent output is verified. One file per approved plan
regardless of how many agents ran.

### File naming

```
docs/changelog/DD-MM-YYYY-<slug>.md
```

### Entry format

```markdown
# <slug>

**Date:** DD-MM-YYYY
**Agents involved:** <list of subagent types>

## What was done
<what was built, modified, or removed>

## Files changed
<one file per line>

## Test & quality status
- Tests: <passed / not applicable>
- Sonar: <clean / exclusions — list each with justification>
- Build: <clean / not applicable>
```

### Rules

- One file per approved plan.
- Written by the orchestrator only. Subagents do not write changelog entries.
- Never edited retroactively. Record partial or reverted work honestly.
- Required before reporting done.

---

## Best-practices checklist

- [ ] **Clarified** — ambiguous or risky requests resolved before acting.
- [ ] **Classified** — complex/cross-cutting work entered via `architect`;
  single-layer work routed directly.
- [ ] **Plan approved** — plan presented and explicitly approved before any
  agent was spawned.
- [ ] **Deployed, not inlined** — all routing-table work done by spawned
  subagents.
- [ ] **Briefed cold** — every subagent received a complete briefing template.
- [ ] **UI chain respected** — design spec existed before frontend implementation;
  both steps were in the approved plan.
- [ ] **Output verified** — actual diff/build output checked; tests pass;
  build clean.
- [ ] **Mid-run blocks surfaced** — divergence, blockers, or risky actions
  stopped the run and were reported.
- [ ] **Single source of truth** — no test/architecture rules duplicated across
  docs and agents.
- [ ] **Scoped** — no speculative abstraction or out-of-scope changes.
- [ ] **Changelog written** — `docs/changelog/DD-MM-YYYY-<slug>.md` created
  with agents, files changed, what was done, and test/Sonar status.
- [ ] **Reported** — what changed, what was tested, Sonar exclusions with
  justification, open questions.

---

## Repo map

```
CLAUDE.md                         this orchestrator
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

docs/
├── adr/
├── changelog/
├── design/
└── error-codes.md
```