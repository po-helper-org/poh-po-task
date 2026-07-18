# poh-po-task — GitHub Issue → Google Task bridge — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A standalone container that turns a GitHub Issue labeled `po-task` into a Google Task (and completes it when the Issue closes or the label is removed), so reclaim.ai can schedule it.

**Architecture:** FastAPI webhook receiver verifies GitHub's HMAC signature, routes `issues` events to a bridge module. The bridge enforces idempotency via a SQLite store and calls a Google Tasks client to create/complete tasks. Fully independent of poh-scheduller — own repo, own `.env`, own volume, own Google token.

**Tech Stack:** Python 3.12, FastAPI + uvicorn, google-api-python-client + google-auth(-oauthlib), pydantic-settings, stdlib sqlite3, pytest.

## Global Constraints

- Python 3.12.
- Google OAuth scope: exactly `https://www.googleapis.com/auth/tasks`.
- HMAC compare uses `hmac.compare_digest` (constant-time). Never `==`.
- Secrets (`.env`, `secrets/`, `state/`) are gitignored — never committed.
- State row is written only AFTER the Google API call succeeds (retry-safe).
- Idempotency key: `f"{repo}#{issue_number}"`, e.g. `kibarik/mts-po-workspace#42`.
- All code and comments in English.

---

## File Structure

```
poh-po-task/
├── app/
│   ├── __init__.py
│   ├── config.py        # env + secrets → Config object
│   ├── state.py         # SQLite StateStore
│   ├── gtasks.py        # Google Tasks client
│   ├── github.py        # parse milestone due from payload
│   ├── bridge.py        # event routing + idempotency + mapping
│   └── webhook.py       # FastAPI app, HMAC verify
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_config.py
│   ├── test_state.py
│   ├── test_gtasks.py
│   ├── test_github.py
│   ├── test_bridge.py
│   └── test_webhook.py
├── google_token_bootstrap.py
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
├── .env.example
└── README.md
```

---

## Task 1: Scaffolding + config

**Files:**
- Create: `requirements.txt`, `app/__init__.py`, `tests/__init__.py`, `app/config.py`, `tests/test_config.py`, `.env.example`
- Test: `tests/test_config.py`

**Interfaces:**
- Produces: `Config` (pydantic-settings). Fields: `github_webhook_secret: str`, `github_token: str = ""`, `github_repo: str`, `po_label: str = "po-task"`, `google_tasklist: str = "PO"`, `port: int = 8080`, `state_dir: Path = Path("state")`, `secrets_dir: Path = Path("secrets")`. Properties: `google_client_id -> str`, `google_client_secret -> str`, `token_path -> Path`, `db_path -> Path`. Constructor loads `.env`.

- [ ] **Step 1: Write requirements.txt**

```
fastapi==0.115.*
uvicorn[standard]==0.32.*
pydantic-settings==2.*
google-api-python-client==2.*
google-auth==2.*
google-auth-oauthlib==1.*
requests==2.*
pytest==8.*
httpx==0.27.*
```

- [ ] **Step 2: Create empty package markers**

`app/__init__.py` and `tests/__init__.py` — empty files.

- [ ] **Step 3: Write the failing test**

`tests/test_config.py`:
```python
from pathlib import Path
from app.config import Config


def _write_secrets(tmp_path: Path):
    sec = tmp_path / "secrets"
    sec.mkdir()
    (sec / "google_client_id").write_text("cid-123\n")
    (sec / "google_client_secret").write_text("csec-456\n")
    return sec


def test_config_loads_fields_and_secret_properties(tmp_path):
    sec = _write_secrets(tmp_path)
    cfg = Config(
        github_webhook_secret="whsec",
        github_repo="kibarik/mts-po-workspace",
        state_dir=tmp_path / "state",
        secrets_dir=sec,
    )
    assert cfg.po_label == "po-task"          # default
    assert cfg.google_tasklist == "PO"        # default
    assert cfg.github_repo == "kibarik/mts-po-workspace"
    assert cfg.google_client_id == "cid-123"  # stripped
    assert cfg.google_client_secret == "csec-456"
    assert cfg.token_path == (tmp_path / "state") / "google_tasks.token"
    assert cfg.db_path == (tmp_path / "state") / "bridge.db"
```

- [ ] **Step 4: Run test to verify it fails**

Run: `pytest tests/test_config.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.config'`

- [ ] **Step 5: Write app/config.py**

