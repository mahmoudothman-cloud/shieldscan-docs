# REBUILD-RUNBOOK.md

**Rebuilding ShieldScan on a fresh host from GitHub + the backup tarball.**

Written 2026-08-20 by reading the live box (`185.247.117.235`, Ubuntu 24.04.4)
and the three repos, not from memory. Every version, path, command and image
digest below was observed or read out of the code on the day of writing; where a
document disagrees with what the code does, this file records what the code
does and says so.

> **Do not cross-reference OPERATIONS-RUNBOOK.md while following this.** §4.2–4.4
> there describe a fleet that does not exist: systemd units (`shieldscan-worker`),
> a worker `/health` endpoint on :9100, and Redis keys of the shape
> `worker:worker-id-xxx` (the real key is `shieldscan:workers:{id}`). None of it
> is wired. **VERSIONS.md §2.5 has drifted too** — see §3.3. This file supersedes
> both for rebuild purposes.

---

## 0. What this rebuild actually is

One host running everything: three datastores in Docker, a FastAPI process, a Go
worker, and Caddy. No orchestration, no systemd. **The API and the worker run in
`tmux` sessions**, which is the single most important piece of host state that
exists nowhere in any repo or backup.

| Component | How it runs | Survives the box? |
|---|---|---|
| PostgreSQL 18.3 | Docker, `pgdata` volume | via `postgres.sql.gz` |
| Redis 8.6.2 | Docker, **no volume** | no — and doesn't need to |
| Qdrant v1.11.5 | Docker, `qdrantdata` volume | no — regenerable |
| API (uvicorn) | `tmux` session `api` | command in §6.1 |
| Worker (Go) | `tmux` session `worker` | command in §6.2 |
| Caddy | apt package + systemd | Caddyfile in tarball |

---

## 1. Inventory: what you have, and what you don't

**In the tarball** (`~/ss-backup/`, verified present on the old box):

| File | Contents | Notes |
|---|---|---|
| `api.env` | 18 keys | includes `APP_DB_PASSWORD` and `MIGRATION_DATABASE_URL` — both load-bearing, neither in `.env.example` |
| `engine.env` | 10 keys | includes `SHIELDSCAN_INFRA_CIDRS` — **must change**, see §5.2 |
| `postgres.sql.gz` | PG18.3 `pg_dump --clean` | 2.4 MB gzipped; `alembic_version = e4b8c2a71f36` |
| `caddyfile.txt` | 3 lines | see §7 — the target is not what you'd assume |
| `compose.yml` | copy of `docker-compose.dev.yml` | the repo copy is authoritative |
| `pg-roles.txt` | `\du` output | two roles; see §5.1 |

**On GitHub** (`mahmoudothman-cloud`): `shieldscan-api`, `shieldscan-engine`,
`shieldscan-docs`.

**Host-only, in neither** — this is the list you asked me to find:

| Item | Recreate by | Cost |
|---|---|---|
| **tmux launch commands** for API + worker | §6, verbatim | — |
| **ufw rules** | §7.3 | — |
| **Caddy ACME/TLS state** (`/var/lib/caddy`) | automatic once DNS points at the new IP | minutes |
| **`~/nuclei-templates`** (83 MB) | refetch, or copy — §2 | template set changes over time, so a refetch can change findings |
| **`~/.local/bin` tool binaries** (11) + `~/.local/dependency-check` (37 MB) | **copy — §2.** No provisioning script exists; three of the eleven cannot be pinned back to what ran | ~20 min to rebuild, seconds to copy |
| **pipx venvs** (checkov, semgrep, sslyze, wapiti3) | reinstall — §3.2, all four are exact pins | ~5 min |
| **Python 3.13.13** (`/usr/bin/python3.13`) | §3.1 — system `python3` is 3.12.3, and `pyproject` requires `^3.13` | — |
| **Docker images** (12) | pulled on demand by digest — §3.3 | ~5 min |
| **Qdrant collections** | regenerated on the next scan's embed stage | — |
| **`shieldscan-engine/bin/worker`** | `go build` — §4.3 | seconds |
| **`.claude/settings.local.json`** (36 KB permission allowlist) | copy it off the box now, or rebuild it prompt-by-prompt | worth saving |
| `/tmp/api.log`, `/tmp/worker.log` | not worth keeping | — |
| `~/workspace/internal/tools/docker/nmap/` | empty stray dirs from a mistaken path; **do not recreate** | — |

