# Design — Companion apps (generic add-on mechanism) + Apache Superset

## Goal

Make "run a pre-configured open-source tool in a container next to the database" a **repeatable add-on**, not a per-tool bespoke script — and ship **Apache Superset** as the first BI add-on so a first-time user can go from "I have sample data" to "I built a dashboard" with zero setup.

## Why Superset (and not Metabase/Redash/Grafana)

| Tool | Exasol connectivity | Stock image has driver? | Newcomer UX | Verdict |
|---|---|---|---|---|
| **Apache Superset** | `sqlalchemy-exasol` dialect (`exa+websocket://`), Exasol-maintained | No — add via thin image / pip | SQL Lab + charts + dashboards | ✅ **First add-on** |
| Metabase | third-party driver `.jar` (less certain/maintained) | No — drop-in plugin | Easiest (ask-a-question) | ⏸ deferred — verify driver first |
| Redash | SQLAlchemy query runner | partial | query+viz | ✗ project semi-dormant |
| Grafana | plugin, time-series focus | n/a | ops dashboards | ✗ not general exploration |

Superset wins on the three things that actually gate feasibility: **(1)** a maintained Exasol dialect exists, **(2)** it installs cleanly into the image, **(3)** connections can be seeded programmatically.

## The companion-app contract

Extract what `installAiLab.sh` already does into a shared contract. A companion app is defined by:

```
name           e.g. "ai-lab", "superset"
image          OCI image ref (docker.io/...)
container_port internal port
default_port   published host port (infra variable, → --<name>-port)
secrets[]      generated random_passwords surfaced into secrets.json
seed()         app-specific step that writes the Exasol DB/BucketFS connection
needs_docker   bool — whether to mount the Podman socket (AI Lab: yes; Superset: no)
info()         what `exasol info` prints (URL + which secrets.json keys)
```

Shared mechanics (today inline in `installAiLab.sh`, to become `companion_lib.sh`):
- rootless Podman run with `--restart`, retry-on-first-start, `allow_host_loopback` so the container reaches the DB/BucketFS via `host.containers.internal`
- `podman generate systemd` user unit + lingering (Ubuntu 22.04 / Podman 3.4.4 — no Quadlet), with the **stop-before-`enable --now`** handover fix
- `exasol info` rendering via the existing connection-instructions template

`needs_docker` captures the AI-Lab-only Podman-socket complexity (socket mount, `registries.conf`, `DockerRegistryImageChecker` patch, `SocketMode=0666`) so Superset — which does **not** need the Docker socket — doesn't carry any of it. That keeps the generalization honest: the gnarly bits stay opt-in per app.

## Superset specifics

- **Image**: stock `apache/superset` does not bundle the Exasol dialect. Two options:
  1. Thin derived image: `FROM apache/superset` + `pip install sqlalchemy-exasol` (preferred — reproducible, no network at start).
  2. Install at container start (simpler to ship, needs egress to PyPI at deploy time).
  Decide in implementation; (1) is cleaner for restricted-egress customers.
- **Connection seeding**: create a Superset database connection with SQLAlchemy URI
  `exa+websocket://sys:<pw>@host.containers.internal:8563/?ENCRYPTION=Yes&SSLCertificate=SSL_VERIFY_NONE`
  (cert validation disabled for the self-signed deployment cert — same trust model as AI Lab `cert_vld=false`). Seed via Superset's import API / `superset import-datasources`, run once at install.
- **Admin user**: generate a Superset admin password (`random_password` → `secrets.json` as `supersetAdminPassword`); create the admin via `superset fab create-admin` at first start.
- **Newcomer landing**: optionally pre-import a small starter asset — a SQL Lab query and/or a dashboard over `PRODUCTS`/`PRODUCT_REVIEWS` — so the first screen is immediately useful. Keep it minimal; the connection alone already unblocks SQL Lab.

## Surfacing

`exasol info` lists each installed companion app:
```
=== How to open Superset ===
  URL: http://<host>:8088
  Username: admin
  Password: <stored in secrets.json>
  The Exasol database connection is pre-configured.
```
Same template mechanism as the AI Lab block; gated on the `superset` metadata being present.

## Resource & footprint notes

- Each app adds memory/CPU. DB + AI Lab + Superset fit on the default single-node memory-optimized instance, but this is not "enable everything." The CLI should surface footprint (e.g. a note when stacking multiple `--with-*` flags).
- Companion apps are cloud-only (need the host + exposed ports + DB/BucketFS reachability). Local deployment is out of scope, as with AI Lab.

## Migration / sequencing

1. Refactor `installAiLab.sh` → `companion_lib.sh` + `installAiLab.sh` (consumer). Pure refactor; AI Lab behavior unchanged, covered by the existing AI Lab verification.
2. Add the Superset add-on (`installSuperset.sh`, infra variables, secrets, SG rule, `exasol info`).
3. Validate Superset end-to-end on AWS (the `add-ai-lab-notebook-tests` harness pattern could be reused to smoke-test "connection works + landing page reachable").

## Open questions

1. Derived Superset image (build + host where?) vs install-at-start — restricted-egress customers favor a prebuilt image; where would it be published?
2. How much starter content to pre-import (nothing / one SQL Lab query / a full sample dashboard)?
3. Should multiple companion apps share one published-port range and one SG rule, or one each? (One each is simpler and matches AI Lab.)
4. Is there appetite to expose companion apps on the **local** backend later (e.g. Superset locally even without BucketFS)? The contract should not preclude it.
