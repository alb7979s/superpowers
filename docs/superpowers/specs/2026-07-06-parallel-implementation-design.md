# Parallel Implementation + Verify/Review Extension — Design Spec

**Date:** 2026-07-06
**Status:** Approved (brainstorming), pending implementation plan
**Plugin:** superpowers-home

## Goal

Extend the superpowers workflow so that, after a plan exists, implementation can
run **independent tasks in parallel** (each in its own git worktree), enforce
**config-driven lint/build/test gates** at every task boundary, and finish with a
**parallel code / performance / security review** whose findings are fixed
automatically by severity. All additions are **new, modular skills** so the
forked stock skills stay close to upstream and remain mergeable.

## Non-Goals

- Modifying the stock `subagent-driven-development` or `executing-plans` skills
  (they remain as-is, selectable execution options).
- Baking any project-specific command (e.g. `ktlint`) into skill bodies.
  Project specifics live in each project's config, not in the plugin.
- Parallelizing tightly-coupled tasks. Integration stays sequential for safety.

## Where It Fits

```
brainstorming → writing-plans → [execution choice]
                                   ├ 1. subagent-driven-development (stock, sequential)
                                   ├ 2. executing-plans (stock)
                                   └ 3. parallel-implementation (NEW)
                                         → finishing-a-development-branch (stock)
```

The ONLY edit to a stock skill: `writing-plans` gains one line in its Execution
Handoff offering option 3.

## Components

### 1. `parallel-implementation` skill (orchestrator)

Reads the plan and drives execution:

1. **Build a DAG** from each task's `Interfaces: Consumes/Produces` (dependency
   edges) and `Files:` (any file overlap forces sequential ordering).
2. **Split into waves** — tasks with no unmet dependency and no file overlap form
   one wave and run in parallel.
3. **Per task in a wave:** create a dedicated `git worktree` (via stock
   `using-git-worktrees`), dispatch a fresh implementer subagent (stock
   `implementer-prompt.md`), require `verify-gates` to pass, then commit. A stock
   task review (spec + quality) runs as in subagent-driven-development.
4. **Sequential integration:** the orchestrator merges each completed task's
   worktree branch into the working branch one at a time. Merge/integration
   conflicts are resolved by the orchestrator, not the implementers.
5. Proceed to the next wave once the current wave is integrated.
6. After the final wave, invoke `review-triad`, then hand off to
   `finishing-a-development-branch`.

**Safety rationale:** parallel safety depends on the accuracy of the plan's
`Files`/`Interfaces` metadata. Sequential integration with orchestrator-owned
conflict resolution is the backstop when that metadata is imperfect.

Model selection follows the stock guidance (cheap model for mechanical tasks,
capable model for integration/review). Models are always specified explicitly
when dispatching.

### 2. `verify-gates` skill (config-driven lint/build/test)

- Reads `.claude/verify.json` from the project root (schema below). If absent,
  attempts lightweight autodetection (gradle/npm) and, failing a confident
  guess, asks the user once and offers to write the config.
- Selects only the **targets relevant to the changed files** (e.g. changes under
  `backend/` run only the `backend` target).
- Runs `lint` → `build` → `test` for each relevant target.
- **Any failure triggers a fix loop regardless of severity.** After a configurable
  number of attempts (default 3) without success, escalate to the user.
- Invoked at every task boundary and after each integration.

### 3. `review-triad` skill (parallel review + severity-based fix)

- At branch end, dispatches **three reviewers in parallel**: code (stock
  `code-reviewer.md`), performance (`perf-reviewer.md`), security
  (`security-reviewer.md`).
- Consolidates findings by severity:
  - **Critical / Important** → dispatch a fix subagent, then re-run affected
    `verify-gates` and re-review the fix.
  - **Minor** → recorded in the report only.
- Reviewer prompts are project-aware: they read the project's `.claude/rules/*`
  (or equivalent) when present so, e.g., pawlife's security rules (no hardcoded
  JWT secret, BCrypt, OWASP Top 10) and performance/multi-instance rules
  (`@SchedulerLock`, no new instance-local state, N+1 avoidance,
  expand-contract migrations) are enforced.

## Config Schema — `.claude/verify.json`

```json
{
  "targets": [
    {
      "name": "backend",
      "dir": "backend",
      "lint": "./gradlew ktlintCheck",
      "build": "./gradlew build -x test",
      "test": "./gradlew test"
    }
  ]
}
```

- `targets` is an array so multi-module repos (e.g. pawlife: Kotlin backend +
  Android frontend) each get their own commands.
- `dir` scopes both the working directory for commands and the file-path prefix
  used to decide which targets a change touches.
- Each of `lint` / `build` / `test` is optional; a missing key means that gate is
  skipped for that target.

## File Structure (in superpowers-home)

```
skills/parallel-implementation/SKILL.md
                              /orchestrator-notes.md   (DAG construction + integration detail)
skills/verify-gates/SKILL.md
                   /verify-schema.md
skills/review-triad/SKILL.md
                   /perf-reviewer.md
                   /security-reviewer.md
skills/writing-plans/SKILL.md   (one-line edit: add execution option 3)
```

## Generality Decision

Skill bodies are generic and read configuration; nothing project-specific is
embedded. pawlife-specific behavior comes entirely from pawlife's
`.claude/verify.json` and its existing `.claude/rules/*.md`.

## Success Criteria

1. Given a plan with independent tasks, `parallel-implementation` runs them
   concurrently in separate worktrees and integrates them without losing work.
2. A lint/build/test failure in any task blocks completion of that task and is
   fixed (or escalated after the retry limit).
3. Branch-end review runs code, performance, and security reviewers in parallel;
   Critical/Important findings are fixed and re-verified automatically.
4. Running the same skills in a non-pawlife project works by supplying only a
   `.claude/verify.json` — no skill edits required.
5. The forked stock skills differ from upstream by only the single documented
   `writing-plans` line, keeping upstream merges clean.
