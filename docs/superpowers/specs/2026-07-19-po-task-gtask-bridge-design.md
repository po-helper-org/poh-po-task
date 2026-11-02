# poh-po-task — GitHub Issue → Google Task bridge

**Date:** 2026-07-19
**Status:** Approved design
**Source issue:** https://github.com/po-helper-org/poh-po-task/issues/1

## 1. Business goal

Product tasks live as GitHub Issues (some created automatically by PO-helper). Daily
planning is done by [reclaim.ai](https://reclaim.ai), which integrates for free only with
**Google Tasks**. This service bridges the gap: selected Issues become Google Tasks so
reclaim can see and schedule them.

Single user, single Google account.

## 2. Scope

**In scope**
- Create a Google Task when an Issue gets the `po-task` label.
- Complete the Google Task when the Issue is closed or the label is removed.

**Out of scope**
- Updating title/notes/due when the Issue is edited (full two-way sync).
- Comments, attachments, checklists.
- Reverse direction (Google Task → Issue).
- Multi-user.

## 3. Architecture

One container. Reverse-proxy (external, already exists) terminates TLS and forwards to the
container's internal port. Fully independent of poh-scheduller (calendar sync): own repo,
own `docker-compose.yml`, own `.env`, own volume, own Google token.

```
GitHub  --POST /webhook-->  webhook (FastAPI)
                                 |  verify HMAC, parse event
                                 v
                              bridge  --create/complete-->  gtasks (Google Tasks API)
                                 |  idempotency
                                 v
                              state (SQLite, volume)
```

### Modules

| Module | Responsibility | Depends on |
|--------|----------------|-----------|
| `app/webhook.py` | FastAPI app. `POST /webhook`: verify `X-Hub-Signature-256`, parse GitHub event, route to bridge. `GET /health`: liveness. | `bridge`, `config` |
| `app/bridge.py` | Decide action from event (create / complete / ignore). Enforce idempotency via `state`. Map Issue fields → task fields. | `gtasks`, `state`, `github` |
| `app/gtasks.py` | Google Tasks client: resolve/create task list by name, create task, patch task `status=completed`. Handles token load + refresh. | `google-api-python-client`, `google-auth` |
| `app/github.py` | Fetch milestone `due_on` for an Issue (event payload usually carries it; fetch as fallback). Read-only PAT. | `requests` |
| `app/state.py` | SQLite store. Table `task_map`. Idempotency + issue→task lookup. | stdlib `sqlite3` |
| `app/config.py` | Load `.env` + `secrets/` files into a typed config object. | `pydantic-settings` |
| `google_token_bootstrap.py` | One-off OAuth installed-app flow, scope `tasks`. Writes `state/google_tasks.token`. | `google-auth-oauthlib` |

## 4. Event handling

Only `issues` webhook events are processed. Action routing:

| GitHub action | Condition | Behavior |
|---------------|-----------|----------|
| `labeled` | label == `PO_LABEL` | If issue not in state → create task, store mapping. Else no-op. |
| `unlabeled` | label == `PO_LABEL` | If issue in state → complete task. |
| `closed` | issue in state | Complete task. |
| any other | — | 200 OK, ignore. |

**Idempotency:** state key = `f"{repo}#{issue_number}"`. Create writes the row only after the
Google API call succeeds. GitHub redeliveries and duplicate labels are no-ops.

## 5. Field mapping (FR-2)

| Google Task field | Source |
|-------------------|--------|
| `title` | Issue `title` |
| `notes` | Issue `body` + `\n\n` + Issue `html_url` |
| `due` | Milestone `due_on` (RFC3339, date-only per Tasks API) if the issue has a milestone with a due date; else omitted |

Google Tasks `due` stores date only (time ignored by the API).

## 6. State schema

```sql
CREATE TABLE IF NOT EXISTS task_map (
    issue_key   TEXT PRIMARY KEY,   -- "owner/repo#123"
    task_id     TEXT NOT NULL,
    list_id     TEXT NOT NULL,
    status      TEXT NOT NULL DEFAULT 'open',  -- open | completed
    created_at  TEXT NOT NULL,
    updated_at  TEXT NOT NULL
);
```

DB file lives in the mounted volume (`state/bridge.db`).

## 7. Configuration

`.env`:
```
GITHUB_WEBHOOK_SECRET=...     # HMAC verification (required)
GITHUB_TOKEN=...              # read-only PAT for milestone fallback
GITHUB_REPO=<owner>/<private-workspace>
PO_LABEL=po-task
GOOGLE_TASKLIST=PO            # list name; created if absent
PORT=8080
```

`secrets/` (files, gitignored, mounted read-only):
```
secrets/google_client_id
secrets/google_client_secret
```

`state/` (volume): `google_tasks.token`, `bridge.db`.

## 8. Security

- `X-Hub-Signature-256` HMAC-SHA256 verified against `GITHUB_WEBHOOK_SECRET`; mismatch → 401.
  Use `hmac.compare_digest`.
- GitHub PAT is read-only.
- Google token scope limited to `https://www.googleapis.com/auth/tasks`. Stored in volume,
  separate from the calendar token.
- Secrets never baked into the image; mounted at runtime.

## 9. Error handling

- Google API failure during create/complete → return 500 so GitHub retries. State is written
  only after success, so a retry re-attempts cleanly (no partial row).
- Unknown / non-`issues` events → 200 OK, ignored (avoid GitHub marking the hook failing).
- Missing mapping on `closed`/`unlabeled` → 200 OK, no-op.

## 10. Deployment

1. Fill `.env` and `secrets/`.
2. Run `python google_token_bootstrap.py` once on a machine with a browser → mints
   `state/google_tasks.token`.
3. `docker compose up -d`.
4. Register the GitHub webhook (URL from reverse-proxy, content-type `application/json`,
   secret = `GITHUB_WEBHOOK_SECRET`, events = Issues) on `<private-workspace>` (the private PO workspace repo).

## 11. Testing

Unit tests, no network (GitHub + Google mocked):
- HMAC verify: valid passes, tampered/absent → 401.
- Event routing: labeled→create, unlabeled/closed→complete, other→ignore.
- Field mapping incl. milestone-due present/absent.
- Idempotency: duplicate labeled → single create; redelivery → no-op.
- State store CRUD.

## 12. Tech stack

Python 3.12, FastAPI + uvicorn, `google-api-python-client` + `google-auth(-oauthlib)`,
`requests`, `pydantic-settings`, stdlib `sqlite3`, `pytest`. Containerized.
