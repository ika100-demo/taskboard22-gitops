---
spec_id: 001-tasks-in-postgres-visible-and-creatable
title: Tasks in Postgres, visible and creatable in the web UI
status: approved
priority: P1
shape: gitops-app
---

# 001 — Tasks in Postgres, visible and creatable in the web UI

## Problem

The Task API keeps tasks in memory: every restart or redeploy loses them, and each running instance has its own list. That makes the taskboard unusable for real work. The website is still the starter page, so people can reach tasks only by calling the API directly. Keeping tasks in a Postgres database and giving people a page to see and create them turns taskboard22 into a first usable product.

## Stories

As a team member, I want the tasks I record to still be there after the system restarts or is redeployed, so that I can rely on the taskboard for real work.
**Repos:** taskboard22-api, taskboard22-gitops

As a team member, I want to open the website and see all tasks with their due dates, so that I know what has to be done.
**Repos:** taskboard22-web, taskboard22-api, taskboard22-gitops

As a team member, I want to add a task with a title and an optional due date from the website, so that I do not need an API client to record work.
**Repos:** taskboard22-web, taskboard22-api

## Acceptance criteria

- **AC-001.1** Given tasks were created through the Task API, when every instance of the Task API is restarted or redeployed, then `GET /tasks` afterwards returns `200` with each of those tasks once, with the same `id`, `title` and `due_date` as returned on creation, oldest first.
- **AC-001.2** Given the Task API runs with more than one instance, when a task is created through any instance, then `GET /tasks` answered by any other instance lists that task.
- **AC-001.3** Given the Task API cannot reach its database, when a client sends `GET /tasks` or `POST /tasks`, then the response is `503` with a JSON body holding a `detail` message, no task is created, and the readiness endpoint `/ready` answers `503` until the database is reachable again; once it is, the tasks created before the outage are listed again.
- **AC-001.4** Given tasks exist, when a person opens the website's start page, then it shows every task once, oldest first, each with its title and its due date as `YYYY-MM-DD`, or "No due date" when it has none.
- **AC-001.5** Given no task exists, when a person opens the website's start page, then it shows "No tasks yet" and the form to add a task.
- **AC-001.6** Given the start page is open, when a person enters the title "Buy milk", optionally picks a due date, and submits the form, then within 2 seconds the task appears at the end of the list without a manual page reload, the form fields are emptied, and reloading the page still shows the task.
- **AC-001.7** Given the start page is open, when a person submits the form with a title that is empty, whitespace only, or longer than 200 characters, then the page shows the message "Enter a title of 1 to 200 characters" next to the title field, keeps the entered values, and no task is created.
- **AC-001.8** Given the Task API is unreachable or answers with an error, when a person opens the start page, then the page shows "Tasks could not be loaded. Try again later." instead of the list; and when a person submits the form in that state, then the page shows "The task could not be saved. Try again." and keeps the entered title and due date.

## Non-goals

- Login, users and per-user task lists: in dev, anyone who can reach the website or the Task API can see and create every task; a login spec comes before promotion to staging or prod.
- Editing, completing, deleting or reading a single task; filtering, search and pagination.
- Database backups and restore (a later spec can turn on `backup:`).
- Promotion to staging or prod; this spec delivers to dev, where components start.
- Keeping tasks created in memory before the switch.
- Live updates of the list when another person adds a task (a page reload shows them).

## Assumptions

- **A1** The Task API's contract stays as it is: `POST /tasks` and `GET /tasks`, fields `id`, `title`, `due_date`, the same validation rules (title 1 to 200 characters, not whitespace only; due date `YYYY-MM-DD`, optional) and `422` for invalid input (other option: change the contract with the move; affects AC-001.1, AC-001.6, AC-001.7).
- **A2** Tasks held in memory at the moment of the switch are not carried over; the list starts empty in the database, as the in-memory list already did after every restart (other option: export and import them; affects AC-001.1).
- **A3** A database outage answers `503` with a `detail` message and marks the API not ready, instead of failing at start or answering `500` (other option: `500`; affects AC-001.3).
- **A4** The task list and the form are on the website's start page `/`, replacing the starter content (other option: a separate `/tasks` page; affects AC-001.4 to AC-001.8).
- **A5** The website checks the title before sending it with the same rules as the API and shows "Enter a title of 1 to 200 characters"; the due date is picked with a date input, so an invalid date cannot be entered (other option: rely on the API's `422` only; affects AC-001.7).
- **A6** The list on the website shows all tasks oldest first, like the API, with no limit (other option: newest first, or a cap; affects AC-001.4, AC-001.6).
- **A7** Messages on the website are in English with the wording in the criteria (other option: other wording or language; affects AC-001.5, AC-001.7, AC-001.8).
- **A8** A newly created task shows in the list within 2 seconds of submitting (other option: another limit; affects AC-001.6).
- **A9** The database is one in-cluster Postgres declared for the system (the platform's own database resource, no paid service), used only by the Task API (other option: an external managed database; affects AC-001.1, AC-001.2).

## Notes for the architect

- The database is declared with `/resource:add postgres --type database` and the Task API gets `uses: [postgres]`, which injects `DATABASE_URL`; the website needs the Task API's address wired as an environment variable in `system.yaml`, not hard-coded.
- The Task API's done component spec `001-create-and-list-tasks-in-memory` has AC-001.6 ("Given the service started and no task was created since ... empty array") and the non-goal "Durable storage"; the API's component slice of this spec should supersede AC-001.6 there, because after this spec a restart no longer empties the list.

## Open questions

- ~~Anyone who can reach the website (and the Task API) can see and create every task, with no login; is that acceptable while the system runs in dev only? (suggested: yes, no login in dev; a login spec comes before promotion to staging or prod; affects AC-001.4, AC-001.6, new criterion if no)~~ Answered 2026-11-20: yes, no login in dev; a login spec comes before promotion to staging or prod (folded into Non-goals).

<!-- Priority levels (spec 196) for the product-manager's PRIORITY line: P0 an outage, data loss, a security problem, or something that blocks all other work; P1 a normal bug or feature; the fallback when nothing signals urgency; P2 cosmetic, a convenience, or something with an easy workaround. A reason is "P<n>: <phrase>", one of: P0: outage, data loss, security problem, blocks all other work; P1: normal bug or feature; P2: cosmetic, convenience, easy workaround. -->

## Changelog

- 2026-11-20 created
- 2026-11-20 priority P1 proposed: P1: normal bug or feature
- 2026-11-20 amended: access question answered (no login in dev; login spec before staging or prod), folded into Non-goals
- 2026-11-20 approved