```python
from pathlib import Path
from pydantic_settings import BaseSettings, SettingsConfigDict


class Config(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    github_webhook_secret: str
    github_token: str = ""
    github_repo: str
    po_label: str = "po-task"
    google_tasklist: str = "PO"
    port: int = 8080
    state_dir: Path = Path("state")
    secrets_dir: Path = Path("secrets")

    @property
    def google_client_id(self) -> str:
        return (self.secrets_dir / "google_client_id").read_text().strip()

    @property
    def google_client_secret(self) -> str:
        return (self.secrets_dir / "google_client_secret").read_text().strip()

    @property
    def token_path(self) -> Path:
        return self.state_dir / "google_tasks.token"

    @property
    def db_path(self) -> Path:
        return self.state_dir / "bridge.db"
```

- [ ] **Step 6: Run test to verify it passes**

Run: `pytest tests/test_config.py -v`
Expected: PASS

- [ ] **Step 7: Write .env.example**

```
GITHUB_WEBHOOK_SECRET=change-me
GITHUB_TOKEN=
GITHUB_REPO=kibarik/mts-po-workspace
PO_LABEL=po-task
GOOGLE_TASKLIST=PO
PORT=8080
```

- [ ] **Step 8: Commit**

```bash
git add requirements.txt app/__init__.py tests/__init__.py app/config.py tests/test_config.py .env.example
git commit -m "feat: project scaffolding and config module"
```

---

## Task 2: SQLite state store

**Files:**
- Create: `app/state.py`, `tests/test_state.py`
- Test: `tests/test_state.py`

**Interfaces:**
- Consumes: nothing.
- Produces: `StateStore(db_path: Path)`. Methods: `get(issue_key: str) -> dict | None` (keys: `issue_key`, `task_id`, `list_id`, `status`); `add(issue_key: str, task_id: str, list_id: str) -> None`; `mark_completed(issue_key: str) -> None`. Constructor creates the table if absent.

- [ ] **Step 1: Write the failing test**

`tests/test_state.py`:
```python
from app.state import StateStore


def test_add_get_and_complete(tmp_path):
    store = StateStore(tmp_path / "bridge.db")
    assert store.get("r#1") is None

    store.add("r#1", task_id="t1", list_id="L1")
    row = store.get("r#1")
    assert row["task_id"] == "t1"
    assert row["list_id"] == "L1"
    assert row["status"] == "open"

    store.mark_completed("r#1")
    assert store.get("r#1")["status"] == "completed"


def test_add_is_idempotent_on_duplicate_key(tmp_path):
    store = StateStore(tmp_path / "bridge.db")
    store.add("r#1", task_id="t1", list_id="L1")
    store.add("r#1", task_id="t2", list_id="L2")  # must not raise or duplicate
    assert store.get("r#1")["task_id"] == "t1"     # first write wins
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_state.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.state'`

- [ ] **Step 3: Write app/state.py**

```python
import sqlite3
from datetime import datetime, timezone
from pathlib import Path

_SCHEMA = """
CREATE TABLE IF NOT EXISTS task_map (
    issue_key   TEXT PRIMARY KEY,
    task_id     TEXT NOT NULL,
    list_id     TEXT NOT NULL,
    status      TEXT NOT NULL DEFAULT 'open',
    created_at  TEXT NOT NULL,
    updated_at  TEXT NOT NULL
);
"""


def _now() -> str:
    return datetime.now(timezone.utc).isoformat()


class StateStore:
    def __init__(self, db_path: Path):
        db_path = Path(db_path)
        db_path.parent.mkdir(parents=True, exist_ok=True)
        self._db = str(db_path)
        with self._conn() as c:
            c.executescript(_SCHEMA)

    def _conn(self) -> sqlite3.Connection:
        conn = sqlite3.connect(self._db)
        conn.row_factory = sqlite3.Row
        return conn

    def get(self, issue_key: str) -> dict | None:
        with self._conn() as c:
            r = c.execute(
                "SELECT * FROM task_map WHERE issue_key = ?", (issue_key,)
            ).fetchone()
        return dict(r) if r else None

    def add(self, issue_key: str, task_id: str, list_id: str) -> None:
        now = _now()
        with self._conn() as c:
            c.execute(
                "INSERT OR IGNORE INTO task_map "
                "(issue_key, task_id, list_id, status, created_at, updated_at) "
                "VALUES (?, ?, ?, 'open', ?, ?)",
                (issue_key, task_id, list_id, now, now),
            )

    def mark_completed(self, issue_key: str) -> None:
        with self._conn() as c:
            c.execute(
                "UPDATE task_map SET status = 'completed', updated_at = ? "
                "WHERE issue_key = ?",
                (_now(), issue_key),
            )
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/test_state.py -v`
Expected: PASS (both tests)

