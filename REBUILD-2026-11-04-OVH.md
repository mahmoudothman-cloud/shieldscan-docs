# Rebuild record — OVHcloud VPS-2, 2026-11-04

Host `vps-295191ed` (144.217.27.177), Canada-East (Beauharnois), 4 vCore / 7.6 GB / 75 GB NVMe, Ubuntu 26.04.1, user `ubuntu`.
Source of truth followed: `shieldscan-docs/REBUILD-RUNBOOK.md`. Source host: Kamatera 185.247.117.235 (`shieldscan` SSH alias), copied live — NOT the Aug-25 tarball — because the repos there carried unpushed commits (api +1, engine +3, docs +3 vs origin/main).

## Deviations from REBUILD-RUNBOOK.md (record in DRIFT-LOG if kept)

| # | Runbook says | Done here | Why |
|---|---|---|---|
| 1 | API + worker in `tmux` sessions | systemd units `shieldscan-api.service`, `shieldscan-worker.service` (enabled, Restart=always) | survive reboot; tmux state lived on no host |
| 2 | API `--host 0.0.0.0` | `--host 127.0.0.1` (runbook itself recommends this) | :8000 never exposed |
| 3 | Go via snap channel 1.26/stable + hold | tarball 1.26.10 at /usr/local/go (relayed through the Mac — `dl.google.com` returns 404 to this OVH range) | same pin; no snapd auto-refresh risk at all |
| 4 | Python 3.13 from apt | uv-managed 3.13 (`uv python find 3.13`); Ubuntu 26.04 has no python3.13 package | pyproject `^3.13` |
| 5 | pipx tools on system python | semgrep 1.95.0 + checkov 3.2.340 on uv Python **3.12** (+ `setuptools<81` injected into semgrep); sslyze 6.1.0, wapiti3 3.2.4 on 3.13 | both refuse 3.13 (no pkg_resources / build failure); Kamatera ran them on 24.04's 3.12 |
| 6 | restore `postgres.sql.gz` (Aug 25) | fresh `pg_dump -Fc` taken today, `pg_restore --exit-on-error`; role `shieldscan_app` created first per §5.1 | newer data: 1 org, 48 scans, 1007 vulns |
| 7 | Qdrant regenerates | restored Kamatera `qdrantdata` volume as well | free, avoids cold re-embed |
| 8 | ufw only | ufw + fail2ban + unattended-upgrades + 2 GB swap + `docker-compose.override.yml` memory limits (PG 1.8 G / Redis 512 M / Qdrant 600 M) | single 8 GB box |
| 9 | — | apt: libpango/libcairo/gdk-pixbuf + fonts-noto (WeasyPrint import crashed the API without them) | missing from runbook §3.1 |

Memory-limit override is **untracked** (`shieldscan-api/docker-compose.override.yml`); compose picks it up automatically.

## Verification (runbook §8)

- `\du`: `shieldscan` superuser, `shieldscan_app` NOINHERIT with LOGIN — 13 `tenant_isolation` policies, 84 grants to app role
- `SELECT count(*) FROM scans` with no tenant GUC → **0**
- alembic `current` = `heads` = `e4b8c2a71f36`
- `scripts/check-toolchain.sh` → ok go.mod 1.26.10 / go 1.26.10 / golangci-lint 2.11.4
- worker startup: 9 native binaries verified, 14 engines registered; Redis `processing`/`queue` empty, 1 live worker; backfill → "No stranded jobs"
- `SHIELDSCAN_INFRA_CIDRS=144.217.27.177/32`
- Scan `e5f78eab` (`full_web`, Juice Shop): recon ✓ nmap 3 · sslyze 0 · wapiti 2 · zap 607 · nuclei 27 · **nikto FAILED (15 m timeout)** → 639 raw → 31 vulnerabilities, 31 real fixes / 0 fallbacks, AI errors 0, spend $0.125 (8 fix-gen @ $0.0135, 1 summary $0.016, 7 embeddings). Scan status `PARTIAL`, `ai_pipeline_degraded=false`.

Nikto timeout cause: target `shieldscan.odyssey-eg.com` still resolves to Kamatera (Europe); from Canada each of ~7 000 TLS probes pays the RTT. On Kamatera the same hostname was the box itself. Clears when DNS moves here (then Caddy + juice-shop on this box serve it locally). If it recurs, raise the nikto timeout in the engine rather than editing nikto.conf.

## Toolchain on this box

nuclei v3.11.0 · subfinder v2.14.0 · httpx v1.10.0 · gitleaks 8.30.1 · semgrep 1.95.0 · checkov 3.2.340 · sslyze 6.1.0 · wapiti 3.2.4 · nikto 2.5.0 (source, `~/.local/nikto-2.5.0`) · dependency-check 10.0.4 · corstest · nuclei-templates 86 MB (copied) · Docker images pulled by digest from Go source · Caddy 2.11 · Poetry 2.5.1 · Go 1.26.10 (`/usr/local/go-1.27.3` kept aside) · Node 26.13 · Docker 29.9.3 / Compose 5.7.0.

Known, pre-existing: qdrant-client 1.17.1 vs server v1.11.5 warning (same on Kamatera).

## Still to do

- [x] DNS: `shieldscan.odyssey-eg.com` A → 144.217.27.177 (done 2026-11-05; nikto local)
- [x] Re-run a `full_web` after DNS; 7/7 completed (394 raw → 40 unique)
- [x] Push the three repos (deploy key on this box, GitHub `mahmoudothman-cloud`) — Kamatera cancel still owner-side after ~Nov 11
- [x] Commit this file + DRIFT-LOG entry in `shieldscan-docs`
- [ ] `DEPCHECK_NVD_API_KEY` still unset (runbook §3.2 note) — depcheck produces nothing until it is
- [x] Residency line in docs (CONSTITUTION §17.3 note); DPA text still owner-side — re-check OVH EU stock before public beta
- [ ] Demo login reset for this test: `~/.demo-login` (chmod 600) on the box

## Added since (M11, 2026-11-07/08)

- `app.shieldscan.odyssey-eg.com` A → 144.217.27.177; Caddy site for the React app with `/v1` proxied (OPERATIONS-RUNBOOK §5.5).
- `.env`: `ENVIRONMENT=production`, `APP_BASE_URL=https://app.shieldscan.odyssey-eg.com`.
- Alembic head `b7c3e9d1a2f4` (public_scans + vulnerability identity); `scripts/backfill_vulnerability_identity.py` run once (1116 rows → 68 open).
- Worker binary rebuilt from engine `622412c` (nuclei `-tags`); `bin/worker` replaced with the service stopped (text-file-busy otherwise).
- Public quick scans run in system org `shieldscan-public` (no members); their project rows live there.

## Operate

```
systemctl status shieldscan-api shieldscan-worker caddy
journalctl -u shieldscan-worker -f
cd ~/shieldscan/shieldscan-api && sudo docker compose -f docker-compose.dev.yml ps
~/shieldscan/stack/retired/          # bootstrap's own compose, not used
```
