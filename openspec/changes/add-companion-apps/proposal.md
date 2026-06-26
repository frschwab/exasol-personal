## Why

The `--with-ai-lab` capability proved a valuable pattern: co-locate a useful open-source tool in a rootless Podman container on the database host, pre-wire it to the Exasol database and BucketFS, and surface it with zero manual configuration. That pattern is not specific to the AI Lab — it generalizes to any web tool that benefits from a ready-made Exasol connection.

The most impactful next add-on is a **BI / data-exploration tool**, because the biggest friction for someone touching Exasol for the first time is "I have a database and sample data — now what?". A pre-connected, browser-based BI tool turns that into a guided, step-by-step experience: write a first query, build a first chart, save a first dashboard — against the sample data Exasol Personal already loads (`PRODUCTS` / `PRODUCT_REVIEWS`).

Rather than copy `installAiLab.sh` for each new tool, this change introduces a small **companion-app mechanism** so additional containers are declared as add-ons, and adds **Apache Superset** as the first BI add-on on top of it.

## What Changes

- Extract the cloud-agnostic mechanics currently hard-coded in `installAiLab.sh` into a reusable **companion-app contract**: a companion app declares its image, published port, persistence (systemd unit + lingering), firewall exposure (gated by `allowed_cidr`), generated secrets, a connection-seeding step, and the `exasol info` surface. AI Lab becomes the first consumer of this contract (no behavior change), and new add-ons reuse it.
- Add **Apache Superset** as an opt-in companion app via `--with-superset` (and `--superset-port`, default e.g. `8088`), mirroring `--with-ai-lab`.
- **Pre-seed Superset's Exasol database connection** using the `sqlalchemy-exasol` dialect (`exa+websocket://`), with certificate validation disabled for the self-signed deployment certificate — the same trust model as the AI Lab SCS seeding. Pre-create a Superset DB connection pointing at the deployment so the user lands in a working SQL Lab immediately.
- Restrict availability to infrastructure presets that declare support (cloud only, AWS first), exactly as AI Lab does. Local deployments are out of scope for the same reasons (and resource footprint).
- **Print connection info** for each installed companion app in `exasol info` (URL + secrets reference), shown only when that app is present.

## Capabilities

### New Capabilities
- `companion-apps`: a generic mechanism for optionally installing additional pre-configured containers (beyond AI Lab) alongside the database on supporting infrastructure — declaring image, port, firewall exposure, secrets, DB/BucketFS connection seeding, persistence, and connection-info output.
- `superset-access`: the Apache Superset BI add-on built on `companion-apps`, including its Exasol connection seeding and a newcomer-friendly default (sample-data dashboard / SQL Lab landing).

### Modified Capabilities
- `ai-lab-access`: re-expressed as the first consumer of `companion-apps`. No user-facing behavior change; the install mechanics move behind the shared contract.

## Impact

- **Installation (OS) preset** (`assets/installation/ubuntu/`): generalize the AI Lab post-install hook into a shared library plus per-app hooks (`installAiLab.sh`, `installSuperset.sh`) that call common helpers (Podman run + systemd unit + socket/registry setup where needed + connection seeding). Add the `sqlalchemy-exasol` dialect to Superset's image (thin custom image or install-at-start).
- **Infrastructure preset (AWS first)** (`assets/infrastructure/aws/`): add `with_superset` + `superset_port` variables, a security-group ingress rule gated by `allowed_cidr`, a generated Superset admin password (`random_password` → `secrets.json`), and registration of the Superset hook — the same three opt-in bits AI Lab uses.
- **CLI**: `--with-superset` / `--superset-port` flags auto-generated from the infra variables; `exasol info` extended to list companion-app URLs.
- **Deployment metadata**: optional `superset` connection object in `deployment.json`; `supersetAdminPassword` in `config.Secrets`.
- **External dependencies**: the Superset container image + `sqlalchemy-exasol`. No new Go dependencies.
- **Security**: one more exposed port and one more generated secret per enabled app; same `allowed_cidr` gating and `secrets.json` model as AI Lab.
- **Resource footprint**: each companion app adds memory/CPU. A single-node default instance comfortably hosts the DB + one or two add-ons; enabling many at once is not a goal. The CLI should make the footprint visible.

## Non-goals

- Not a plugin marketplace or arbitrary user-supplied containers — companion apps are a curated, in-tree set.
- Not local-deployment support (cloud only, as with AI Lab).
- Metabase and other BI tools are explicitly deferred: Metabase's Exasol connectivity depends on a less-certain third-party driver plugin; Superset's maintained SQLAlchemy dialect makes it the right first BI add-on (see design.md).