- [ ] **Step 5: Commit**

```bash
git add app/state.py tests/test_state.py
git commit -m "feat: SQLite state store with idempotent add"
```

---

## Task 3: GitHub payload helper

**Files:**
- Create: `app/github.py`, `tests/test_github.py`
- Test: `tests/test_github.py`

**Interfaces:**
- Consumes: nothing.
- Produces: `parse_due(issue: dict) -> str | None`. Reads `issue["milestone"]["due_on"]` (RFC3339, e.g. `"2026-07-20T07:00:00Z"`), returns a Google-Tasks `due` string `"2026-07-20T00:00:00.000Z"` (date only, midnight UTC). Returns `None` if no milestone or no due date.

Rationale: GitHub's `issues` webhook payload embeds the full milestone object including `due_on`, so no network fetch is needed. `GITHUB_TOKEN` stays reserved in config for future use.

- [ ] **Step 1: Write the failing test**

`tests/test_github.py`:
```python
from app.github import parse_due


def test_parse_due_from_milestone():
    issue = {"milestone": {"due_on": "2026-07-20T07:00:00Z"}}
    assert parse_due(issue) == "2026-07-20T00:00:00.000Z"


def test_parse_due_none_when_no_milestone():
    assert parse_due({"milestone": None}) is None
    assert parse_due({}) is None


def test_parse_due_none_when_milestone_without_due():
    assert parse_due({"milestone": {"due_on": None}}) is None
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_github.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.github'`

- [ ] **Step 3: Write app/github.py**

```python
def parse_due(issue: dict) -> str | None:
    """Return a Google-Tasks `due` string from an Issue's milestone, or None.

    GitHub milestone `due_on` is RFC3339 (e.g. "2026-07-20T07:00:00Z").
    Google Tasks stores date only, so we normalize to midnight UTC.
    """
    milestone = issue.get("milestone")
    if not milestone:
        return None
    due_on = milestone.get("due_on")
    if not due_on:
        return None
    date_part = due_on[:10]  # "YYYY-MM-DD"
    return f"{date_part}T00:00:00.000Z"
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/test_github.py -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add app/github.py tests/test_github.py
git commit -m "feat: parse Google Tasks due date from Issue milestone"
```

---

## Task 4: Google Tasks client

**Files:**
- Create: `app/gtasks.py`, `tests/test_gtasks.py`
- Test: `tests/test_gtasks.py`

**Interfaces:**
- Consumes: nothing (takes a pre-built Google API `service` object for testability).
- Produces:
  - `build_service(token_path, client_id, client_secret)` → googleapiclient service (real; not unit-tested, thin wrapper).
  - `GTasksClient(service)`. Methods:
    - `resolve_list(name: str) -> str` — return list id whose title == `name`; create the list if absent.
    - `create_task(list_id: str, title: str, notes: str, due: str | None = None) -> str` — return new task id.
    - `complete_task(list_id: str, task_id: str) -> None` — patch `status="completed"`.

The `service` object mirrors google-api-python-client: `service.tasklists().list().execute()`, `service.tasklists().insert(body=...).execute()`, `service.tasks().insert(tasklist=, body=).execute()`, `service.tasks().patch(tasklist=, task=, body=).execute()`.

- [ ] **Step 1: Write the failing test**

