# Docker Compose Startup Fixes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `docker compose up -d` on a fresh clone yields a healthy backend + frontend, without breaking existing deployments (existing DBs, existing `jwt_secret.txt`).

**Architecture:** Schema bootstrap moves to Alembic (already in the image, never invoked), driven by a small `migrate.py` that stamps pre-Alembic databases before upgrading. The entrypoint restores the documented "JWT secret auto-generated on first run" behaviour. The external `proxy` network becomes opt-in via the override example.

**Tech Stack:** Docker Compose, bash entrypoint, Python 3.11, SQLAlchemy, Alembic, pydantic-settings, SQLite.

**Spec:** No spec doc — the investigation findings below are the spec.

## Investigation Findings (spec)

| # | Symptom on fresh `up -d` | Root cause |
|---|---|---|
| F1 | `network proxy declared as external, but could not be found` | `docker-compose.yml` requires a pre-created external `proxy` network |
| F2 | Backend crash loop: `ALTER TABLE users ...: no such table: users` | `backend/docker-entrypoint.sh:21-26` — `PRAGMA table_info` on a missing table returns `[]` |
| F3 | No tables ever created | `init_db()` (`backend/app/db/__init__.py:85`) has no callers; `alembic upgrade head` never runs. Alembic schema == ORM except `users.is_admin` |
| F4 | `JWT_SECRET_KEY is still the development default` | Nothing generates `jwt_secret.txt` any more; compose never forwards `JWT_SECRET_KEY` |
| F5 | Healthchecks never pass | backend image has no `wget` (has `curl`); frontend `localhost` → `::1`, nginx listens IPv4 only |
| F6 | `start.sh` vars ignored | compose reads `DATA_DIR`, not `EINK_DATA_DIR`; `JWT_SECRET_KEY` not forwarded |

## Decisions (defaults chosen — reject before execution if wrong)

| Decision | Default | Consequence |
|---|---|---|
| External `proxy` network | Removed from base compose; documented in `docker-compose.override.yml.example` | Deployments relying on it must add the override snippet once |
| Pre-Alembic databases (tables, no `alembic_version`) | `migrate.py` auto-stamps: `3b2e8a4f1d6b` if `pdf_jobs.diagnostics` exists, else `29e8204f390a`; then `upgrade head` | Assumes legacy schema otherwise matches the initial migration |
| Empty `JWT_SECRET_KEY` (compose forwards `${JWT_SECRET_KEY:-}`) | Normalised to "unset" in `config.py` so file fallback + production check still apply | Closes the empty-secret bypass of `assert_production_safe` |
| `start.sh` data dir | `DATA_DIR=/home/artur/Documents/eink-data` (not `~/Documents` root — the container writes root-owned `users/`, `assets/`, `jobs/`, sqlite there) | `start.sh` is untracked; edited locally, never committed |

## Global Constraints

- Work on branch `fix/docker-compose-startup` off `main`.
- No new dependencies (Python or system packages). No Dockerfile changes.
- No `pytest` on host or in image: the runnable check is `backend/scripts/check_startup.py`, run inside `eink-backend:latest` with `backend/` bind-mounted.
- Existing deployments must keep working: existing `jwt_secret.txt` is reused, never regenerated; existing DBs are stamped, not recreated.
- Commit messages end with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Out of scope (do not touch): `docs/DOCKER_SETUP.md`, `backend/scripts/add_is_admin_column.py`, `backend/app/db/__init__.py:init_db`, frontend code, Dockerfiles.

## File Map

| File | Action | Responsibility |
|---|---|---|
| `backend/alembic/versions/5c1d9e7a2f40_add_users_is_admin.py` | Create | Idempotent migration adding `users.is_admin` |
| `backend/scripts/migrate.py` | Create | Stamp pre-Alembic DBs, then `alembic upgrade head` |
| `backend/scripts/check_startup.py` | Create | Assert-based self-check: migration scenarios + JWT secret resolution |
| `backend/docker-entrypoint.sh` | Modify | Generate JWT secret file if needed; call `migrate.py` |
| `backend/app/config.py:127` | Modify | Treat empty `JWT_SECRET_KEY` as unset |
| `docker-compose.yml` | Modify | Forward `JWT_SECRET_KEY`, drop external network, fix healthchecks, fix stale comment |
| `docker-compose.override.yml.example` | Modify | Opt-in external `proxy` network snippet |
| `README.md:94-108`, `.env.example` | Modify | Correct env var docs |
| `start.sh` | Modify (local, uncommitted) | `DATA_DIR`, drop placeholder secret |

