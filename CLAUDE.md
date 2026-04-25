# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Dockerized Znuny (OTRS Community Edition fork) ticketing system. Two-container architecture: an OTRS application container (CentOS 7, Apache, Perl/mod_perl) and a MariaDB 10.1 database container. Forked from `juanluisbaptiste/docker-otrs`.

## Build & Run Commands

```bash
# Development build (uses docker-compose.override.yml automatically)
docker-compose build

# Development build with no cache
docker-compose build --no-cache

# Run (foreground)
docker-compose up

# Run (background)
docker-compose up -d

# Production (skip override, use pre-built images)
docker-compose -f docker-compose.yml up -d

# Helper script (build + run)
./launch-testing.sh -b -r        # build and run
./launch-testing.sh -B -r        # no-cache build and run
./launch-testing.sh -c           # clean volumes first
./launch-testing.sh -V           # debug mode
```

## Architecture

### Container Structure

- **`otrs/`** — OTRS application container. `Dockerfile` installs Znuny 6.0.45 RPM on CentOS 7. `run.sh` is the entrypoint; it sources `functions.sh` (core init logic) and `util_functions.sh` (logging). Supervisord manages background processes (Apache, cron, rsyslog).
- **`mariadb/`** — Database container. `Dockerfile` extends `mariadb:10.1` with OTRS-specific MySQL settings (UTF-8, packet sizes). `run.sh` handles first-run root password setup.

### Startup Flow (otrs/run.sh)

1. Wait for database availability (`mysqladmin ping` loop)
2. Branch on `OTRS_INSTALL`: `no` (load defaults), `restore` (restore backup), `yes` (web installer, unsupported)
3. Set admin password, configure permissions, install addons from `/opt/otrs/addons`
4. Rebuild config cache, configure skins, start OTRS daemon + cron
5. Launch supervisord, enter signal-wait loop (handles SIGTERM gracefully)

### Compose Files

- **`docker-compose.yml`** — Production config using pre-built Docker Hub images
- **`docker-compose.override.yml`** — Dev overrides: adds `build:` directives to build from local Dockerfiles, uses `:dev` image tags
- **`stack.yml`** — Docker Swarm deployment with Traefik reverse proxy, SSL, sticky sessions

### Key Volume Mounts

| Container Path | Purpose |
|---|---|
| `/opt/otrs/Kernel` | OTRS configuration (persistent, required) |
| `/var/otrs/backups` | Backup storage (required) |
| `/var/lib/mysql` | MariaDB data (required) |
| `/opt/otrs/addons` | `.opm` module packages |
| `/opt/otrs/var/httpd/htdocs/skins/` | Custom skins |
| `/opt/otrs/db_upgrade` | SQL upgrade scripts |

## Configuration

All configuration is via environment variables. Copy `.env.example` to `.env` to get started.

**Critical variables:** `OTRS_DB_PASSWORD`, `MYSQL_ROOT_PASSWORD`, `OTRS_ROOT_PASSWORD` (all default to `changeme`).

**Operational modes:** `OTRS_INSTALL` controls startup behavior (`no`=default load, `restore`=restore backup, `yes`=web installer). `OTRS_UPGRADE=yes` triggers major version upgrade flow.

**Backup:** Controlled by `OTRS_BACKUP_TIME` (cron schedule, default `0 4 * * *`), `OTRS_BACKUP_TYPE`, `OTRS_BACKUP_COMPRESSION`, `OTRS_BACKUP_ROTATION` (days). Backup script is `otrs/otrs_backup.sh`.

**Docker Secrets:** Supported via `OTRS_SECRETS_FILE` for production credential management.

## Web Access

- Admin: `http://localhost/otrs/index.pl`
- Customer: `http://localhost/otrs/customer.pl`
- MariaDB: port 3306 (internal only, not exposed to host)

## CI/CD

Uses Drone CI (`.drone.yml`) to build and publish images to Docker Hub. No GitHub Actions or test suites in the repo.