`tests/test_gtasks.py`:
```python
from app.gtasks import GTasksClient


class FakeExec:
    def __init__(self, result):
        self._result = result

    def execute(self):
        return self._result


class FakeTasklists:
    def __init__(self, existing, created_id="new-list"):
        self._existing = existing
        self._created_id = created_id
        self.inserted = None

    def list(self):
        return FakeExec({"items": self._existing})

    def insert(self, body):
        self.inserted = body
        return FakeExec({"id": self._created_id, "title": body["title"]})


class FakeTasks:
    def __init__(self):
        self.inserted = None
        self.patched = None

    def insert(self, tasklist, body):
        self.inserted = {"tasklist": tasklist, "body": body}
        return FakeExec({"id": "task-1"})

    def patch(self, tasklist, task, body):
        self.patched = {"tasklist": tasklist, "task": task, "body": body}
        return FakeExec({"id": task})


class FakeService:
    def __init__(self, existing_lists):
        self._tl = FakeTasklists(existing_lists)
        self._tk = FakeTasks()

    def tasklists(self):
        return self._tl

    def tasks(self):
        return self._tk


def test_resolve_list_returns_existing_id():
    svc = FakeService([{"id": "L1", "title": "PO"}, {"id": "L2", "title": "Other"}])
    client = GTasksClient(svc)
    assert client.resolve_list("PO") == "L1"
    assert svc._tl.inserted is None  # did not create


def test_resolve_list_creates_when_absent():
    svc = FakeService([{"id": "L1", "title": "Other"}])
    client = GTasksClient(svc)
    assert client.resolve_list("PO") == "new-list"
    assert svc._tl.inserted == {"title": "PO"}


def test_create_task_with_due():
    svc = FakeService([])
    client = GTasksClient(svc)
    tid = client.create_task("L1", "My title", "notes body", due="2026-07-20T00:00:00.000Z")
    assert tid == "task-1"
    assert svc._tk.inserted["tasklist"] == "L1"
    assert svc._tk.inserted["body"] == {
        "title": "My title",
        "notes": "notes body",
        "due": "2026-07-20T00:00:00.000Z",
    }


def test_create_task_without_due_omits_key():
    svc = FakeService([])
    client = GTasksClient(svc)
    client.create_task("L1", "t", "n")
    assert "due" not in svc._tk.inserted["body"]


def test_complete_task_patches_status():
    svc = FakeService([])
    client = GTasksClient(svc)
    client.complete_task("L1", "task-9")
    assert svc._tk.patched == {
        "tasklist": "L1",
        "task": "task-9",
        "body": {"status": "completed"},
    }
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_gtasks.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.gtasks'`

- [ ] **Step 3: Write app/gtasks.py**

```python
import json
from pathlib import Path

from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build

SCOPES = ["https://www.googleapis.com/auth/tasks"]


def build_service(token_path: Path, client_id: str, client_secret: str):
    """Build a Google Tasks API service from a stored oauthlib token dict."""
    tok = json.loads(Path(token_path).read_text())
    creds = Credentials(
        token=tok.get("access_token"),
        refresh_token=tok.get("refresh_token"),
        token_uri="https://oauth2.googleapis.com/token",
        client_id=client_id,
        client_secret=client_secret,
        scopes=SCOPES,
    )
    return build("tasks", "v1", credentials=creds, cache_discovery=False)


class GTasksClient:
    def __init__(self, service):
        self._svc = service

    def resolve_list(self, name: str) -> str:
        resp = self._svc.tasklists().list().execute()
        for item in resp.get("items", []):
            if item.get("title") == name:
                return item["id"]
        created = self._svc.tasklists().insert(body={"title": name}).execute()
        return created["id"]

    def create_task(
        self, list_id: str, title: str, notes: str, due: str | None = None
    ) -> str:
        body = {"title": title, "notes": notes}
        if due:
            body["due"] = due
        created = self._svc.tasks().insert(tasklist=list_id, body=body).execute()
        return created["id"]

    def complete_task(self, list_id: str, task_id: str) -> None:
        self._svc.tasks().patch(
            tasklist=list_id, task=task_id, body={"status": "completed"}
        ).execute()
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/test_gtasks.py -v`
Expected: PASS (all 5)

- [ ] **Step 5: Commit**

```bash
git add app/gtasks.py tests/test_gtasks.py
git commit -m "feat: Google Tasks client (resolve list, create, complete)"
```

---

## Task 5: Bridge (routing + idempotency + mapping)

**Files:**
- Create: `app/bridge.py`, `tests/test_bridge.py`
- Test: `tests/test_bridge.py`

**Interfaces:**
- Consumes: `StateStore` (Task 2), `GTasksClient` (Task 4), `parse_due` (Task 3), `Config` (Task 1).
- Produces: `Bridge(state, gtasks, config)`. Method `handle(action: str, payload: dict) -> None`. Routes per the spec table. `payload` is the parsed GitHub webhook body: `payload["action"]`, `payload["issue"]`, `payload["repository"]["full_name"]`, and for label events `payload["label"]["name"]`.