**External services** (unaffected by the rebuild, no action): Anthropic, OpenAI,
Stripe, Cloudflare R2. Keys are in `api.env` / `engine.env`.

---

## 2. Before the old box dies

```bash
# 1. The permission allowlist — 36 KB you do not want to retype.
cp ~/workspace/.claude/settings.local.json ~/ss-backup/

# 2. The scanner toolchain. NOT optional — there is no provisioning script
#    (see §3.2), and several tools cannot be reinstalled at the versions that
#    were actually running. ~140 MB total; scp it.
tar czf ~/ss-backup/local-toolchain.tgz -C ~ \
  .local/bin .local/dependency-check nuclei-templates
#    Restore on the new box with:  tar xzf local-toolchain.tgz -C ~
#    Then confirm PATH: ~/.local/bin must be on the PATH of whatever starts
#    the worker, or buildRegistry fails and the worker exits 1.
#
#    pipx venvs (~644 MB, under ~/.local/share/pipx) are deliberately NOT in
#    that tarball: all four are exact-version pins and reinstall cleanly.

# 3. Re-verify the dump is readable, not just present.
zcat ~/ss-backup/postgres.sql.gz | tail -5
zcat ~/ss-backup/postgres.sql.gz | grep -c shieldscan_app        # expect 24

# 4. Nothing uncommitted, nothing unpushed — GitHub is the only copy of the code.
for r in shieldscan-api shieldscan-engine shieldscan-docs; do
  echo "== $r"
  git -C ~/workspace/$r status --porcelain
  echo "  HEAD $(git -C ~/workspace/$r rev-parse --short HEAD) / origin/main $(git -C ~/workspace/$r rev-parse --short origin/main)"
done
```

Step 4 is the one with no recovery. As of writing, all three repos are pushed and
`HEAD == origin/main` (api `159a9e2`, engine `1d0086b`, docs `a4d52f5`) — but
there are **uncommitted working-tree changes that exist only on this box**:

- `shieldscan-api`: `pyproject.toml`, `poetry.lock` (the anthropic 0.40.0 → 0.125.0 bump)
- `shieldscan-docs`: `VERSIONS.md`, `DRIFT-LOG.md`, and this file

Commit and push those before the box goes, or the SDK bump has to be redone and
this runbook does not exist on the new host.

---

## 3. Fresh host: packages

### 3.1 Base

```bash
sudo apt update && sudo apt install -y \
  git curl tmux ufw ca-certificates \
  python3.13 python3.13-venv python3.13-dev \
  nikto pipx

# Docker (official repo)
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"    # log out and back in

# Caddy (official repo)
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' \
  | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' \
  | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install -y caddy

# Go — the snap is what the old box used; go.mod requires >= 1.26.2
sudo snap install go --classic

# Poetry
pipx install poetry
pipx ensurepath
```

Observed on the old box: Go 1.26.7, Poetry 2.3.4, Caddy v2.11.4, Python 3.13.13.

### 3.2 The eleven scanner binaries

`cmd/worker/registry_wiring.go` resolves each tool by
`SHIELDSCAN_<TOOL>_BINARY` first, then `$PATH` (`resolveBinary`, Pattern 2). No
`SHIELDSCAN_*_BINARY` overrides are set on the old box, so **every tool is found
on `$PATH`** — which means `~/.local/bin` must be on the `PATH` of whatever
launches the worker.

