---
name: openspec-converge-change
description: Verify whether a codebase fully satisfies the specs, design, proposal, and tasks of a change, without modifying code. Appends unmet requirements, partial implementations, and unrequested additions as traceable tasks in tasks.md. Use when the user wants to reconcile implementation drift or catch up tasks.md with reality.
allowed-tools: Bash(openspec:*)
license: MIT
compatibility: Requires openspec CLI.
metadata:
  author: openspec
  version: "1.0"
---

Verify whether a codebase fully converges on the intended specs, design, proposal, and tasks of a change. This is a **read-only, code-preserving** audit: never edit application/source code, specs, or design. The only artifact this workflow ever writes to is `tasks.md` (or the schema's task artifact), and only by appending new, traceable follow-up tasks for anything found to be unmet, partial, or unrequested.

### Metadata

| Field         | Value                                                                       |
|---------------|------------------------------------------------------------------------------|
| name          | `openspec-converge-change`                                                  |
| description   | Verify whether a codebase fully satisfies specs/design/proposal/tasks, without modifying code; append gaps as traceable tasks in `tasks.md`. |
| license       | MIT                                                                          |
| compatibility | Requires the `openspec` CLI                                                 |
| author        | openspec                                                                     |
| version       | 1.0                                                                          |

### Input Handling

Determine the active change using, in order:

1. **Explicit argument**: a change name passed with the command (e.g., `/opsx-converge add-auth`). Always takes precedence.
2. **Auto-detect**: if no argument is given, infer from conversation context (a change discussed earlier in the session), or auto-select if `openspec list --json` shows exactly one active change.
3. **User selection**: if the change is still ambiguous (multiple active changes, no context to infer from), run `openspec list --json` and ask the user to pick one. Show each candidate's schema and mark changes with incomplete tasks as "(In Progress)".

Always announce the resolved choice: "Using change: <name>" and how to override it (e.g., `/opsx-converge <other>`).

### Execution Workflow

```
Select change
      ↓
Verify readiness
      ↓
Load artifacts
      ↓
Build traceability
      ↓
Detect gaps
      ↓
Classify gaps
      ↓
Generate tasks
      ↓
Generate report
```

**Store selection:** If the user names a store (a store is a standalone OpenSpec repo registered on this machine) or the work lives in one, run `openspec store list --json` to discover registered store ids, then pass `--store <id>` on the commands that read or write specs and changes (`new change`, `status`, `instructions`, `list`, `show`, `validate`, `archive`, `doctor`, `context`, `schemas`, `view`). Once selected, treat `--store <id>` as sticky for the rest of the workflow. Every unscoped example of those commands below is shorthand: before running it, append the flag. Other commands do not take the flag. Without a store, commands act on the nearest local `openspec/` root.

1. **Select change** — resolve the target change per [Input Handling](#input-handling).

2. **Verify readiness**

   ```bash
   openspec status --change "<name>" --json
   ```
   Parse the JSON to understand:
   - `schemaName`: the workflow being used (e.g., "spec-driven")
   - `planningHome`, `changeRoot`, `artifactPaths`, and `actionContext`: path and scope context
   - Which artifacts exist for this change, and which one is the task artifact (typically `tasks`)
   - Whether the change is already archived (see [Edge Cases](#edge-cases))

3. **Load artifacts**

   ```bash
   openspec instructions apply --change "<name>" --json
   ```

   This returns the change directory and `contextFiles` (artifact ID -> array of concrete file paths). Read all available artifacts from `contextFiles` (proposal, specs, design, tasks, or the schema's equivalents).

4. **Build traceability**

   Build a checklist of everything the change commits to, each item tagged with its origin so every later finding can cite a reference:
   - Every requirement in the delta specs (`### Requirement:` blocks) and their scenarios (`#### Scenario:`)
   - Every decision/approach stated in `design.md` (if present)
   - Every task already listed in the task artifact, and its checkbox state
   - Any explicit scope statement in the proposal (goals and non-goals)

5. **Detect gaps**

   For each requirement, scenario, design decision, and existing task:
   - Search the codebase for implementation evidence (keywords, symbols, file/dir conventions related to the affected libs/apps)
   - Note whether evidence is absent, partial, or complete

   Separately, scan the affected areas for **unrequested additions**: code, files, or behavior that isn't traceable to any requirement, design decision, or task in this change (scope creep, leftover experiments, dead code introduced alongside the change).

   Do not modify any source, spec, or design file during this scan. This step is strictly observational.

6. **Classify gaps**

   Classify each traced item into exactly one bucket:
   - **Met**: implementation exists and matches intent
   - **Unmet**: no implementation evidence found
   - **Partial**: some implementation exists but is incomplete, diverges from the requirement/scenario, or a checked-off task (`- [x]`) doesn't hold up under inspection
   - **Unrequested**: implementation found in step 5 with no traceable origin

   Reconcile against the existing task artifact before generating new tasks:
   - Do not duplicate a follow-up task if an equivalent open task (`- [ ]`) already covers the same gap
   - If a task is checked complete (`- [x]`) but was classified Unmet or Partial, do not un-check or edit its line; instead classify a new discrepancy finding that references the original task

7. **Generate tasks**

   For every Unmet, Partial, or Unrequested finding, append a new task to the end of the task artifact (or under a dedicated trailing section if the schema's task file uses numbered sections), using this traceable format:

   ```markdown
   ## Convergence Follow-ups (<date>)

   - [ ] [UNMET] <requirement/design id> — <what is missing>. Ref: <spec/design file>:<section>
   - [ ] [PARTIAL] <requirement/design id> — <what exists> vs <what's missing>. Ref: <file>:<lines> / <spec file>:<section>
   - [ ] [UNREQUESTED] <short description> — not traceable to any requirement/design/task. Ref: <file>:<lines>
   ```

   Rules for appended tasks:
   - Each task must be a single, actionable, checkable item
   - Each task must cite a traceable reference: the spec/design/proposal location it stems from, and/or the code location where the gap or addition was found
   - Use a fresh `## Convergence Follow-ups (<date>)` heading each time this command runs, so repeated runs are distinguishable and auditable; do not merge into a prior run's section
   - If nothing is found (fully converged, no unrequested additions), do not add an empty heading — skip the append entirely (see [Already-converged changes](#edge-cases))

8. **Generate report**

   ```markdown
   ## Convergence Report: <change-name>

   ### Summary
   | Category            | Count |
   |----------------------|-------|
   | Requirements Met      | X/Y   |
   | Requirements Partial  | X/Y   |
   | Requirements Unmet    | X/Y   |
   | Unrequested additions | N     |
   | Follow-up tasks added | N     |

   ### Appended to tasks.md
   - [ ] [UNMET] ...
   - [ ] [PARTIAL] ...
   - [ ] [UNREQUESTED] ...

   ### Notes
   - <any items skipped, ambiguous, or needing human judgment>
   ```

   **Final Assessment**:
   - If nothing appended: "Fully converged. No follow-up tasks added."
   - If tasks appended: "N follow-up task(s) appended to <tasks file path>. Codebase not yet fully converged with specs/design/proposal."

### Safety Constraints

This workflow is strictly read-only with respect to everything except the task artifact's tail. It is explicitly prohibited from:

- **Code implementation** — never write, edit, or delete application/source files
- **Specification modification** — never edit `specs/` delta files, requirements, or scenarios
- **Design modification** — never edit `design.md` or any recorded decision
- **Task deletion** — never remove or blank out an existing task line
- **Task reordering** — never renumber, move, or resequence existing tasks or sections
- **Scope expansion** — never introduce new requirements, decisions, or work items beyond what is traceable to existing artifacts or observed gaps; appended tasks describe gaps, they don't add new scope

If any step would require violating one of these constraints, stop and report the conflict instead of proceeding.

### Edge Cases

- **Missing artifacts**: if `contextFiles` lacks specs or design, degrade gracefully — verify only what exists (e.g., tasks-only: check task completion and unrequested additions against the proposal's stated scope) and note skipped checks in the report's Notes section.
- **Archived changes**: if `openspec status` shows the change is archived, note that the change is no longer active. Still run the audit against the archived artifacts if the user explicitly asks, but warn that appending to an archived task file is unusual and confirm with the user before writing.
- **Empty specifications**: if a delta spec file exists but contains no `### Requirement:` blocks, treat spec coverage as N/A rather than 0/0, and rely on tasks/design/proposal for traceability.
- **No implementation yet**: if the codebase shows no evidence of work started, classify every requirement as Unmet and every existing unchecked task as still Unmet — do not treat "no code found" as a failure of the audit itself, just report it plainly.
- **Already-converged changes**: if every requirement, scenario, design decision, and task is classified Met with no unrequested additions, skip the append step entirely (do not create an empty `## Convergence Follow-ups` heading) and report "Fully converged. No follow-up tasks added."
