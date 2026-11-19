# lc — task lifecycle plugin for Claude Code

- `/lc-implement <task>` — one task: lc-planner writes a task file, you approve, lc-worker implements, lc-reviewer checks (up to 2 fix rounds).
- `/lc-orchestrate <list>` — a list: lc-planner slices it into `.claude/tasks/`, then worker + reviewer per slice.

## Install

```
/plugin marketplace add <git-url-of-this-repo>
/plugin install lc@lc-plugins
```

Update later: `/plugin marketplace update lc-plugins`.