Routing:
- `labeled` + label == `config.po_label`: if `state.get(key)` is None → `list_id = gtasks.resolve_list(...)`, `task_id = gtasks.create_task(...)`, then `state.add(...)`. Else no-op.
- `unlabeled` + label == `config.po_label`: if row exists and open → `gtasks.complete_task`, `state.mark_completed`.
- `closed`: if row exists and open → complete + mark.
- anything else → no-op.

- [ ] **Step 1: Write the failing test**

`tests/test_bridge.py`:
```python
import pytest
from app.bridge import Bridge


class FakeState:
    def __init__(self):
        self.rows = {}

    def get(self, k):
        return self.rows.get(k)

    def add(self, issue_key, task_id, list_id):
        self.rows.setdefault(
            issue_key,
            {"issue_key": issue_key, "task_id": task_id, "list_id": list_id, "status": "open"},
        )

    def mark_completed(self, issue_key):
        if issue_key in self.rows:
            self.rows[issue_key]["status"] = "completed"


class FakeGTasks:
    def __init__(self):
        self.created = []
        self.completed = []

    def resolve_list(self, name):
        return "L1"

    def create_task(self, list_id, title, notes, due=None):
        self.created.append({"list_id": list_id, "title": title, "notes": notes, "due": due})
        return f"task-{len(self.created)}"

    def complete_task(self, list_id, task_id):
        self.completed.append({"list_id": list_id, "task_id": task_id})


class Cfg:
    po_label = "po-task"
    google_tasklist = "PO"


def _payload(action, *, label="po-task", with_milestone=False):
    issue = {
        "number": 42,
        "title": "Do the thing",
        "body": "details here",
        "html_url": "https://github.com/kibarik/mts-po-workspace/issues/42",
        "milestone": {"due_on": "2026-07-20T07:00:00Z"} if with_milestone else None,
    }
    return {
        "action": action,
        "issue": issue,
        "repository": {"full_name": "kibarik/mts-po-workspace"},
        "label": {"name": label},
    }


@pytest.fixture
def bridge():
    return Bridge(FakeState(), FakeGTasks(), Cfg())


def test_labeled_creates_task_with_mapped_fields(bridge):
    bridge.handle("labeled", _payload("labeled", with_milestone=True))
    created = bridge._gtasks.created
    assert len(created) == 1
    assert created[0]["title"] == "Do the thing"
    assert created[0]["notes"] == (
        "details here\n\nhttps://github.com/kibarik/mts-po-workspace/issues/42"
    )
    assert created[0]["due"] == "2026-07-20T00:00:00.000Z"
    assert bridge._state.get("kibarik/mts-po-workspace#42")["task_id"] == "task-1"


def test_labeled_without_milestone_has_no_due(bridge):
    bridge.handle("labeled", _payload("labeled"))
    assert bridge._gtasks.created[0]["due"] is None


def test_duplicate_labeled_is_idempotent(bridge):
    bridge.handle("labeled", _payload("labeled"))
    bridge.handle("labeled", _payload("labeled"))
    assert len(bridge._gtasks.created) == 1


def test_wrong_label_ignored(bridge):
    bridge.handle("labeled", _payload("labeled", label="bug"))
    assert bridge._gtasks.created == []


def test_closed_completes_existing_task(bridge):
    bridge.handle("labeled", _payload("labeled"))
    bridge.handle("closed", _payload("closed"))
    assert bridge._gtasks.completed == [{"list_id": "L1", "task_id": "task-1"}]
    assert bridge._state.get("kibarik/mts-po-workspace#42")["status"] == "completed"


def test_unlabeled_completes_existing_task(bridge):
    bridge.handle("labeled", _payload("labeled"))
    bridge.handle("unlabeled", _payload("unlabeled"))
    assert len(bridge._gtasks.completed) == 1


def test_closed_without_mapping_is_noop(bridge):
    bridge.handle("closed", _payload("closed"))
    assert bridge._gtasks.completed == []


def test_completing_twice_is_noop(bridge):
    bridge.handle("labeled", _payload("labeled"))
    bridge.handle("closed", _payload("closed"))
    bridge.handle("unlabeled", _payload("unlabeled"))
    assert len(bridge._gtasks.completed) == 1  # already completed, skip
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_bridge.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.bridge'`

- [ ] **Step 3: Write app/bridge.py**