Nine are registered as `ToolRunner`s (`checkov`, `corstest`, `depcheck`,
`gitleaks`, `nikto`, `nuclei`, `semgrep`, `sslyze`, `wapiti`); `subfinder` and
`httpx` are recon helpers invoked directly (ADR-022), not registered.

**Startup is fail-fast**: `buildRegistry` returns an error if any binary is
missing, and `runMain` exits 1. A missing tool means the worker does not start —
you will not get a partial fleet.

> **There is no provisioning script, and there never was.** The only shell script
> in any repo is `shieldscan-api/scripts/verify-versions.sh`, which *checks*
> versions against PyPI and GitHub and installs nothing. The block below is the
> first time these steps have been written down — it was reconstructed by reading
> the old box, not by transcribing an existing script. Treat it as a starting
> point that has never been executed end to end.
>
> **Because of that, prefer copying `~/.local` over reinstalling.** See §2.

```bash
# Go-installed — versions pinned to what was actually running, NOT @latest.
# @latest would drift to whatever ships that week and change your findings.
go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@v3.11.0
go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@v2.14.0
go install github.com/projectdiscovery/httpx/cmd/httpx@v1.10.0
go install github.com/gitleaks/gitleaks/v8@v8.30.1
cp ~/go/bin/{nuclei,subfinder,httpx,gitleaks} ~/.local/bin/

# pipx-managed — exact pins, reproducible
pipx install checkov==3.2.340
pipx install semgrep==1.95.0
pipx install sslyze==6.1.0
pipx install wapiti3==3.2.4

# CORStest — a single Python script, no release tags. UNPINNABLE by version;
# the old box has whatever HEAD was on 2026-08-01. Copy the file rather than
# re-cloning if you want the same behaviour.
git clone https://github.com/RUB-NDS/CORStest.git /tmp/CORStest
install -m 0755 /tmp/CORStest/corstest.py ~/.local/bin/corstest

# OWASP Dependency-Check 10.0.4 — release archive unpacked to
# ~/.local/dependency-check (37 MB) with ~/.local/bin/dependency-check.sh a
# symlink into its bin/. Needs a JDK on PATH.
#   NOTE: the old box has NO data/ directory under the install, i.e. the NVD
#   database was never populated, and DEPCHECK_NVD_API_KEY is not in engine.env.
#   depcheck is installed but has almost certainly never produced real findings.
#   It only appears in FULL_WEB_SOURCE and FULL_SPECTRUM, so the §8.2 FULL_WEB
#   check will not surface this. Fix it on the new box or accept it knowingly.

# nikto comes from apt (/usr/bin/nikto) — version tracks the Ubuntu release,
# so a different Ubuntu gives a different nikto. Old box: Ubuntu 24.04.4.

nuclei -update-templates        # writes ~/nuclei-templates (83 MB)
```

**Versions observed on the old box** (what was actually working):

| Tool | Installed | VERSIONS.md §2.5 pin |
|---|---|---|
| nuclei | **v3.11.0** | v3.7.1 |
| subfinder | **v2.14.0** | v2.6.7 |
| httpx | **v1.10.0** | v1.6.10 |
| gitleaks | **8.30.1** | v8.21.2 |
| semgrep | 1.95.0 | 1.95.0 ✓ |
| checkov | 3.2.340 | 3.2.340 ✓ |
| sslyze | 6.1.0 | 6.1.0 ✓ |
| wapiti | 3.2.4 | 3.2.4 ✓ |
| dependency-check | **10.0.4** | 9.2.0 |

Six of nine match; the projectdiscovery tools and gitleaks are all ahead of the
pin. Install what was working, not what the table says.

### 3.3 Docker images — do not pre-pull from VERSIONS.md

The engine pins every image **by digest, in Go source**, which is authoritative
and rebuild-safe. Nothing needs pulling by hand; the worker pulls exactly these:

