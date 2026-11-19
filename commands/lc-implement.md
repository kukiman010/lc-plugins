---
description: Task lifecycle for ONE task — lc-planner writes one task file, you approve, then lc-worker implements and lc-reviewer checks, up to 2 fix rounds.
argument-hint: <task text, task id, or path to a spec / .claude/tasks/NNN-*.md>
---

<!--
  КОМАНДА /lc-implement — цикл для ОДНОЙ задачи.
  Пример: /lc-implement убери заголовки страниц у плагинов
  Основная сессия здесь только «диспетчер»: сама задачу не делает и
  НИЧЕГО не правит до вашего «да» на план.
  Запрет на brainstorming/writing-plans и т.п. — чтобы superpowers и ECC
  не перехватили задачу своим циклом.
-->

This command IS the workflow. Do not invoke brainstorming, writing-plans,
subagent-driven-development, executing-plans, or any orch-* / prp-* skill.
You are the parent: dispatch only, do not do the task yourself. No file
edits by anyone until the user approves the plan in step 1.

Task: $ARGUMENTS

<!-- Сколько раундов исправлений разрешено. Менять число здесь. -->

Max fix rounds: 2.

## 1. Plan and approve

<!--
  Шаг 1. lc-planner пишет ОДИН файл задачи (Goal, Files, Mirror,
  Done when, Gate…), сверяясь с кодом.
  - Вернул QUESTIONS → показывает вам, ждёт ответы, перезапускает.
  - Решил, что задача на несколько срезов → останавливается и
    предлагает /lc-orchestrate.
  - Иначе показывает вам файл задачи целиком и ЖДЁТ «да».
    Правки → планировщик перезапускается с ними.
  Если передан уже готовый .claude/tasks/NNN-*.md — планировщик не
  нужен, сразу показ и ожидание «да».
-->

If the argument is an existing `.claude/tasks/NNN-*.md`, skip the planner
and use that file. Otherwise launch **lc-planner** with the task and this
instruction: "Single-task mode: write exactly one slice (001). If the work
clearly needs more than one slice, do not write files; return
`NEEDS ORCHESTRATE` with the proposed slice list."

- `QUESTIONS` → show them as-is, wait for answers, relaunch lc-planner
  with the task plus the answers.
- `NEEDS ORCHESTRATE` → show the proposed slices and suggest
  `/lc-orchestrate`. Stop.
- A task file → show its full contents to the user and ask whether to run
  it. Wait for an explicit yes. On changes, relaunch lc-planner with the
  feedback.

## 2. Loop

<!--
  Шаг 2 (только после «да»).
  worker делает → reviewer проверяет →
  если must-fix: worker исправляет → reviewer снова (не больше 2 раз) →
  стоп на ok или когда раунды кончились.
-->

1. Launch **lc-worker** with only the approved task file. Tell it not to
   take other items.
2. Launch **lc-reviewer** with the task file, the worker summary, and the
   paths it touched.
3. If `VERDICT: must-fix` and fewer than 2 fix rounds used: launch
   **lc-worker** with the task file plus the must-fix list, then
   **lc-reviewer** again. Repeat.
4. Stop when the verdict is ok, or after 2 fix rounds still must-fix.

## 3. Report

<!--
  Итоговый отчёт: задача, проверка и её результат, вердикт, сколько
  раундов ушло. Если так и не ok — что осталось и что задачу, скорее
  всего, надо переформулировать.
-->

Task, gate command and result, final verdict, fix rounds used. If still
must-fix, list the open items and say the task likely needs re-slicing.

Do not commit. Do not start a second task.
