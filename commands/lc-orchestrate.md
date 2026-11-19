---
description: Task lifecycle for a LIST — lc-planner slices it, you approve, then lc-worker + lc-reviewer per slice with up to 2 fix rounds.
argument-hint: <list of tasks of any kind>
---

<!--
  КОМАНДА /lc-orchestrate — цикл для СПИСКА задач или большой задачи.
  Можно передать текст списком или путь к файлу со спецификацией
  (например, результат brainstorming).
  Основная сессия — «диспетчер»: сама срезы не делает и не отдаёт
  worker'у весь план целиком.
-->

This command IS the workflow. Do not invoke brainstorming, writing-plans,
subagent-driven-development, executing-plans, or any orch-* / prp-* skill.
You are the parent: run the loop, never implement a slice yourself, never
hand lc-worker the whole plan.

Request: $ARGUMENTS

<!-- Сколько раундов исправлений на каждый срез. Менять число здесь. -->

Max fix rounds per slice: 2.

## 1. Plan

<!--
  Шаг 1. lc-planner пишет задачи в .claude/tasks/.
  Если он вернул QUESTIONS — показывает вам вопросы, ждёт ответов и
  перезапускает планировщик с запросом + ответами.
  Затем ПАУЗА: показывает план (задачи, зависимости, допущения) и ждёт «да».
  Если есть правки — планировщик перезапускается с ними.
-->

Launch **lc-planner** with the request as given.

- If it returns `QUESTIONS`: show them to the user as-is, wait for answers,
  then relaunch lc-planner with the original request plus the answers.
- When it returns a plan: show the user the task list (with dependencies)
  and the Assumptions from `.claude/tasks/plan.md`, and ask whether to run
  it. Wait for a yes. On changes, relaunch lc-planner with the feedback.

## 2. Each slice, in order

<!--
  Шаг 2. По каждой задаче по порядку:
  worker → reviewer → (до 2 раундов исправлений).
  Если после 2 раундов всё ещё must-fix — весь запуск останавливается,
  следующие задачи не трогаются.
  При ok: галочка в plan.md и точка отката через git stash
  (рабочую папку не меняет; новые неотслеживаемые файлы в снимок
  не попадают). Откат: git stash list → git stash apply stash@{N}.
-->

For each `.claude/tasks/NNN-*.md` linked from `plan.md`:

1. Launch **lc-worker** with only that file.
2. Launch **lc-reviewer** on that slice.
3. If `VERDICT: must-fix` and fewer than 2 fix rounds used: launch
   **lc-worker** with the file plus the must-fix list, then
   **lc-reviewer** again. Repeat.
4. Still must-fix after 2 rounds: stop the whole run, report the blocking
   file and open items. Do not skip ahead.
5. On ok:
   - check its box in `plan.md`;
   - save a checkpoint without touching the working tree:
     `h=$(git stash create) && [ -n "$h" ] && git stash store -m "lc: NNN-slug ok" "$h"`
     (shortcut: untracked files are not in the snapshot; the user can
     `git stash list` / `git stash apply` to roll back).

## 3. Optional specialists

<!--
  Шаг 3 (не всегда). Если срез про авторизацию/секреты/ввод пользователя —
  дополнительно ecc:security-reviewer; если про скорость — 
  ecc:performance-optimizer. Серьёзные находки считаются must-fix и
  тратят те же 2 раунда. Если ECC не установлен — шаг пропускается.
-->

After reviewer ok, only if the slice matches and the agent is available:
auth / secrets / user input → **ecc:security-reviewer**; hot-path cost →
**ecc:performance-optimizer**. Not every slice. Critical/High findings
count as must-fix and use the same fix-round budget.

## 4. End

<!-- Итог: что сделано, где застряло, какие проверки и точки отката. Без коммита. -->

Report: done, blocked, gates run, checkpoints saved. Do not commit. Do not
delete `.claude/tasks/`.
