---
plan_id: 001-tasks-in-postgres-visible-and-creatable
spec: 001-tasks-in-postgres-visible-and-creatable
feature: Tasks kept in Postgres, listed and created on the taskboard28 website
gitops_app: ika100-demo/taskboard28-gitops
status: draft
repos:
  - id: taskboard28-gitops
    shape: gitops-app
    summary: Postgres database for taskboard28, used by taskboard28-api; website wired to the API
    acs: [AC-001.1, AC-001.2, AC-001.8]
    order: before-merge
    gitops:
      - {resource: postgres}
      - {uses: postgres, component: taskboard28-api}
      - {env: {TASKBOARD28_API_URL: 'http://taskboard28-api'}, component: taskboard28-web}
    arguments: |
      dev only (spec non-goal: no staging/prod). One Postgres for the system, used only by taskboard28-api (A1);
      `uses` injects DATABASE_URL and PGHOST/PGPORT/PGDATABASE/PGUSER/PGPASSWORD into the API. taskboard28-api
      keeps its own exposure (A9); no replicas change (single instance, non-goal).
    depends_on: []
    done: false
  - id: taskboard28-api
    shape: service-python
    summary: Keep tasks in Postgres so they survive restarts
    acs: [AC-001.1, AC-001.2, AC-001.3, AC-001.4, AC-001.5, AC-001.6, AC-001.8]
    criteria:
      - "Given an empty Postgres database at DATABASE_URL, when the service starts, then `/ready` answers `200` and `POST /tasks` with `{\"title\": \"Buy milk\"}` answers `201`, without any manual schema step (system AC-001.1)"
      - "Given tasks created through `POST /tasks`, when the service is restarted or redeployed against the same database, then `GET /tasks` returns `200` with the same tasks (same `id`, `title`, `due_date` and `created_at`) in creation order (system AC-001.1) (replaces AC-001.7)"
      - "Given tasks \"A\" with `due_date` `2026-12-24` and then \"B\" without `due_date` created through `POST /tasks`, when a client sends `GET /tasks`, then the response is `200` with \"A\" (`due_date` `\"2026-12-24\"`) followed by \"B\" (`due_date` `null`), the last one being the task returned by the latest `201` (system AC-001.2, AC-001.4)"
      - "Given no task is stored in the database, when a client sends `GET /tasks`, then the response is `200` with an empty JSON array `[]` (system AC-001.3) (replaces AC-001.6)"
      - "Given a `POST /tasks` whose `title` is missing, not a string, empty or whitespace-only, or longer than 200 characters after trimming, when it is sent, then the response is `422` with a `detail` item whose `loc` ends in `title`, and no task is stored (system AC-001.5)"
      - "Given 1000 tasks are stored in the database (also after a restart), when a client sends a valid `POST /tasks`, then the response is `409` with `{\"detail\": \"task limit of 1000 reached\"}` and `GET /tasks` still returns exactly 1000 tasks (system AC-001.6)"
      - "Given the database is unreachable or does not answer within 5 seconds, when a client sends `GET /tasks` or `POST /tasks`, then the response is `503` with `{\"detail\": \"task store unavailable\"}`, no task is stored, `/ready` answers `503` and `/health` still answers `200` (system AC-001.8)"
      - "Given the database became reachable again after an outage, when a client sends `GET /tasks` without the service being restarted, then the response is `200` with all tasks stored before the outage and `/ready` answers `200` (system AC-001.8)"
    arguments: |
      Config only from env: DATABASE_URL (and PG*) injected by the gitops `uses: postgres`. The schema is created or
      migrated by the service itself at start. Endpoints, fields, validation and the 409 limit are unchanged from the
      repo's spec 001 (system A2); only storage changes. Tests run against a real Postgres (e.g. testcontainers).
    depends_on: []
    done: false
  - id: taskboard28-web
    shape: web-nextjs
    summary: Start page that lists tasks and creates them through taskboard28-api
    acs: [AC-001.1, AC-001.2, AC-001.3, AC-001.4, AC-001.5, AC-001.6, AC-001.7]
    criteria:
      - "Given `GET /tasks` on taskboard28-api returns \"A\" (`due_date` `2026-12-24`) followed by \"B\" (`due_date` `null`), when a person opens the start page, then it shows a task list with \"A\" followed by \"B\", each with its title and its due date as `2026-12-24` or the text \"No due date\" (system AC-001.1, AC-001.2)"
      - "Given `GET /tasks` returns `[]`, when a person opens the start page, then it shows the text \"No tasks yet\" instead of a list, together with the create form (system AC-001.3)"
      - "Given the start page, when a person enters the title \"Buy milk\", picks 2026-12-24 in the date input (or leaves it empty) and submits, then the site sends `POST /tasks` with `{\"title\": \"Buy milk\", \"due_date\": \"2026-12-24\"}` (or `\"due_date\": null`) and, on `201`, without leaving the page shows the task last in the list with that title and due date (or \"No due date\") and clears the form (system AC-001.4)"
      - "Given the create form, when a person submits an empty or whitespace-only title, or a title longer than 200 characters after trimming, then it shows \"Title is required\" or \"Title must be at most 200 characters\" next to the title field, keeps the entered values and sends no `POST /tasks` (system AC-001.5)"
      - "Given the API answers `POST /tasks` with `422` and a `detail` item whose `loc` ends in `title`, when the answer arrives, then the form shows that item's `msg` next to the title field and keeps the entered values (system AC-001.5)"
      - "Given the API answers `POST /tasks` with `409`, when the answer arrives, then the form shows \"The task limit of 1000 is reached\", keeps the entered values and the list is unchanged (system AC-001.6)"
      - "Given taskboard28-api is unreachable, does not answer within 10 seconds, or answers `GET /tasks` with anything but `200` and a JSON array, when a person opens the start page, then it shows \"Tasks could not be loaded. Try again later.\" instead of the list (system AC-001.7)"
      - "Given taskboard28-api is unreachable, does not answer within 10 seconds, or answers `POST /tasks` with any status other than `201`, `409` or `422` (such as `503`), when a person submits a valid task, then the form shows \"The task could not be saved. Try again.\" and keeps the entered values (system AC-001.7)"
    arguments: |
      The site calls the API from the server side only (Server Component / Server Action), base URL from the env var
      TASKBOARD28_API_URL (set by the gitops entry; no NEXT_PUBLIC_ variable). No sign-in (system A10).
    depends_on: []
    done: false
