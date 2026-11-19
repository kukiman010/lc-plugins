[Русский](README.md) | **English**

# lc — task lifecycle plugin for Claude Code

Two commands that run a task through the same chain:
**plan → your "yes" → implement → review → fix**.
The main session in this chain is only a dispatcher: it never writes the code itself and
nothing gets edited until you approve the plan.

| Command | Use for |
|---|---|
| `/lc-implement <task>` | one clear task |
| `/lc-orchestrate <list or spec file>` | a list of tasks, or a big task that needs slicing |

## Install

```
/plugin marketplace add kukiman010/lc-plugins
/plugin install lc@lc-plugins
```

Update: `/plugin marketplace update lc-plugins`.

## Contents

```
.claude-plugin/
  plugin.json        — plugin manifest: name, version
  marketplace.json   — makes this repo a marketplace, so it installs from git
commands/
  lc-implement.md    — one-task command
  lc-orchestrate.md  — task-list command
agents/
  lc-planner.md      — planner
  lc-worker.md       — worker
  lc-reviewer.md     — reviewer
```

### Agents (subagents with their own context)

**lc-planner** (model: opus)
- Plans only: no implementation, no commits.
- Moves the previous plan in `.claude/tasks/` to `.claude/tasks/archive/<date-time>/` (never deletes it).
- Never guesses: every listed path is confirmed by search, new code cites an example
  `path:line` (Mirror), and gates use the project's real build, test or CI commands.
- If an ambiguity would change the outcome, it writes no tasks and returns `QUESTIONS`.
  It decides minor points itself and records them under `Assumptions`.
- Splits work into slices (small self-contained steps, usually up to ~5 files). A slice depends
  only on earlier slices, and the project still works after each one.
- Writes `.claude/tasks/plan.md` (checklist plus assumptions) and one `NNN-slug.md` per slice.

Task file fields:

| Field | Meaning |
|---|---|
| Goal | one-sentence goal |
| Depends on | earlier slices it needs |
| Files | paths (new ones marked `(new)`) |
| Mirror | existing code to copy from |
| Done when | observable result |
| Gate | real project command (build, test) the reviewer runs |
| Forbidden | what this slice must not touch |

**lc-worker** (model: inherits from the session)
- Implements exactly one slice and never opens the next one.
- Makes the smallest change, reuses what the project already has, and follows the project's `CLAUDE.md`.
- Runs the Gate itself and reports the files changed, the command and pass/fail.
- On a fix round it fixes only the must-fix items. Never commits.

**lc-reviewer** (model: inherits from the session)
- Cannot edit files: it has no Edit or Write tools.
- Reads the real `git diff` and `git status` and trusts the diff over the worker's summary.
- Re-runs the Gate itself.
- Checks that the goal is met, the Gate passes, project rules are followed and nothing outside the slice changed.
  Style, naming, performance and security are out of scope.
- The first line is exactly `VERDICT: ok` or `VERDICT: must-fix`; for must-fix a numbered list follows.

## How `/lc-implement` works

1. **Plan.** lc-planner writes one task file.
   - `QUESTIONS` — you get the questions, answer them, and the planner is relaunched.
   - `NEEDS ORCHESTRATE` — the task is too big; you see the proposed slices and a suggestion to use `/lc-orchestrate`.
   - If you pass an existing `.claude/tasks/NNN-*.md`, the planner is skipped.
2. **Your "yes".** You see the full task file. Nobody edits anything until you say yes. If you ask for changes, the plan is rewritten.
3. **Loop.** worker → reviewer → on `must-fix`, the worker again with that list → reviewer. At most 2 fix rounds.
4. **Report:** the task, the Gate command and its result, the verdict and the fix rounds used. No commit.

## How `/lc-orchestrate` works

1. **Plan.** lc-planner slices the request into `.claude/tasks/`. Questions are handled the same way as in `/lc-implement`.
2. **Your "yes".** You see the task list with dependencies and the assumptions.
3. **Each slice in order:** worker → reviewer → up to 2 fix rounds.
   - If a slice is still must-fix after 2 rounds, the whole run stops and later slices are not touched.
   - On ok, the slice is checked off in `plan.md` and a rollback point is saved with
     `git stash create` + `git stash store` (the working tree is not touched).
     Roll back: `git stash list` → `git stash apply stash@{N}`.
     New untracked files are not in that snapshot.
4. **Specialists (optional).** If ECC is installed, slices about auth, secrets or user input also get
   `ecc:security-reviewer`, and slices about hot paths get `ecc:performance-optimizer`.
   Critical/High findings count as must-fix and use the same 2-round budget.
5. **End:** what is done, where the run got blocked, the gates run and the checkpoints saved.
   No commit; `.claude/tasks/` is kept.

## Configuration

- **Fix rounds** — the `Max fix rounds` line in `commands/*.md`.
- **Planner model** — the `model:` field in `agents/lc-planner.md`.
- The commands explicitly forbid brainstorming, writing-plans, orch-* and prp-* skills,
  so other plugins (superpowers, ECC) don't take over the task with their own loop.

## Notes

- You commit the result yourself: the plugin never commits.
- If you used to keep these commands and agents in `~/.claude/commands` and `~/.claude/agents`,
  delete those copies after installing the plugin, or they will be duplicated.