| Tool | Constant | Reference |
|---|---|---|
| nmap | `internal/tools/docker/nmap/nmap.go:18` | `instrumentisto/nmap:7.94@sha256:59e2c0bb…` |
| trivy | `internal/tools/docker/trivy/trivy.go:37` | `aquasec/trivy:0.70.0@sha256:be1190af…` |
| sqlmap | `internal/tools/docker/sqlmap/sqlmap.go:23` | `parrotsec/sqlmap:latest@sha256:740197a8…` |
| zap | `internal/tools/docker/service/zap/zap.go:42` | `ghcr.io/zaproxy/zaproxy@sha256:8770b23f…` |
| mobsf | `internal/tools/docker/service/mobsf/mobsf.go:19` | `opensecurity/mobile-security-framework-mobsf@sha256:72311e35…` |

VERSIONS.md §2.5 names `aquasec/trivy:0.58.0`, `paoloo/sqlmap:1.9`,
`zaproxy/zap-stable:2.16.0` and `instrumentisto/nmap:7.95` — **none of which is
what runs.** `deploy/docker-compose.services.yml` is also stale (trivy 0.58.0)
and is a dev/CI artefact; the production path spins containers per scan from the
digests above and does not read that compose file.

---

## 4. Repos, datastores, secrets

### 4.1 Clone

```bash
mkdir -p ~/workspace && cd ~/workspace
for r in shieldscan-api shieldscan-engine shieldscan-docs; do
  git clone git@github.com:mahmoudothman-cloud/$r.git
done
cp ~/ss-backup/api.env    ~/workspace/shieldscan-api/.env
cp ~/ss-backup/engine.env ~/workspace/shieldscan-engine/.env
chmod 600 ~/workspace/shieldscan-api/.env ~/workspace/shieldscan-engine/.env
```

### 4.2 Datastores

`docker-compose.dev.yml` in `shieldscan-api` is authoritative — use it, not the
tarball's `compose.yml` copy. It reads `POSTGRES_PASSWORD` and `REDIS_PASSWORD`
from `.env` with the `:?` form, so a missing value fails loudly.

```bash
cd ~/workspace/shieldscan-api
docker compose -f docker-compose.dev.yml up -d
docker compose -f docker-compose.dev.yml ps    # postgres, redis, qdrant
```

Two things the compose file's own comments make explicit and are easy to trip on:

- The pgdata volume mounts `/var/lib/postgresql` (the **parent**), not
  `/var/lib/postgresql/data`. PG18 refuses to start with the latter.
- `POSTGRES_PASSWORD` applies **only at first init** of the volume. On a fresh
  box that is exactly what happens, so the value in `.env` becomes the real
  password — good. Do not `docker compose up` against a restored volume expecting
  the password to change.

### 4.3 Build the worker

```bash
cd ~/workspace/shieldscan-engine
go mod download
go build -o bin/worker ./cmd/worker/
```

### 4.4 Install API deps

```bash
cd ~/workspace/shieldscan-api
poetry env use python3.13
poetry install
```

---

## 5. The two traps

### 5.1 Create `shieldscan_app` BEFORE restoring the dump

**`pg_dump` does not carry roles — they are cluster-level.** This is not
theoretical; here is what the dump actually contains, counted:

```
shieldscan_app            24 references
CREATE ROLE                0
CREATE POLICY             13
FORCE ROW LEVEL SECURITY  13
OWNER TO                  32
```

Twenty-four `GRANT`/`ALTER DEFAULT PRIVILEGES` statements naming a role the dump
never creates. The four distinct shapes:

```sql
GRANT USAGE ON SCHEMA public TO shieldscan_app;
GRANT SELECT,INSERT,DELETE,UPDATE ON TABLE public.<each table> TO shieldscan_app;
ALTER DEFAULT PRIVILEGES FOR ROLE shieldscan IN SCHEMA public
  GRANT SELECT,INSERT,DELETE,UPDATE ON TABLES TO shieldscan_app;
ALTER DEFAULT PRIVILEGES FOR ROLE shieldscan IN SCHEMA public
  GRANT SELECT,USAGE ON SEQUENCES TO shieldscan_app;
```