gitops_pin:
  - service: taskboard28-api
    overlay: dev
    apply_after: taskboard28-api
    note: the Postgres-backed API image; the database and DATABASE_URL are in place from the gitops entry, applied before merge
  - service: taskboard28-web
    overlay: dev
    apply_after: taskboard28-web
    note: the task page; it uses the unchanged API contract, so it needs no wait for the API pin
---

## taskboard28-gitops — Postgres database for taskboard28, used by taskboard28-api; website wired to the API

**Shape:** `gitops-app` · **Criteria:** AC-001.1, AC-001.2, AC-001.8 · **Depends on:** []

The system declares no resource yet. This entry adds the one Postgres database of taskboard28 (A1), lets only
taskboard28-api use it (`DATABASE_URL`, `PG*`), and gives taskboard28-web the in-cluster API address
`TASKBOARD28_API_URL=http://taskboard28-api` (service port 80). It owns the "survives a restart" storage part of
AC-001.1 and AC-001.8 and the website-to-API wiring of AC-001.2. It is applied before the components merge, so the
Postgres-backed API finds its database on its first deploy. Only dev changes: staging and prod are a non-goal.

## taskboard28-api — Keep tasks in Postgres so they survive restarts

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2, AC-001.3, AC-001.4, AC-001.5, AC-001.6, AC-001.8 · **Depends on:** []

