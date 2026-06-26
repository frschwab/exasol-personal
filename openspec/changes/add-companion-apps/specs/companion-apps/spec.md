## ADDED Requirements

### Requirement: Generic companion-app installation mechanism
The system SHALL provide a reusable mechanism for optionally installing additional pre-configured containers ("companion apps") alongside the database on supporting infrastructure, so that adding a new app does not require duplicating the full install script.

#### Scenario: AI Lab is installed via the companion-app mechanism
- **WHEN** AI Lab is requested
- **THEN** it is installed through the shared companion-app mechanics (rootless Podman run, systemd unit + lingering, firewall exposure, connection-info output) with no change to its existing user-facing behavior

#### Scenario: A new companion app reuses the shared mechanics
- **WHEN** a new companion app is added
- **THEN** it declares its image, port, secrets, connection-seeding step, and info output, and reuses the shared install/persistence/exposure mechanics rather than reimplementing them

#### Scenario: Docker-socket complexity is opt-in per app
- **WHEN** a companion app does not require a Docker daemon (e.g. Superset)
- **THEN** the Podman-socket setup (socket mount, search registry, image-checker patch, socket mode) is not applied for that app

### Requirement: Companion apps are opt-in and preset-gated
The system SHALL install a companion app only when explicitly requested, and only for infrastructure presets that declare support.

#### Scenario: Request a companion app on a supporting preset
- **WHEN** a user runs `exasol install` for a supporting preset with the app's `--with-<app>` flag
- **THEN** the launcher installs the database and additionally installs that companion app

#### Scenario: Companion apps unavailable on unsupported infrastructure
- **WHEN** a deployment uses infrastructure that does not provide the companion-apps capability (e.g. local)
- **THEN** the `--with-<app>` flags are not offered and no companion app is installed

### Requirement: Companion-app connection information is surfaced
The system SHALL display connection information for each installed companion app in `exasol info`, and omit it when the app is not present.

#### Scenario: Installed app appears in info
- **WHEN** a companion app is installed for a deployment
- **THEN** `exasol info` shows its URL and references its credentials in `secrets.json` without printing the secrets

#### Scenario: Absent app is omitted
- **WHEN** a companion app is not installed
- **THEN** `exasol info` shows no section for it and no corresponding port is opened

### Requirement: Apache Superset BI add-on
The system SHALL offer Apache Superset as a companion app, pre-configured with a working Exasol database connection.

#### Scenario: Install Superset with a pre-seeded Exasol connection
- **WHEN** a user installs with `--with-superset` on a supporting preset
- **THEN** the launcher runs Superset, creates an admin user from a generated password stored in `secrets.json`, and pre-creates a Superset database connection to the deployment's Exasol database (using the `sqlalchemy-exasol` dialect, with validation disabled for the self-signed certificate)

#### Scenario: Superset is reachable and usable against sample data
- **WHEN** Superset has been installed and the deployment is running
- **THEN** Superset is reachable on its published port (gated by `allowed_cidr`) and its pre-seeded connection can query the sample data in SQL Lab without manual configuration

#### Scenario: Superset does not require the Docker socket
- **WHEN** Superset is installed
- **THEN** no Podman/Docker API socket is mounted into the Superset container