**Why it fails silently rather than loudly.** `psql` without `ON_ERROR_STOP`
prints `ERROR: role "shieldscan_app" does not exist`, *keeps going*, and finishes
with a schema that looks complete. The 13 RLS policies land (they reference a
GUC, not the role), the data lands, and the API then connects as a role with no
grants — every query returns a permission error, or worse, the policies apply to
a role that was never granted anything and you spend a day on it.

**And re-running Alembic will not fix it.** The dump carries
`alembic_version = e4b8c2a71f36`, which is the current head in the repo
(`poetry run alembic heads` → `e4b8c2a71f36`), so `alembic upgrade head` is a
no-op. Migration `5b30069416cd` (which creates the role) and `c3e7a15d8b92`
(which grants it `LOGIN`) will never re-run.

**Do this, in this order:**

```bash
cd ~/workspace/shieldscan-api
set -a && source .env && set +a
PGC="docker exec -i shieldscan-api-postgres-1 psql -v ON_ERROR_STOP=1 -U shieldscan -d shieldscan"

# 1. Create the role, exactly as app_role_upgrade_sql() does (src/app/db/policies.py:182).
$PGC <<'SQL'
DO $$ BEGIN
  IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'shieldscan_app')
  THEN CREATE ROLE shieldscan_app NOSUPERUSER NOBYPASSRLS NOINHERIT;
  END IF;
END $$;
SQL

# 2. LOGIN — added separately by migration c3e7a15d8b92, NOT by the create above.
$PGC -c "ALTER ROLE shieldscan_app WITH LOGIN;"

# 3. Password — set out of band so no secret enters git. Must match the one
#    embedded in DATABASE_URL in .env.
$PGC -c "ALTER ROLE shieldscan_app WITH PASSWORD '${APP_DB_PASSWORD}';"

# 4. Verify BEFORE restoring.
$PGC -c "\du"
```

Expect exactly this (matches `pg-roles.txt`):

```
   Role name    |                         Attributes
----------------+------------------------------------------------------------
 shieldscan     | Superuser, Create role, Create DB, Replication, Bypass RLS
 shieldscan_app | No inheritance
```

`shieldscan_app` showing **no** "Cannot login" is the check — `\du` lists that
attribute only when the role lacks `LOGIN`. `shieldscan` is created by the
compose file's `POSTGRES_USER`, already superuser, and owns all 32 objects.

The three attributes are each load-bearing:

- `NOSUPERUSER` and `NOBYPASSRLS` — `FORCE ROW LEVEL SECURITY` constrains the
  table owner but **not** a superuser or a `BYPASSRLS` role. Grant either and
  every tenant policy silently stops applying while every test still passes.
- `NOINHERIT` — keeps the role from picking up privileges via membership.

### 5.2 `SHIELDSCAN_INFRA_CIDRS` is this box's address

`engine.env` carries `SHIELDSCAN_INFRA_CIDRS=185.247.117.235/32`. Copy it
forward unchanged and the guard protects the *old, terminated* host and nothing
at all.

```bash
NEW_IP=$(curl -4 -s ifconfig.me)
sed -i "s|^SHIELDSCAN_INFRA_CIDRS=.*|SHIELDSCAN_INFRA_CIDRS=${NEW_IP}/32|" \
  ~/workspace/shieldscan-engine/.env
grep SHIELDSCAN_INFRA_CIDRS ~/workspace/shieldscan-engine/.env
```

Keep it `/32`. The interface's `/24` is the provider's subnet — the other ~253
addresses belong to unrelated customers, some of whom may be legitimate targets,
and blocking them is both wrong and invisible.

