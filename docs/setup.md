# Znuny Ticketing System - NMQ Deployment

Znuny (OTRS Community Edition fork) running on the OVHcloud VPS behind Pangolin reverse proxy. Uses an external shared MariaDB instance.

## Architecture

```
Internet → Pangolin (SSL) → znuny_otrs:80 ──┐
                                              ├── nmqnet
                         mariadb_prod:3306 ──┘
```

- **znuny_otrs** — Znuny application (CentOS 7, Apache, Perl)
- **mariadb_prod** — Shared MariaDB instance ([separate repo](https://github.com/dmpohlmann/mariadb_prod))
- **nmqnet** — Docker bridge network connecting all services
- **Pangolin** — Reverse proxy handling SSL termination and routing

## Prerequisites

- Docker and Docker Compose installed
- `nmqnet` network exists: `docker network create nmqnet`
- `mariadb_prod` container running (see [MariaDB setup](https://github.com/dmpohlmann/mariadb_prod/blob/master/docs/setup.md))
- Pangolin configured to route `support.nmq.info` → `znuny_otrs:80`

## Installation

### 1. Clone and enter the repo

```bash
git clone https://github.com/dmpohlmann/znuny_nmq.git
cd znuny_nmq
```

### 2. Configure secrets

Edit `secrets/otrs_secrets` with real credentials:

```
OTRS_DB_PASSWORD=<otrs db user password>
MYSQL_ROOT_PASSWORD=<must match mariadb_prod root password>
OTRS_ROOT_PASSWORD=<znuny admin login password>
```

```bash
chmod 600 secrets/otrs_secrets
```

### 3. Configure environment

Edit `.env` and update the SMTP settings:

```
OTRS_SMTP_SERVER=<your smtp host>
OTRS_SMTP_PORT=587
OTRS_SMTP_USERNAME=<smtp username>
OTRS_SMTP_PASSWORD=<smtp password>
```

All other values (hostname, timezone, backup schedule) are pre-configured.

### 4. Create the OTRS database

On the running `mariadb_prod` container:

```bash
docker exec -it mariadb_prod mysql -u root -p
```

```sql
CREATE DATABASE otrs CHARACTER SET utf8 COLLATE utf8_general_ci;
GRANT ALL ON otrs.* TO 'otrs'@'%' IDENTIFIED BY '<same password as OTRS_DB_PASSWORD>';
FLUSH PRIVILEGES;
```

### 5. Start Znuny

```bash
docker compose up -d
```

First boot takes a few minutes while it initialises the database schema and default configuration.

### 6. Verify

Check logs for successful startup:

```bash
docker logs -f znuny_otrs
```

Look for the OTRS ASCII logo and "Starting OTRS daemon" message.

## Access

| Interface | URL |
|-----------|-----|
| Admin | `https://support.nmq.info/otrs/index.pl` |
| Customer | `https://support.nmq.info/otrs/customer.pl` |

Default admin login: `root@localhost` with the password set in `OTRS_ROOT_PASSWORD`.

## Domains

Currently served at `support.nmq.info`. When the domain migration completes, Pangolin should also route `support.northmarque.net` to the same container.

## Backups

Automated daily at 04:00 AEST (Australia/Brisbane). Stored in `./backups/`.

| Setting | Value |
|---------|-------|
| Schedule | `0 4 * * *` |
| Type | Full backup |
| Compression | gzip |
| Retention | 30 days |

### Manual backup

```bash
docker exec znuny_otrs /opt/otrs/scripts/backup.pl -d /var/otrs/backups -t fullbackup -c gzip
```

### Restore

Set these in `.env`, then recreate the container:

```
OTRS_INSTALL=restore
OTRS_BACKUP_DATE=<backup folder name, e.g. 2026-04-25_04-00>
```

```bash
docker compose up -d --force-recreate
```

Remove `OTRS_INSTALL=restore` from `.env` after restore completes.

## Useful Commands

```bash
# View logs
docker logs -f znuny_otrs

# Shell into container
docker exec -it znuny_otrs bash

# Restart
docker compose restart

# Stop
docker compose down

# Rebuild (after image update)
docker compose pull && docker compose up -d
```