The API replaces its in-memory store with Postgres. It owns persistence and creation order (AC-001.1, AC-001.2),
the empty list (AC-001.3), returning a created task (AC-001.4), the server side of title validation (AC-001.5),
the 1000-task limit now counted in the database (AC-001.6), and the `503` plus failing readiness on a database
outage (AC-001.8, A8). Its spec 001 criteria AC-001.7 ("empty after restart") and AC-001.6 ("empty since the
service started") are superseded by this slice; the rest of spec 001 stays as is (A2).

## taskboard28-web — Start page that lists tasks and creates them through taskboard28-api

**Shape:** `web-nextjs` · **Criteria:** AC-001.1, AC-001.2, AC-001.3, AC-001.4, AC-001.5, AC-001.6, AC-001.7 · **Depends on:** []

The website is a fresh template. Its start page gets the task list and the create form (A3, A4, A5), the
client-side title check with the AC-001.5 texts (A6), and the error texts of AC-001.6 and AC-001.7 (A7). It
talks to the API only (A1), so it needs no wait for the API: the contract below is the existing API contract.

Why this split: the database is infrastructure (gitops data), storage is the API's, presentation is the web's;
no library is needed because the only shared shape is the HTTP contract below.

## Contract

taskboard28-api, reached by the website at `TASKBOARD28_API_URL` (dev: `http://taskboard28-api`) and from outside
at its own dev URL (A9). JSON bodies, snake_case. No authentication (A10). Unchanged from taskboard28-api spec 001
except that tasks are stored in Postgres and `503` is new.

**Task object**

| field | type | notes |
|---|---|---|
| `id` | string | server-assigned UUID, opaque |
| `title` | string | trimmed, 1 to 200 characters |
| `due_date` | string or `null` | calendar date `YYYY-MM-DD` |
| `created_at` | string | ISO 8601 UTC timestamp, set by the server |

**`GET /tasks`**

- `200`: JSON array of all task objects in creation order, oldest first; `[]` when none exists.
- `503`: `{"detail": "task store unavailable"}` when the database is unreachable or slow.

**`POST /tasks`**

- Request: `{"title": "<string>", "due_date": "<YYYY-MM-DD>" | null}`; `due_date` may be omitted; unknown fields
  are ignored.
- `201`: the created task object.
- `422`: validation error (see Errors); no task created.
- `409`: `{"detail": "task limit of 1000 reached"}` when 1000 tasks are stored; no task created.
- `503`: `{"detail": "task store unavailable"}` when the database is unreachable or slow; no task created.

**Probes:** `GET /health` `200` while the process runs; `GET /ready` `200` only while the database answers,
else `503` (A8), so Kubernetes routes no traffic to the API during an outage.

### Rules

```yaml
rules:
  - id: title-required
    endpoint: POST /tasks
    field: title
    kind: required
    value: true
    provider: taskboard28-api
    consumers: [taskboard28-web]
  - id: title-min
    endpoint: POST /tasks
    field: title
    kind: min_length
    value: 1
    provider: taskboard28-api
    consumers: [taskboard28-web]
  - id: title-max
    endpoint: POST /tasks
    field: title
    kind: max_length
    value: 200
    provider: taskboard28-api
    consumers: [taskboard28-web]
```

Prose rules no tool checks:

- Lengths count the title after trimming leading and trailing whitespace; a whitespace-only title counts as empty.
  The API stores and returns the trimmed title; the website sends the trimmed title.
- `due_date` is `YYYY-MM-DD` or `null`; past dates are accepted. The website sends the value of the browser's date
  input as is, or `null` when it is empty; any other form answers `422` with `loc` ending in `due_date`.
- The website's own messages for a title it rejects before sending are "Title is required" (missing, empty,
  whitespace-only) and "Title must be at most 200 characters".

### Errors

```yaml
errors:
  status: 422
  body:
    list: detail
    field: loc
    rule: type
    message: msg
  unlisted:
    message: "The task could not be saved. Try again."
```

The `422` body is FastAPI's standard one: `{"detail": [{"loc": ["body", "title"], "type": "...", "msg": "..."}]}`.
The website takes the field from the last element of `loc` (a plain string `loc` is the field itself) and shows
that item's `msg` next to that field, keeping the entered values.

Other listed answers on `POST /tasks`: `409` shows "The task limit of 1000 is reached"; `503` shows the unlisted
message. Any answer the contract does not list (another status, a body that is not JSON, a network error or a
timeout) shows "The task could not be saved. Try again." on the form and keeps the entered values. On `GET /tasks`,
anything but `200` with a JSON array shows "Tasks could not be loaded. Try again later." instead of the list.

### Timeouts

- **taskboard28-web -> taskboard28-api** `GET /tasks` and `POST /tasks`: 10 seconds each, no retry (a retried
  `POST` could create a task twice). A timeout or a refused connection is treated as an unlisted answer (texts
  above). The page itself still renders, with the create form.
- **taskboard28-api -> Postgres**: connect timeout 3 seconds, statement timeout 5 seconds; either answers the
  request with `503 {"detail": "task store unavailable"}` and writes nothing. The API does not crash or exit while
  the database is down; it reconnects on the next request once the database is back.
- **`/ready`**: checks the database with a 2-second timeout, so a readiness probe never hangs; `/health` never
  touches the database.

## gitops-app PR

The entry PR of this repo (resource postgres, `uses` for taskboard28-api, `TASKBOARD28_API_URL` for taskboard28-web)
lands before the components merge. Then each component is pinned in dev after its own PR merges: taskboard28-api
(the Postgres-backed image) and taskboard28-web (the task page). No staging or prod pins (non-goal).

After Argo reconciles dev, verify: the Postgres cluster of taskboard28 is healthy; the taskboard28-api pod has
`DATABASE_URL` and is Ready; `POST /tasks` then a restart of the API pod and `GET /tasks` returns the task
(AC-001.1); the website's start page lists tasks and creates one (AC-001.2, AC-001.4); scaling the database to
zero makes `GET /tasks` answer `503` and the API's readiness fail, and scaling it back returns all tasks (AC-001.8).