```python
from app.github import parse_due


class Bridge:
    def __init__(self, state, gtasks, config):
        self._state = state
        self._gtasks = gtasks
        self._cfg = config

    def _key(self, payload: dict) -> str:
        repo = payload["repository"]["full_name"]
        number = payload["issue"]["number"]
        return f"{repo}#{number}"

    def _is_po_label(self, payload: dict) -> bool:
        label = payload.get("label") or {}
        return label.get("name") == self._cfg.po_label

    def handle(self, action: str, payload: dict) -> None:
        if action == "labeled" and self._is_po_label(payload):
            self._create(payload)
        elif action == "unlabeled" and self._is_po_label(payload):
            self._complete(payload)
        elif action == "closed":
            self._complete(payload)
        # else: ignore

    def _create(self, payload: dict) -> None:
        key = self._key(payload)
        if self._state.get(key) is not None:
            return  # idempotent: already tracked
        issue = payload["issue"]
        list_id = self._gtasks.resolve_list(self._cfg.google_tasklist)
        notes = f"{issue.get('body') or ''}\n\n{issue['html_url']}"
        task_id = self._gtasks.create_task(
            list_id,
            title=issue["title"],
            notes=notes,
            due=parse_due(issue),
        )
        self._state.add(key, task_id=task_id, list_id=list_id)

    def _complete(self, payload: dict) -> None:
        key = self._key(payload)
        row = self._state.get(key)
        if row is None or row["status"] == "completed":
            return  # nothing to do
        self._gtasks.complete_task(row["list_id"], row["task_id"])
        self._state.mark_completed(key)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/test_bridge.py -v`
Expected: PASS (all 8)

- [ ] **Step 5: Commit**

```bash
git add app/bridge.py tests/test_bridge.py
git commit -m "feat: bridge event routing with idempotency and field mapping"
```

---

## Task 6: FastAPI webhook + HMAC verification

**Files:**
- Create: `app/webhook.py`, `tests/conftest.py`, `tests/test_webhook.py`
- Test: `tests/test_webhook.py`

**Interfaces:**
- Consumes: `Config`, `StateStore`, `GTasksClient`/`build_service`, `Bridge`.
- Produces:
  - `verify_signature(secret: str, body: bytes, header: str | None) -> bool` — constant-time HMAC-SHA256; `header` is `"sha256=<hex>"`.
  - `create_app(bridge, config) -> FastAPI` — `GET /health` → `{"status": "ok"}`; `POST /webhook` → verify signature (401 on fail), only handle `X-GitHub-Event: issues`, call `bridge.handle(payload["action"], payload)`, return `{"ok": True}`.
  - module-level `app` built from real `Config` + wired dependencies (import-time; used by uvicorn).

- [ ] **Step 1: Write conftest + failing test**

`tests/conftest.py`:
```python
import hashlib
import hmac
import json


def sign(secret: str, body: bytes) -> str:
    return "sha256=" + hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()


def body_bytes(payload: dict) -> bytes:
    return json.dumps(payload).encode()
```

`tests/test_webhook.py`:
```python
from fastapi.testclient import TestClient

from app.webhook import create_app, verify_signature
from tests.conftest import sign, body_bytes


class FakeBridge:
    def __init__(self):
        self.calls = []

    def handle(self, action, payload):
        self.calls.append((action, payload["issue"]["number"]))


class Cfg:
    github_webhook_secret = "whsec"


def _client(bridge):
    return TestClient(create_app(bridge, Cfg()))


def _issue_payload():
    return {
        "action": "labeled",
        "issue": {"number": 7},
        "repository": {"full_name": "o/r"},
        "label": {"name": "po-task"},
    }


def test_verify_signature_valid_and_invalid():
    body = b'{"a":1}'
    good = sign("whsec", body)
    assert verify_signature("whsec", body, good) is True
    assert verify_signature("whsec", body, "sha256=deadbeef") is False
    assert verify_signature("whsec", body, None) is False


def test_health_ok():
    r = _client(FakeBridge()).get("/health")
    assert r.status_code == 200
    assert r.json() == {"status": "ok"}


def test_webhook_rejects_bad_signature():
    bridge = FakeBridge()
    body = body_bytes(_issue_payload())
    r = _client(bridge).post(
        "/webhook",
        content=body,
        headers={"X-GitHub-Event": "issues", "X-Hub-Signature-256": "sha256=bad"},
    )
    assert r.status_code == 401
    assert bridge.calls == []


def test_webhook_dispatches_issues_event():
    bridge = FakeBridge()
    body = body_bytes(_issue_payload())
    r = _client(bridge).post(
        "/webhook",
        content=body,
        headers={"X-GitHub-Event": "issues", "X-Hub-Signature-256": sign("whsec", body)},
    )
    assert r.status_code == 200
    assert bridge.calls == [("labeled", 7)]


def test_webhook_ignores_non_issues_event():
    bridge = FakeBridge()
    body = body_bytes({"action": "opened"})
    r = _client(bridge).post(
        "/webhook",
        content=body,
        headers={"X-GitHub-Event": "ping", "X-Hub-Signature-256": sign("whsec", body)},
    )
    assert r.status_code == 200
    assert bridge.calls == []
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_webhook.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.webhook'`