Know what this actually buys, because it is less than the name suggests
(`.env.example` spells this out, and it is accurate): it blocks **IP-literal**
nmap targets inside the listed ranges and nothing else. `validateTarget`
classifies IP literals only — a *hostname* returns early, so a target that
resolves into the range passes straight through, which is how the self-scans
work. It applies to nmap alone. Unset, it is a no-op that always returns false.
Set it correctly anyway; it is one line and the failure mode is silent.

**This is the only value in either `.env` that is host-specific.** A repo-wide
grep for the old IP finds exactly one other occurrence — a log-message fixture in
`tests/services/test_completions_consumer_logging.py:334` — which is cosmetic and
needs no change.

---

## 6. Restore, then run

```bash
cd ~/workspace/shieldscan-api
zcat ~/ss-backup/postgres.sql.gz \
  | docker exec -i shieldscan-api-postgres-1 \
      psql -v ON_ERROR_STOP=1 -U shieldscan -d shieldscan
```

`ON_ERROR_STOP=1` is the whole point — it turns §5.1's silent failure into an
immediate stop.

Restore with the **PG18 client**, i.e. `docker exec` into the `postgres:18.3`
container as above. The dump begins with a `\restrict` token (a PG18 pg_dump
feature); an older host `psql` will choke on it.

Then confirm the schema is at head and no migration is pending:

```bash
poetry run alembic current      # expect e4b8c2a71f36
poetry run alembic heads        # expect e4b8c2a71f36 (head)
```

### 6.1 API

```bash
tmux new -d -s api \
  'cd ~/workspace/shieldscan-api && poetry run uvicorn app.main:app --host 0.0.0.0 --port 8000 2>&1 | tee -a /tmp/api.log'
```

Verbatim from the old box. Two notes:

- `--host 0.0.0.0` binds every interface. Port 8000 is not in the ufw ruleset
  (§7.3), which is the only thing keeping it off the internet. Consider
  `--host 127.0.0.1`; nothing external needs :8000 today.
- Uvicorn configures only its own loggers. `app/logging_config.py` installs the
  root handler at lifespan start, so `LOG_LEVEL` in `.env` governs the
  `job_completed` lines the verification in §8 depends on. `LOG_LEVEL=WARNING`
  hides them.

### 6.2 Worker

```bash
tmux new -d -s worker \
  'cd ~/workspace/shieldscan-engine && set -a && source .env && set +a && ./bin/worker 2>&1 | tee -a /tmp/worker.log'
```

Verbatim, and each part earns its place:

- `set -a && source .env && set +a` — the worker reads its config from the
  environment. It does not load `.env` itself.
