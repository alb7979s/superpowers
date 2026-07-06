# superpowers-home

A **personal fork of [obra/superpowers](https://github.com/obra/superpowers)** — the
skills-based software-development methodology for coding agents — with my own
workflow and domain skills layered on top.

It ships the full upstream skills library (brainstorming → planning →
subagent-driven implementation → review → finish) under the plugin name
`superpowers-home`, so it can be installed **alongside** the official
`superpowers` plugin without a name collision.

> Upstream project, docs, and credit: **[obra/superpowers](https://github.com/obra/superpowers)** (MIT, by Jesse Vincent / Prime Radiant).

## Install (Claude Code)

### From GitHub

```bash
/plugin marketplace add alb7979s/superpowers
/plugin install superpowers-home@superpowers-home-dev
```

### From a local checkout

```bash
git clone https://github.com/alb7979s/superpowers.git
```

```bash
/plugin marketplace add /absolute/path/to/superpowers
/plugin install superpowers-home@superpowers-home-dev
```

The plugin runs its own `SessionStart` hook, so the `using-superpowers` bootstrap
is active from the first message.

> If you also have the official `superpowers` plugin installed, both will load and
> some stock skills will appear twice (`superpowers:*` and `superpowers-home:*`).
> To avoid that, uninstall the official one via `/plugin`, or prune the duplicated
> stock skills from this fork.

## What's different from upstream

- **Renamed** to `superpowers-home` (plugin + marketplace) for side-by-side install.
- **Cross-skill references** rewritten to the `superpowers-home:` namespace so
  Skill invocations resolve within this plugin.
- **`skills/_template/`** — a starter skill folder to copy when adding new skills.
- **Personal skills (in design)** — see the spec below.

### Personal additions

| Skill | Status | Purpose |
|---|---|---|
| `parallel-implementation` | designed | Run independent plan tasks in parallel, each in its own git worktree, with sequential integration |
| `verify-gates` | designed | Config-driven `lint`/`build`/`test` gates at every task boundary, with a fix loop |
| `review-triad` | designed | Parallel code / performance / security review at branch end, with severity-based auto-fix |

Design spec:
[`docs/superpowers/specs/2026-07-06-parallel-implementation-design.md`](docs/superpowers/specs/2026-07-06-parallel-implementation-design.md).
These read a per-project `.claude/verify.json`, so they stay generic and reusable.

## Adding a skill

Copy `skills/_template/` to `skills/<your-skill>/`, rename it, and rewrite the
`SKILL.md` frontmatter `description` — that description is the trigger that tells
the agent when to use the skill. Use the upstream `writing-skills` / `skill-creator`
skills for format and testing guidance.

## Syncing upstream

```bash
git remote add upstream https://github.com/obra/superpowers.git   # once
git fetch upstream
git merge upstream/main
```

The fork deliberately keeps edits to stock skills minimal to keep these merges clean.

## The workflow (inherited)

1. **brainstorming** — refine the idea through questions, save a design spec.
2. **writing-plans** — break the spec into bite-sized TDD tasks.
3. **subagent-driven-development** / **executing-plans** / **parallel-implementation** — execute the plan.
4. **requesting-code-review** / **review-triad** — review the work.
5. **finishing-a-development-branch** — merge / PR / clean up.

## License

MIT — see [LICENSE](LICENSE). Forked from
[obra/superpowers](https://github.com/obra/superpowers).
