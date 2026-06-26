## 1. Companion-app mechanism (refactor)

- [ ] 1.1 Extract shared install mechanics from `installAiLab.sh` into `companion_lib.sh` (rootless Podman run + retry, `podman generate systemd` user unit + lingering, the stop-before-`enable --now` handover, `allow_host_loopback`)
- [ ] 1.2 Define the companion-app contract (name, image, ports, secrets, `seed()`, `needs_docker`, `info()`)
- [ ] 1.3 Reframe `installAiLab.sh` as the first consumer of `companion_lib.sh`; keep `needs_docker=true` mechanics (socket mount, registries.conf, checker patch, SocketMode) scoped to AI Lab only
- [ ] 1.4 Verify AI Lab behavior is unchanged (re-run the existing AI Lab verification end-to-end)

## 2. Superset add-on — installation preset

- [ ] 2.1 `installSuperset.sh` consumer: run `apache/superset` container, publish the port, systemd unit, no Docker socket (`needs_docker=false`)
- [ ] 2.2 Provide the Exasol dialect: thin derived image (`pip install sqlalchemy-exasol`) vs install-at-start (design open question 1)
- [ ] 2.3 Create Superset admin from the generated password (`superset fab create-admin`)
- [ ] 2.4 Seed the Exasol DB connection (`exa+websocket://...SSLCertificate=SSL_VERIFY_NONE`) via Superset import/API
- [ ] 2.5 (Optional) pre-import a starter SQL Lab query / dashboard over `PRODUCTS`/`PRODUCT_REVIEWS` (design open question 2)

## 3. Infrastructure preset (AWS first)

- [ ] 3.1 Declare companion-app support; add `with_superset` + `superset_port` infra variables
- [ ] 3.2 Security-group ingress for the Superset port, gated by `allowed_cidr`
- [ ] 3.3 Generate `supersetAdminPassword` (`random_password`) → `secrets.json`
- [ ] 3.4 Inject settings into `cloudinit.tf`, register `installSuperset.sh` in `postInstall.scripts`
- [ ] 3.5 Document the opt-in bits in `doc/presets.md` (extend the AI Lab section to the generic companion-app pattern)

## 4. CLI & metadata

- [ ] 4.1 `--with-superset` / `--superset-port` flags (auto-generated from infra variables)
- [ ] 4.2 Optional `superset` connection object in `deployment.json`; `supersetAdminPassword` in `config.Secrets`
- [ ] 4.3 Extend `exasol info` / connection-instructions template to list companion apps (URL + secrets reference), shown only when present
- [ ] 4.4 Surface a footprint note when multiple `--with-*` add-ons are enabled together

## 5. Tests

- [ ] 5.1 Preset capability resolution (companion-apps / superset provided by aws, not local)
- [ ] 5.2 `--with-superset` flag auto-generation
- [ ] 5.3 Metadata read/write (superset present / omitted)
- [ ] 5.4 Connection-instructions output with/without Superset (URL shown, password referenced not printed)

## 6. Documentation

- [ ] 6.1 README: companion-app concept; install Superset (`--with-superset`), reach it, zero-config DB connection
- [ ] 6.2 Note the exposed port, the `secrets.json` entry, and the `allowed_cidr` / tunnel recommendation

## 7. Verification

- [ ] 7.1 Manual verification on a live AWS deployment: `--with-superset` opens the port, Superset reachable, the pre-seeded Exasol connection works in SQL Lab against the sample data
- [ ] 7.2 Verify a deployment without `--with-superset` shows no Superset section and opens no port
- [ ] 7.3 Verify AI Lab + Superset can run together on the default instance (footprint sanity)