- `tee -a`, **not** `tee`. Without `-a` every restart truncates the log, which
  once destroyed the evidence for a shutdown bug (Drift #69).
- The `| tee` pipe is why the SIGPIPE guard exists: if the reader dies, an
  unguarded worker is killed by SIGPIPE mid-shutdown and leaks its warm-pool
  containers. `installSIGPIPEGuard()` in `cmd/worker/main.go` handles it. If you
  replace this launcher with systemd, that guard stays correct but the failure it
  prevents disappears.

`~/.local/bin` must be on `PATH` inside the tmux environment or `buildRegistry`
fails and the worker exits 1 at startup.

---

## 7. Edge: Caddy, DNS, firewall

### 7.1 The Caddyfile fronts juice-shop, not the API — and that is intentional

```
shieldscan.odyssey-eg.com {
	reverse_proxy localhost:3000
}
```

> **This is correct. Do not "fix" it during the rebuild.**
>
> Port 3000 is **juice-shop**, the deliberately-vulnerable scan target. The
> ShieldScan API is on `:8000` and is deliberately not proxied. Anyone reading
> this fresh will assume the product belongs behind the product's hostname; it
> does not, and repointing `reverse_proxy` at `localhost:8000` breaks the scan
> target and exposes the API in one move.

The domain exists to give the scanner a **public HTTPS target with a real
certificate** — self-signed or plain-HTTP targets do not exercise the same code
paths, and `sslyze` needs a genuine chain to say anything useful. The engine's
own tests encode this: `internal/tools/recon/recon_test.go` uses
`https://shieldscan.odyssey-eg.com` and expects the page title "Juice Shop".

Verified on the old box against `docker ps` (`juice-shop` → `127.0.0.1:3000`)
and `ss -tlnp` (uvicorn on `0.0.0.0:8000`, unproxied).

Restore the Caddyfile as-is, and start juice-shop:

```bash
sudo cp ~/ss-backup/caddyfile.txt /etc/caddy/Caddyfile
docker run -d --name juice-shop --restart unless-stopped \
  -p 127.0.0.1:3000:3000 bkimminich/juice-shop
sudo systemctl restart caddy
```

### 7.2 DNS

Point `shieldscan.odyssey-eg.com` at the new IP **before** starting Caddy, or the
ACME HTTP-01 challenge fails and you get no certificate. Certificates live in
`/var/lib/caddy` and regenerate automatically once DNS resolves.

### 7.3 ufw

Exactly what the old box had — nothing else was open:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
```

5432 / 6379 / 6333 need no rule — compose binds them to `127.0.0.1`.

---

## 8. Verification

Services starting proves nothing. These are the checks that prove it works.

### 8.1 The app connects as `shieldscan_app`, not the owner

The §5.1 trap's detector. If this shows `shieldscan`, RLS is not being enforced
on any query the API makes, and every test will still pass.

```bash
docker exec -i shieldscan-api-postgres-1 psql -U shieldscan -d shieldscan -c \
  "SELECT usename, count(*) FROM pg_stat_activity
    WHERE datname='shieldscan' GROUP BY usename ORDER BY 2 DESC;"
```

Expect `shieldscan_app` with the API's pool connections. `shieldscan` appearing
alone means `DATABASE_URL` is still pointing at the owner.

Prove the policies actually bite, rather than merely existing:

```bash
docker exec -i shieldscan-api-postgres-1 psql -U shieldscan -d shieldscan -c \
  "SELECT count(*) FROM pg_policies WHERE policyname='tenant_isolation';"
# expect 13

cd ~/workspace/shieldscan-api && poetry run python -c "
import asyncio
from sqlalchemy import text
from app.db import AsyncSessionLocal
async def main():
    async with AsyncSessionLocal() as db:
        n = await db.scalar(text('SELECT count(*) FROM scans'))
        print('scans visible with NO tenant GUC set:', n, '(expect 0)')
asyncio.run(main())"
```

Zero is correct and is the proof: the policy predicate is null-safe, so an unset
GUC matches nothing rather than everything.

### 8.2 A real scan: seven `job_completed` lines, counts agreeing

A `FULL_WEB` scan dispatches one phase-1 `recon` job plus six phase-2 tools —
`nuclei, zap, wapiti, nikto, nmap, sslyze` (`SCAN_TYPE_TOOLS` in
`services/orchestrator.py:109`). **Seven terminal events.**

Run a scan against `https://shieldscan.odyssey-eg.com`, then:

```bash
grep job_completed /tmp/api.log | tail -7
```

Check three things on those lines:

1. **Seven of them.** Six means a tool failed to dispatch or a job was stranded.
2. **`finding_count` equals `persisted_count` on every line.** Divergence is
   silent finding loss between what the engine reported and what survived
   validation into `raw_findings` — the condition that hid nmap producing zero
   findings for fourteen consecutive runs while every status read "completed".
3. **`status=completed`**, not `failed`. A `failed` line now carries
   `error_message`; read it.

```bash
# The same thing from the database, which is the authority:
docker exec -i shieldscan-api-postgres-1 psql -U shieldscan -d shieldscan -c \
  "SELECT engine, status, finding_count, error_message
     FROM scan_jobs WHERE scan_id = '<scan-uuid>' ORDER BY engine;"
```

### 8.3 The AI pipeline ran for real, not on fallbacks

```bash
docker exec -i shieldscan-api-postgres-1 psql -U shieldscan -d shieldscan -c \
  "SELECT count(*) FILTER (WHERE ai_fix_is_fallback) AS fallbacks,
          count(*) FILTER (WHERE NOT ai_fix_is_fallback) AS real_fixes
     FROM vulnerabilities WHERE scan_id = '<scan-uuid>';"
```

`fallbacks = 0`. Anything else means budget exhaustion or a provider failure, and
the scan's `ai_pipeline_degraded` flag will be set. Cross-check the spend:

```bash
docker exec -i shieldscan-api-postgres-1 psql -U shieldscan -d shieldscan -c \
  "SELECT operation_type, count(*), sum(cost_usd),
          count(*) FILTER (WHERE error_type IS NOT NULL) AS errors
     FROM ai_api_calls WHERE scan_id = '<scan-uuid>' GROUP BY 1;"
```

Reference figures measured on the old box: **$0.0138 per fix**, **$0.0148 per
summary**, embeddings negligible. `errors = 0`. A row with `tokens_in = 0` and
`cost_usd = 0` is a failure or a budget denial, not a cheap call.

### 8.4 Redis is clean after the scan

Processing lists hold jobs a worker has checked out but not yet acknowledged. A
non-empty list after a scan finishes means a job was lost or an acknowledgement
failed.

```bash
cd ~/workspace/shieldscan-api && set -a && source .env && set +a
RC="docker exec -i shieldscan-api-redis-1 redis-cli -a ${REDIS_PASSWORD} --no-auth-warning"

$RC --scan --pattern 'shieldscan:processing:*'   # expect NOTHING
$RC --scan --pattern 'shieldscan:queue:*'        # expect NOTHING
$RC --scan --pattern 'shieldscan:workers:*'      # expect exactly one live worker
```

An empty `shieldscan:processing:*` is the reliable-queue invariant holding. If a
key lingers, `LLEN` it — a live worker sweeps a dead worker's lists every 60s, so
anything still there after two minutes wants investigating.

The complementary check, from the other side:

```bash
cd ~/workspace/shieldscan-api
poetry run python scripts/backfill_stranded_jobs.py     # dry run, writes nothing
```

On a healthy rebuild this prints `No stranded jobs.` If it lists jobs, the
restored database carried non-terminal rows across from the old box — expected if
you never ran the backfill there, and safe to `--apply`.

### 8.5 Both suites green on the new host

```bash
cd ~/workspace/shieldscan-api    && poetry run pytest -q       # 913 passed, 1 skipped
cd ~/workspace/shieldscan-engine && go test -race -count=1 ./... # 27 packages ok
```

The API suite needs `APP_DB_PASSWORD` in `.env` — `tests/conftest.py:131` fails
loudly without it, because the suite connects as the app role rather than
simulating it. Note both repos' lint gates (`ruff check src tests`, `mypy src`)
are red independently of the rebuild; do not treat that as a regression.

### 8.6 The infra guard points at this box

```bash
grep SHIELDSCAN_INFRA_CIDRS ~/workspace/shieldscan-engine/.env
curl -4 -s ifconfig.me
```

The two must agree. Nothing in the running system will tell you if they don't.

---

## 9. Things that will look broken and aren't

| Symptom | Cause |
|---|---|
| Qdrant collections missing | Recreated on the first scan's embed stage. Expected. |
| First scan slower than usual | Docker pulling five images by digest, and nuclei templates cold. |
| `docker ps` shows a stray `instrumentisto/nmap` container | Warm-pool container. Reaped at the next worker start once its owner's 60s heartbeat expires. |
| Redis empty after a compose restart | The redis service has **no volume**, by design. Progress streams and heartbeats are ephemeral. |
| `shieldscan.odyssey-eg.com` serves Juice Shop | Correct — §7.1. |
| Nine of the nineteen "security tools" absent | Only nine are registered as ToolRunners; the rest are Docker-backed or recon helpers. |