Shared commands (repo root):

```bash
# Run the self-check inside the backend image (backend/ bind-mounted, no rebuild needed)
CHECK='docker run --rm --entrypoint python3 -v "$PWD/backend:/app/backend" eink-backend:latest scripts/check_startup.py'
```

---

### Task 1: Alembic-driven schema bootstrap (F2, F3)

**Files:**
- Create: `backend/alembic/versions/5c1d9e7a2f40_add_users_is_admin.py`
- Create: `backend/scripts/migrate.py`
- Create: `backend/scripts/check_startup.py`
- Modify: `backend/docker-entrypoint.sh` (replace the whole Python heredoc, lines 4-33)

**Interfaces:**
- Consumes: Alembic revisions `29e8204f390a` (initial) → `3b2e8a4f1d6b` (pdf_jobs.diagnostics); `app.config.settings.DATABASE_URL` (env `DATABASE_URL`, default `sqlite:///data/einkpdf.sqlite`, relative to cwd).
- Produces: head revision `5c1d9e7a2f40`; `python3 /app/backend/scripts/migrate.py` (exit 0 on success, raises on failure); `check_startup.py` with `run()`, `HEAD`, `check_migrations()` used again by Task 2.

- [ ] **Step 1: Create branch and commit the plan**

```bash
git checkout -b fix/docker-compose-startup
git add docs/superpowers/plans/2026-10-10-docker-compose-startup-fixes.md
git commit -m "docs: plan for docker compose startup fixes

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 2: Write the failing check** — `backend/scripts/check_startup.py`

```python
#!/usr/bin/env python3
"""Self-check for container startup: schema migration and JWT secret resolution.

Run inside the backend image (it has alembic/sqlalchemy; the host may not):
    docker run --rm --entrypoint python3 -v "$PWD/backend:/app/backend" \
        eink-backend:latest scripts/check_startup.py
"""
import os
import sqlite3
import subprocess
import sys
import tempfile
from pathlib import Path

BACKEND = Path(__file__).resolve().parent.parent
HEAD = "5c1d9e7a2f40"


def run(args, db, extra_env=None):
    """Run python with DATABASE_URL pointed at db; return stdout."""
    env = {**os.environ, "DATABASE_URL": f"sqlite:///{db}", **(extra_env or {})}
    return subprocess.run(
        [sys.executable, *args], cwd=db.parent, env=env,
        check=True, capture_output=True, text=True,
    ).stdout


def alembic(db, *args):
    run(["-m", "alembic", "-c", str(BACKEND / "alembic.ini"), *args], db)


def sql(db, statement):
    conn = sqlite3.connect(db)
    rows = conn.execute(statement).fetchall()
    conn.commit()
    conn.close()
    return rows


def columns(db, table):
    return {row[1] for row in sql(db, f"PRAGMA table_info({table})")}


def add_is_admin(db):
    # What the old entrypoint heredoc did to running deployments.
    sql(db, "ALTER TABLE users ADD COLUMN is_admin BOOLEAN NOT NULL DEFAULT 0")


SCENARIOS = {
    "fresh database": lambda db: None,
    "pre-alembic, initial schema": lambda db: (
        alembic(db, "upgrade", "29e8204f390a"),
        sql(db, "DROP TABLE alembic_version"),
    ),
    "pre-alembic, diagnostics + is_admin": lambda db: (
        alembic(db, "upgrade", "3b2e8a4f1d6b"),
        add_is_admin(db),
        sql(db, "DROP TABLE alembic_version"),
    ),
    "alembic-tracked, is_admin added by old entrypoint": lambda db: (
        alembic(db, "upgrade", "3b2e8a4f1d6b"),
        add_is_admin(db),
    ),
}