- [ ] **Step 3: Write app/webhook.py**

```python
import hashlib
import hmac

from fastapi import FastAPI, Request, Response


def verify_signature(secret: str, body: bytes, header: str | None) -> bool:
    if not header:
        return False
    expected = "sha256=" + hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, header)


def create_app(bridge, config) -> FastAPI:
    app = FastAPI()

    @app.get("/health")
    def health():
        return {"status": "ok"}

    @app.post("/webhook")
    async def webhook(request: Request):
        body = await request.body()
        sig = request.headers.get("X-Hub-Signature-256")
        if not verify_signature(config.github_webhook_secret, body, sig):
            return Response(status_code=401)
        if request.headers.get("X-GitHub-Event") != "issues":
            return {"ok": True, "ignored": True}
        import json

        payload = json.loads(body)
        bridge.handle(payload.get("action"), payload)
        return {"ok": True}

    return app


def _build_app() -> FastAPI:
    from app.bridge import Bridge
    from app.config import Config
    from app.gtasks import GTasksClient, build_service
    from app.state import StateStore

    cfg = Config()
    service = build_service(cfg.token_path, cfg.google_client_id, cfg.google_client_secret)
    bridge = Bridge(StateStore(cfg.db_path), GTasksClient(service), cfg)
    return create_app(bridge, cfg)


# uvicorn entry point: `uvicorn app.webhook:app`
# Guarded so importing this module in tests does not require real config/token.
app = None
try:
    app = _build_app()
except Exception:  # pragma: no cover - only fails outside a configured container
    app = None
```

Note: the `try/except` lets tests import `create_app`/`verify_signature` without a real `.env` or Google token present. Inside the container both exist, so `app` is a live FastAPI instance.

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/test_webhook.py -v`
Expected: PASS (all 5)

- [ ] **Step 5: Run the full suite**

Run: `pytest -v`
Expected: PASS — all tests across tasks 1-6.

- [ ] **Step 6: Commit**

```bash
git add app/webhook.py tests/conftest.py tests/test_webhook.py
git commit -m "feat: FastAPI webhook with HMAC verification and event dispatch"
```

---

## Task 7: OAuth bootstrap, Docker, docs

**Files:**
- Create: `google_token_bootstrap.py`, `Dockerfile`, `docker-compose.yml`, `README.md`

**Interfaces:**
- Consumes: `secrets/google_client_id`, `secrets/google_client_secret`.
- Produces: `state/google_tasks.token` (oauthlib dict), a runnable container image, deploy docs.

This task has no unit tests (I/O + browser + container). Verify manually per the README steps.

- [ ] **Step 1: Write google_token_bootstrap.py**

```python
#!/usr/bin/env python3
"""Mint a Google OAuth token for the Tasks API (scope: tasks).

Run once on a machine with a browser. Reads client_id/secret from ./secrets/,
opens the Google consent screen, writes ./state/google_tasks.token in the
oauthlib dict format the service reads.

Deps (host):  pip install google-auth-oauthlib
"""
import calendar
import json
import pathlib

from google_auth_oauthlib.flow import InstalledAppFlow

SCOPES = ["https://www.googleapis.com/auth/tasks"]
base = pathlib.Path(__file__).parent
cid = (base / "secrets/google_client_id").read_text().strip()
csec = (base / "secrets/google_client_secret").read_text().strip()

cfg = {
    "installed": {
        "client_id": cid,
        "client_secret": csec,
        "auth_uri": "https://accounts.google.com/o/oauth2/auth",
        "token_uri": "https://oauth2.googleapis.com/token",
        "redirect_uris": ["http://localhost"],
    }
}

