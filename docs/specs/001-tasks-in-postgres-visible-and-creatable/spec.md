---
spec_id: 001-tasks-in-postgres-visible-and-creatable
title: Tasks in Postgres, visible and creatable in the web UI
status: approved
priority: P1
shape: gitops-app
---

# 001 — Tasks in Postgres, visible and creatable in the web UI

## Problem

The Task API keeps tasks in memory: every restart or redeploy of taskboard28-api loses all tasks, and with only one instance at most 1000 tasks fit. The website taskboard28-web shows nothing yet, so a person can only reach tasks by calling the API by hand. Keeping tasks in a Postgres database and giving people a page to see and create them turns taskboard28 into something people can actually use.

## Stories

As a taskboard28 user, I want tasks I created to still be there after the system restarts or is redeployed, so that I can rely on the board.
**Repos:** taskboard28-api, taskboard28-gitops

As a taskboard28 user, I want to open the website and see all tasks with their due dates, so that I know what needs doing without calling the API.
**Repos:** taskboard28-web, taskboard28-api, taskboard28-gitops

As a taskboard28 user, I want to create a task with a title and an optional due date on the website, so that I can record work without calling the API.
**Repos:** taskboard28-web, taskboard28-api

## Acceptance criteria

- **AC-001.1** Given tasks created through `POST /tasks` or the website, when taskboard28-api is restarted or redeployed, then `GET /tasks` returns `200` with the same tasks (same `id`, `title`, `due_date` and `created_at`) in creation order, and the website lists them too.
- **AC-001.2** Given tasks "A" (due `2026-12-24`) and "B" (no due date) created in that order, through the API or the website, when a person opens the website's start page, then it shows a task list with "A" followed by "B", each showing its title, and its due date as `2026-12-24` or the text "No due date".
- **AC-001.3** Given no task exists, when a person opens the website's start page, then it shows the text "No tasks yet" instead of a list, together with the create form.
- **AC-001.4** Given the website's start page, when a person enters the title "Buy milk", picks the due date 2026-12-24 (or leaves it empty) and submits the form, then without leaving the page the new task appears last in the list with that title and due date (or "No due date"), the form is cleared, and `GET /tasks` on the API returns the task.
- **AC-001.5** Given the website's create form, when a person submits it with an empty or whitespace-only title, or a title longer than 200 characters, then the form shows "Title is required" or "Title must be at most 200 characters" next to the title field, keeps the entered values, and no task is created.
- **AC-001.6** Given 1000 tasks already exist, when a person submits a valid task on the website, then the form shows "The task limit of 1000 is reached", keeps the entered values, and the list still shows exactly 1000 tasks.
- **AC-001.7** Given the website cannot reach taskboard28-api (or the API answers with an error), when a person opens the start page, then it shows "Tasks could not be loaded. Try again later." instead of the list; and when a person submits the create form, then it shows "The task could not be saved. Try again." and keeps the entered values.
- **AC-001.8** Given the database is unreachable, when a client sends `GET /tasks` or `POST /tasks`, then the response is `503` with an error saying the task store is unavailable, no task is created, and once the database is reachable again all tasks stored before are returned.

## Non-goals

- Editing, completing, deleting or reading a single task, in the API or on the website.
- Filtering, sorting options, search or pagination of the list.
- Users, sign-in, ownership or any access control (as in taskboard28-api spec 001, A10): a follow-up spec.
- Carrying over tasks held in memory by the running API version; they are lost on any restart today anyway (taskboard28-api AC-001.7).
- Database backups and restore, and running the system in staging or prod: the system stays in dev, as today, until promoted.
- Running several API instances at once.

## Assumptions

- **A1** The database is one Postgres database for the system, declared as a resource of taskboard28 and used only by taskboard28-api; the website reaches tasks only through the API (other option: the website reads the database directly; affects AC-001.1, AC-001.2, AC-001.8).
- **A2** The API keeps its contract from taskboard28-api spec 001 unchanged (`POST /tasks`, `GET /tasks`, fields, validation, `422` errors, `409` at 1000 tasks); only "tasks are lost on restart" changes (other option: lift or raise the 1000-task limit now that memory is no longer the bound; affects AC-001.1, AC-001.6).
- **A3** The website's start page holds both the task list and the create form; there is no separate page per task (other option: a separate "New task" page; affects AC-001.2 to AC-001.7).
- **A4** The website lists tasks in creation order, oldest first, the same order as `GET /tasks` (other option: newest first, or by due date; affects AC-001.2, AC-001.4).
- **A5** Due dates are shown as `YYYY-MM-DD` and picked with the browser's date input; a task without one shows "No due date" (other option: a localized format such as "24 Dec 2026"; affects AC-001.2, AC-001.4).
- **A6** The website checks the title (trimmed, 1 to 200 characters) before sending, with the messages of AC-001.5, and also shows the API's `422` answer the same way when it arrives (other option: rely on the API's answer only; affects AC-001.5).
- **A7** The error texts are the English ones quoted in AC-001.3, AC-001.5, AC-001.6 and AC-001.7 (other option: other wording; affects those criteria).
- **A8** A database outage answers `503` and makes the API's readiness check fail, so that it receives no traffic until the database is back (other option: `500`, readiness unaffected; affects AC-001.8).
- **A9** taskboard28-api stays reachable from outside through its own URL in dev, besides the website (other option: reachable only from inside the system, through the website; affects AC-001.1, AC-001.4).
- **A10** Like the API (taskboard28-api spec 001, A10, decided by the user), the website needs no sign-in: anyone who reaches its URL may see and create tasks (other option: access control in this spec; affects AC-001.2 to AC-001.7).

## Notes for the architect

- This spec changes taskboard28-api's active AC-001.7 ("tasks are empty after a restart"): the api slice derived from the plan must supersede it there (`replaces` in that repo's numbering), because system and component ids share the number 001.

## Open questions

<!-- Priority levels (spec 196) for the product-manager's PRIORITY line: P0 an outage, data loss, a security problem, or something that blocks all other work; P1 a normal bug or feature; the fallback when nothing signals urgency; P2 cosmetic, a convenience, or something with an easy workaround. A reason is "P<n>: <phrase>", one of: P0: outage, data loss, security problem, blocks all other work; P1: normal bug or feature; P2: cosmetic, convenience, easy workaround. -->

## Changelog

- 2026-11-23 created
- 2026-11-23 priority P1 proposed: P1: normal bug or feature
- 2026-11-23 approved