def check_migrations():
    for name, setup in SCENARIOS.items():
        with tempfile.TemporaryDirectory() as tmp:
            db = Path(tmp) / "test.sqlite"
            setup(db)
            for _ in range(2):  # second run proves idempotence
                run([str(BACKEND / "scripts" / "migrate.py")], db)
            assert sql(db, "SELECT version_num FROM alembic_version") == [(HEAD,)], name
            assert "is_admin" in columns(db, "users"), name
            assert "diagnostics" in columns(db, "pdf_jobs"), name
            print(f"ok  migrate: {name}")


if __name__ == "__main__":
    check_migrations()
```

- [ ] **Step 3: Run it to verify it fails**

Run: `eval "$CHECK"`
Expected: FAIL — `CalledProcessError` on `.../scripts/migrate.py` (file does not exist).

- [ ] **Step 4: Create the migration** — `backend/alembic/versions/5c1d9e7a2f40_add_users_is_admin.py`

```python
"""Add is_admin column to users

Revision ID: 5c1d9e7a2f40
Revises: 3b2e8a4f1d6b
Create Date: 2026-10-10 00:00:00.000000
"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision: str = '5c1d9e7a2f40'
down_revision: Union[str, Sequence[str], None] = '3b2e8a4f1d6b'
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    # Deployments that ran the old docker-entrypoint.sh already have the column.
    existing = {c['name'] for c in sa.inspect(op.get_bind()).get_columns('users')}
    if 'is_admin' not in existing:
        op.add_column('users', sa.Column('is_admin', sa.Boolean(), nullable=False, server_default=sa.false()))


def downgrade() -> None:
    op.drop_column('users', 'is_admin')
```

- [ ] **Step 5: Create the runner** — `backend/scripts/migrate.py`

```python
#!/usr/bin/env python3
"""Bring the database schema to alembic head. Called by docker-entrypoint.sh.

