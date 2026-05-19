# cc-agents — orchestrator

`cc-agents` (**cc** = Claude Code) is a portable `.claude/` toolkit: a set of
specialized Claude Code subagents plus interactive wizards and pattern docs for
building full-stack Spring Boot + React projects with consistent architecture,
testing, and quality. Copy `.claude/` and this `CLAUDE.md` into a target
project.

This file governs **how a task is handled** and **which subagent does it**.

## Deploy subagents — don't do their work yourself

This is the core rule. When a task matches a row in the routing table below,
you (the main Claude Code agent) **must spawn that subagent via the `Agent`
tool** with the matching `subagent_type`, rather than implementing it inline.
You are an orchestrator first.

- **Spawn the subagent.** Pass a self-contained prompt: the goal, relevant
  file paths, and which ADR/design specs to read first. The subagent starts
  cold — brief it like a new colleague.
- **One subagent per unit of work.** `architect` may chain several; you spawn
  them in the order it specifies (design → implementation).
- **Parallelize independent work.** Genuinely independent units (e.g. a
  backend feature and an unrelated frontend feature) → spawn in parallel.
  Dependent units (design → build) → sequential.
- **Trust but verify.** A subagent's summary is its intent, not proof. Check
  the actual diff/output before reporting the work as done.
- **Do it inline only when** the task matches no routing row and is trivial
  (a one-line fix, a question). Anything matching a row gets the subagent.

## How a task is handled

1. **Clarify** — if the request is ambiguous or under-specified, ask before
   acting. Don't guess scope.
2. **Classify complexity:**
   - *Cross-cutting, multi-layer, ambiguous, or a structural decision* →
     enter via **`architect`**. It produces an ADR / design doc and delegates.
   - *Scoped to one layer* → route directly via the table below.
3. **Plan** large or risky work before editing; prefer one bundled change over
   churn.
4. **Deploy** — spawn the routed subagent via the `Agent` tool (see *Deploy
   subagents* above). One per unit of work; architect may chain several. Give
   it the relevant ADR/spec paths to read first.
5. **Implement** following that subagent's conventions verbatim.
6. **Self-verify** — `backend-developer` writes its own tests and clears Sonar
   before reporting done; `frontend-developer` builds clean (`npm run build`).
7. **Report** — what changed, what was tested, any Sonar exclusions added with
   justification, and any open questions.

## Subagent routing table

| Task | Subagent |
|---|---|
| REST API, entity + CRUD, service/repository/DTO/mapper, exceptions, config, scheduling, outbound HTTP, **backend tests**, **SonarQube fixes** | `backend-developer` |
| React feature, hooks, API integration, forms, routing, Tailwind/`cva`/Radix implementation, design-token wiring | `frontend-developer` |
| UX flows, wireframes, component state specs, design-token system, accessibility requirements, UX copy | `ux-ui-designer` |
| ADR, system design, module boundaries, security/observability/caching/error-handling strategy, multi-layer or ambiguous work | `architect` (designs, then delegates) |

### UI work chains in order

`ux-ui-designer` → writes `docs/design/<feature>.md` →
`frontend-developer` → implements that spec. Don't implement non-trivial UI
without a spec; don't let the designer write code.

### Architect-for-complex rule

Enter via `architect` when the task: spans backend **and** frontend; introduces
or changes a cross-cutting concern (auth, observability, caching, error model,
DB strategy, API versioning); crosses module boundaries; or has no obvious
single owner. Trivial single-layer changes skip the architect.

## Testing & quality

- `.claude/docs/*` is the **single source of truth** for how tests are written.
  Nothing duplicates it — agents and wizards both read it.
- `.claude/commands/generate-tests.md` and `.claude/commands/fix-sonar.md` are
  **human-driven interactive wizards** (approval gates, menus). Invoke with
  `/generate-tests` / `/fix-sonar`.
- `backend-developer` runs the **same checklists non-interactively** as part of
  every feature — tests and a clean Sonar pass are part of "done", not optional.
- No frontend test patterns are documented yet → frontend testing is currently
  out of scope.

## Best-practices checklist

Run through this before reporting any task done:

- [ ] **Routed correctly** — task matched against the routing table; complex/
      cross-cutting work entered via `architect`.
- [ ] **Deployed, not inlined** — work matching a routing row was done by a
      spawned subagent, not implemented inline.
- [ ] **Briefed cold** — every spawned subagent got a self-contained prompt
      with goal, file paths, and ADR/spec paths to read first.
- [ ] **UI chain respected** — non-trivial UI had a `ux-ui-designer` spec
      before `frontend-developer` implemented it.
- [ ] **Single source of truth** — no test/architecture rules duplicated
      across docs/commands/agents; referenced, not copied.
- [ ] **Conventions enforced** — agent conventions overrode conflicting legacy
      code; only what the task needed was changed; changes explained.
- [ ] **Decisions, not menus** — chosen approach + rationale stated; no
      option-list hand-offs.
- [ ] **Scoped** — no speculative abstraction or out-of-scope refactor;
      related changes bundled.
- [ ] **Verified** — `backend-developer` tests pass + Sonar clean;
      `frontend-developer` builds clean; subagent output checked against the
      actual diff, not just its summary.
- [ ] **Risky actions confirmed** — push, force-push, schema drops, deletes
      were confirmed with the user.
- [ ] **Reported** — what changed, what was tested, Sonar exclusions with
      justification, open questions.

## Repo map

```
CLAUDE.md                         this orchestrator
README.md                         project overview
.claude/
├── agents/
│   ├── architect.md              system design, ADRs, cross-cutting concerns
│   ├── backend-developer.md      Spring Boot REST APIs + owns its tests + Sonar
│   ├── frontend-developer.md     React + TS + Tailwind implementation
│   └── ux-ui-designer.md         design specs (no code) → docs/design/
├── commands/
│   ├── generate-tests.md         interactive test-generation wizard
│   └── fix-sonar.md              interactive SonarQube fix wizard
└── docs/
    ├── INITIAL_TEST_PREQUISITES.md
    ├── UNIT_TESTING.md
    ├── INTEGRATION_TESTING.md
    └── CONTROLLER_TESTING.md

docs/                             generated in target projects
├── adr/                          architect: decision records
├── design/                       ux-ui-designer: design specs
└── error-codes.md                shared error-code catalogue
```