flow = InstalledAppFlow.from_client_config(cfg, SCOPES)
creds = flow.run_local_server(port=0)

exp = 0
if creds.expiry:
    exp = calendar.timegm(creds.expiry.timetuple())

token = {
    "access_token": creds.token,
    "refresh_token": creds.refresh_token,
    "token_type": "Bearer",
    "scope": list(SCOPES),
    "expires_at": exp - 60,
}
(base / "state").mkdir(exist_ok=True)
(base / "state/google_tasks.token").write_text(json.dumps(token))
print("Wrote state/google_tasks.token")
```

- [ ] **Step 2: Write Dockerfile**

```dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ ./app/

EXPOSE 8080
CMD ["uvicorn", "app.webhook:app", "--host", "0.0.0.0", "--port", "8080"]
```

- [ ] **Step 3: Write docker-compose.yml**

```yaml
services:
  po-task:
    build: .
    restart: unless-stopped
    env_file: .env
    ports:
      - "8080:8080"        # reverse-proxy forwards here
    volumes:
      - ./secrets:/app/secrets:ro
      - ./state:/app/state
```

- [ ] **Step 4: Write README.md**

````markdown
# poh-po-task

Turns a GitHub Issue labeled `po-task` into a Google Task (and completes it when the
Issue closes or the label is removed) so [reclaim.ai](https://reclaim.ai) can schedule it.

Standalone service — independent of poh-scheduller. See the design in
`docs/superpowers/specs/2026-07-19-po-task-gtask-bridge-design.md`.

## Setup

1. **Secrets** — put your Google OAuth installed-app credentials in files:
   ```
   secrets/google_client_id
   secrets/google_client_secret
   ```
2. **Config** — `cp .env.example .env` and fill in `GITHUB_WEBHOOK_SECRET`,
   `GITHUB_REPO`, and (optionally) `GOOGLE_TASKLIST`.
3. **Google token** (one-off, needs a browser):
   ```bash
   pip install google-auth-oauthlib
   python google_token_bootstrap.py
   ```
   Writes `state/google_tasks.token`.
4. **Run**:
   ```bash
   docker compose up -d
   ```
5. **GitHub webhook** — in `mts-po-workspace` → Settings → Webhooks → Add webhook:
   - Payload URL: your reverse-proxy HTTPS URL → `/webhook`
   - Content type: `application/json`
   - Secret: same as `GITHUB_WEBHOOK_SECRET`
   - Events: **Issues** only

## How it works

`labeled(po-task)` → create task. `closed` or `unlabeled(po-task)` → complete task.
Idempotent via `state/bridge.db`. Only `issues` events are processed; signatures are
verified (`X-Hub-Signature-256`).

## Tests

```bash
pip install -r requirements.txt
pytest -v
```
````

- [ ] **Step 5: Verify the image builds and health responds**

```bash
docker compose build
# Smoke test without Google creds: the container starts; app may be None until
# creds/token exist, so verify locally instead:
pip install -r requirements.txt && pytest -v
```
Expected: `pytest` PASS. `docker compose build` succeeds.

- [ ] **Step 6: Commit**

```bash
git add google_token_bootstrap.py Dockerfile docker-compose.yml README.md
git commit -m "feat: OAuth bootstrap, Docker packaging, and deploy docs"
```

---

## Self-Review

**Spec coverage:**
- FR-1 (labeled→create) → Task 5. FR-2 (title/notes/due mapping) → Task 3 + Task 5. FR-3 (idempotency) → Task 2 + Task 5. FR-4 (closed/unlabeled→complete) → Task 5. FR-5 (list by name, create) → Task 4. FR-6 (repo/label config) → Task 1.
- NFR-1 (webhook) → Task 6. NFR-2 (HMAC) → Task 6. NFR-3 (standalone) → Task 7 (compose/volumes). NFR-4 (deploy steps) → Task 7 README. NFR-5 (read-only PAT, separate token/scope) → Task 1 + Task 4/7.
- Testing section → tasks 1-6 unit tests.

**Placeholder scan:** none — every step has concrete code or commands.

**Type consistency:** `resolve_list`, `create_task(list_id,title,notes,due)`, `complete_task(list_id,task_id)`, `StateStore.get/add/mark_completed`, `parse_due(issue)`, `Bridge.handle(action,payload)`, `verify_signature(secret,body,header)`, `create_app(bridge,config)` — names and signatures match across tasks 1-7.