Databases created before alembic was wired in (tables via create_all, no
alembic_version table) are first stamped at the revision their schema matches.
"""
import sys
from pathlib import Path

BACKEND = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(BACKEND))

from alembic import command  # noqa: E402
from alembic.config import Config  # noqa: E402
from sqlalchemy import create_engine, inspect  # noqa: E402

from app.config import settings  # noqa: E402

INITIAL = "29e8204f390a"
DIAGNOSTICS = "3b2e8a4f1d6b"


def legacy_revision() -> str | None:
    """Revision a pre-alembic database matches; None if alembic can handle it as-is."""
    engine = create_engine(settings.DATABASE_URL)
    try:
        insp = inspect(engine)
        tables = insp.get_table_names()
        if "alembic_version" in tables or "users" not in tables:
            return None
        pdf_job_columns = {c["name"] for c in insp.get_columns("pdf_jobs")}
        return DIAGNOSTICS if "diagnostics" in pdf_job_columns else INITIAL
    finally:
        engine.dispose()


def main() -> None:
    cfg = Config(str(BACKEND / "alembic.ini"))
    revision = legacy_revision()
    if revision:
        print(f"Pre-alembic database detected; stamping {revision}")
        command.stamp(cfg, revision)
    command.upgrade(cfg, "head")


if __name__ == "__main__":
    main()
```

- [ ] **Step 6: Run the check to verify it passes**

Run: `eval "$CHECK"`
Expected: four `ok  migrate: ...` lines, exit 0.

- [ ] **Step 7: Replace the entrypoint heredoc** — `backend/docker-entrypoint.sh` becomes:

```bash
#!/bin/bash
set -e

echo "Running database migrations..."
python3 /app/backend/scripts/migrate.py

echo "Starting application..."
exec "$@"
```

- [ ] **Step 8: Verify entrypoint on an empty data dir**

```bash
T=$(mktemp -d); docker run --rm -e DEBUG=True -v "$PWD/backend:/app/backend" -v "$T:/app/backend/data" \
  eink-backend:latest python3 -c "import sqlite3;c=sqlite3.connect('/app/backend/data/einkpdf.sqlite');print(sorted(r[0] for r in c.execute('select name from sqlite_master where type=\"table\"')))"
docker run --rm -v "$T:/d" alpine rm -rf /d/*; rmdir "$T"
```
Expected: `['alembic_version', 'assets', 'password_reset_tokens', 'pdf_jobs', 'projects', 'public_projects', 'user_quotas', 'users']`

- [ ] **Step 9: Commit**

```bash
git add backend/alembic/versions/5c1d9e7a2f40_add_users_is_admin.py backend/scripts/migrate.py backend/scripts/check_startup.py backend/docker-entrypoint.sh
git commit -m "fix(backend): bootstrap schema with alembic on container start

Fresh volumes crash-looped: the entrypoint ALTERed a users table that
nothing ever created (init_db has no callers, alembic never ran).
migrate.py stamps pre-alembic databases, then upgrades to head; is_admin
is now an idempotent alembic revision.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: JWT secret on first run (F4)

**Files:**
- Modify: `backend/app/config.py:127` (`_resolve_jwt_secret_from_file`)
- Modify: `backend/docker-entrypoint.sh` (insert secret generation before migrations)
- Modify: `backend/scripts/check_startup.py` (add `check_jwt_secret`)

**Interfaces:**
- Consumes: `run()` and `check_migrations()` from Task 1's `check_startup.py`; env `JWT_SECRET_KEY`, `JWT_SECRET_KEY_FILE`.
- Produces: empty `JWT_SECRET_KEY` behaves as unset; entrypoint writes `$JWT_SECRET_KEY_FILE` (mode 600, 64 hex chars) when `JWT_SECRET_KEY` is empty and the file is missing/empty. Task 3 relies on `JWT_SECRET_KEY=${JWT_SECRET_KEY:-}` being safe.

- [ ] **Step 1: Add the failing check** — in `backend/scripts/check_startup.py`, add after `check_migrations`:

```python
DEV_SECRET = "dev-secret-key-change-in-production"


def jwt_secret(tmp, **env):
    code = f"import sys; sys.path.insert(0, {str(BACKEND)!r}); from app.config import settings; print(settings.JWT_SECRET_KEY)"
    return run(["-c", code], Path(tmp) / "test.sqlite", env).strip()


def check_jwt_secret():
    with tempfile.TemporaryDirectory() as tmp:
        secret_file = Path(tmp) / "jwt_secret.txt"
        secret_file.write_text("from-file\n")
        missing = str(Path(tmp) / "missing.txt")
        # compose forwards ${JWT_SECRET_KEY:-}, so "" must mean "unset", never a usable secret
        assert jwt_secret(tmp, JWT_SECRET_KEY="", JWT_SECRET_KEY_FILE=str(secret_file)) == "from-file"
        assert jwt_secret(tmp, JWT_SECRET_KEY="", JWT_SECRET_KEY_FILE=missing) == DEV_SECRET
        assert jwt_secret(tmp, JWT_SECRET_KEY="explicit", JWT_SECRET_KEY_FILE=str(secret_file)) == "explicit"
        print("ok  jwt secret resolution")
```

and change the `__main__` block to:

```python
if __name__ == "__main__":
    check_migrations()
    check_jwt_secret()
```

- [ ] **Step 2: Run to verify it fails**

Run: `eval "$CHECK"`
Expected: migrations ok, then `AssertionError` on the first `jwt_secret` assert (returns `""`).

- [ ] **Step 3: Fix `backend/app/config.py`** — replace line 127-128:

```python
        if self.JWT_SECRET_KEY != self._DEV_JWT_SECRET_KEY:
            return self
```

with:

```python
        # An empty value (e.g. compose's ${JWT_SECRET_KEY:-}) means "unset":
        # fall back to the file, and keep assert_production_safe effective.
        if not self.JWT_SECRET_KEY:
            object.__setattr__(self, "JWT_SECRET_KEY", self._DEV_JWT_SECRET_KEY)
        if self.JWT_SECRET_KEY != self._DEV_JWT_SECRET_KEY:
            return self
```

- [ ] **Step 4: Run to verify it passes**

Run: `eval "$CHECK"`
Expected: four `ok  migrate` lines + `ok  jwt secret resolution`.

- [ ] **Step 5: Add secret generation to `backend/docker-entrypoint.sh`** — insert after `set -e`:

```bash

# Generate a JWT secret on first run unless one is supplied directly.
# An existing file (e.g. from earlier deployments) is always reused.
if [ -z "${JWT_SECRET_KEY:-}" ] && [ -n "${JWT_SECRET_KEY_FILE:-}" ] && [ ! -s "$JWT_SECRET_KEY_FILE" ]; then
    mkdir -p "$(dirname "$JWT_SECRET_KEY_FILE")"
    (umask 077 && python3 -c 'import secrets; print(secrets.token_hex(32))' > "$JWT_SECRET_KEY_FILE")
    echo "Generated JWT secret at $JWT_SECRET_KEY_FILE"
fi
```

- [ ] **Step 6: Verify generation, reuse, and explicit-secret skip**

```bash
T=$(mktemp -d); F=/app/backend/data/users/jwt_secret.txt
R() { docker run --rm -e JWT_SECRET_KEY_FILE=$F "$@" -v "$PWD/backend:/app/backend" -v "$T:/app/backend/data" eink-backend:latest sh -c "stat -c '%a %s' $F 2>/dev/null || echo absent; md5sum $F 2>/dev/null"; }
R; R; rm -rf "$T"/users 2>/dev/null || docker run --rm -v "$T:/d" alpine rm -rf /d/users
R -e JWT_SECRET_KEY=explicit
docker run --rm -v "$T:/d" alpine rm -rf /d/*; rmdir "$T"
```
Expected: run 1 `600 65` + hash; run 2 same hash (reused); run 3 `absent`.

- [ ] **Step 7: Commit**

```bash
git add backend/app/config.py backend/docker-entrypoint.sh backend/scripts/check_startup.py
git commit -m "fix(backend): generate JWT secret file on first container start

Fresh deployments refused to start with DEBUG=False: compose points at
jwt_secret.txt, which no code generates any more. The entrypoint creates
it (mode 600) when no explicit secret is set; existing files are reused.
An empty JWT_SECRET_KEY is now treated as unset instead of as a valid key.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Compose file, override example, docs (F1, F5, F6 docs)

**Files:**
- Modify: `docker-compose.yml`
- Modify: `docker-compose.override.yml.example`
- Modify: `README.md:94-108`
- Modify: `.env.example` (lines 26-32, section 4 lines 104-115)

**Interfaces:**
- Consumes: Task 2 guarantees `JWT_SECRET_KEY=""` is safe; Task 2 entrypoint generates the file.
- Produces: compose with no external network, working healthchecks; override snippet for `proxy`.

- [ ] **Step 1: Edit `docker-compose.yml`**

1a. Replace the JWT block (lines 10-17, from `# JWT secret. Provide ONE of:` through `- JWT_SECRET_KEY_FILE=...`) with:

```yaml
      # JWT secret. Provide ONE of:
      #   JWT_SECRET_KEY      — the secret value itself (set in your .env)
      #   JWT_SECRET_KEY_FILE — path inside the container to a file holding it
      # If JWT_SECRET_KEY is unset, docker-entrypoint.sh generates the file
      # below on first run, so a fresh deployment needs no secret configured.
      - JWT_SECRET_KEY=${JWT_SECRET_KEY:-}
      - JWT_SECRET_KEY_FILE=/app/backend/data/users/jwt_secret.txt
```

1b. Delete both service-level blocks:

```yaml
    networks:
      - proxy
```

1c. Delete the top-level block at the end of the file:

```yaml
networks:
  proxy:
    external: true
```

1d. Backend healthcheck `test:` line becomes:

```yaml
      test: ["CMD-SHELL", "curl -fsS http://localhost:8000/health > /dev/null || exit 1"]
```

1e. Frontend healthcheck `test:` line becomes:

```yaml
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1/ > /dev/null || exit 1"]
```

- [ ] **Step 2: Validate compose**

Run: `docker compose config -q && docker compose config | grep -cE 'external|proxy:'`
Expected: no errors; count `0`.

- [ ] **Step 3: Append to `docker-compose.override.yml.example`**

```yaml

# Optional: attach to an existing reverse-proxy network (Traefik, nginx-proxy,
# ...) created beforehand with `docker network create proxy`. List `default`
# too, or the services lose the network they use to reach each other.
#
#  frontend:
#    networks: [default, proxy]
#  backend:
#    networks: [default, proxy]
#
#networks:
#  proxy:
#    external: true
```

Note: the `frontend:`/`backend:` lines are meant to sit under the existing `services:` key; the comment is the documentation.

- [ ] **Step 4: Edit `README.md`** — replace lines 95-99 (the env var code block body):

```bash
FRONTEND_PORT=3000              # Web interface port
DATA_DIR=./data                 # Host dir mounted as backend data (DB, assets, JWT secret)
EINK_CLEANUP_TTL_DAYS=14        # Auto-cleanup (0 to disable)
JWT_SECRET_KEY=your-secret-key  # Optional; generated into DATA_DIR/users/jwt_secret.txt if unset
```

and after the `### Docker Compose Override` code block (after line 107's closing fence) add:

```markdown
To attach the stack to an existing reverse-proxy network, see `docker-compose.override.yml.example`.
```

- [ ] **Step 5: Edit `.env.example`**

Replace line 26 `# [REQUIRED in prod] One of these two must be set:` with:

```
# [REQUIRED in prod outside docker compose] One of these two must be set.
# Under docker compose, the entrypoint generates JWT_SECRET_KEY_FILE on first
# run when JWT_SECRET_KEY is empty.
```

Insert after line 109 (`FRONTEND_PORT=3000`):

```

# Host directory bind-mounted as the backend data volume (DB, assets, JWT secret).
# DATA_DIR=./data

# Host directory with device profile YAMLs (mounted read-only).
# PROFILE_DIR=./config/profiles
```

- [ ] **Step 6: Commit**

```bash
git add docker-compose.yml docker-compose.override.yml.example README.md .env.example
git commit -m "fix(compose): make fresh 'docker compose up -d' work

- drop the hard dependency on a pre-created external 'proxy' network;
  the override example shows how to opt back in
- forward JWT_SECRET_KEY into the backend container
- healthchecks: backend image has curl, not wget; frontend nginx listens
  on IPv4 only, so probe 127.0.0.1 instead of localhost (::1)
- document DATA_DIR / PROFILE_DIR and JWT secret auto-generation

Deployments that relied on the external proxy network must add the
override snippet.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Local start.sh + end-to-end verification

**Files:**
- Modify (local only, do NOT commit): `start.sh`

**Interfaces:**
- Consumes: Tasks 1-3.
- Produces: verification evidence only.

- [ ] **Step 1: Rewrite `start.sh`**

```bash
export FRONTEND_PORT=3001                           # Web interface port
export DATA_DIR=/home/artur/Documents/eink-data     # Host dir for backend data
export EINK_CLEANUP_TTL_DAYS=0                      # Auto-cleanup (0 to disable)
# JWT_SECRET_KEY: unset -> generated into $DATA_DIR/users/jwt_secret.txt on first run
docker compose up -d
```

- [ ] **Step 2: End-to-end on a throwaway project + scratch data dir**

```bash
V=$(mktemp -d)
DATA_DIR=$V FRONTEND_PORT=3001 docker compose -p einkverify up -d --build
for i in $(seq 1 18); do s=$(docker compose -p einkverify ps --format '{{.Health}}' | sort -u | tr '\n' ' '); echo "$s"; [ "$s" = "healthy " ] && break; sleep 5; done
docker compose -p einkverify logs backend | grep -cE 'no such table|Traceback|RuntimeError'
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:3001/
ls -l "$V/users/jwt_secret.txt"
```
Expected: final status line `healthy `; error count `0`; HTTP `200`; secret file present (`-rw-------`).

- [ ] **Step 3: Restart reuses state**

```bash
md5sum "$V/users/jwt_secret.txt"; docker compose -p einkverify restart backend; sleep 10
docker compose -p einkverify logs --tail 20 backend | grep -E 'Generated|stamping|Traceback'; md5sum "$V/users/jwt_secret.txt"
```
Expected: identical hashes; no `Generated`/`Traceback` lines after restart.

- [ ] **Step 4: Tear down**

```bash
docker compose -p einkverify down
docker run --rm -v "$V:/d" alpine rm -rf /d/*; rmdir "$V"
git status --short   # expect: only '?? start.sh' (and nothing else uncommitted)
```

- [ ] **Step 5: Run the self-check one final time**

Run: `eval "$CHECK"`
Expected: four `ok  migrate` lines + `ok  jwt secret resolution`.

No commit in this task. Merge/PR decision → `superpowers:finishing-a-development-branch`.
