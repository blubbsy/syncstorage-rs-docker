# Firefox SyncStorage-RS Docker

A reliable, production-ready, and automated Docker deployment for the official [Mozilla Firefox Sync Server (`syncstorage-rs`)](https://github.com/mozilla-services/syncstorage-rs).

---

## Acknowledgments & Lineage

This project builds upon the work of several open-source community contributors:
- **Mozilla Services**: Creators of the official Rust [`syncstorage-rs`](https://github.com/mozilla-services/syncstorage-rs) codebase.
- **[jinkang06](https://github.com/jinkang06/syncstorage-rs-docker)**: Re-architected multi-stage Docker builds, unified Bookworm OS environments, and introduced official MariaDB initialization.
- **[Dan-r](https://github.com/Dan-r/syncstorage-RS-docker)**: The original docker-compose packaging implementation for `syncstorage-rs`.

### Enhancements in this Repository (`blubbsy/syncstorage-rs-docker`):
- **MariaDB 10.11 LTS**: Upgraded from 10.6 to MariaDB 10.11 with automatic schema upgrade flags and robust container health checks (`healthcheck.sh`).
- **Debian Trixie Runtime**: Updated to modern compiler toolchains, Rust 1.98+, and Python 3.13 integration (`libpython3.13`).
- **TLS Client Fixes**: Handled non-interactive client SSL connection requirements (`--skip-ssl`) across container networks.
- **Automated CI/CD Release Pipeline**: Nightly GitHub Action workflow automatically checks upstream `mozilla-services/syncstorage-rs` for new releases, builds the container image in the cloud, and publishes it to GitHub Container Registry (`ghcr.io/blubbsy/syncstorage-rs-docker`).
- **Watchtower Automated Updates**: Optional Watchtower service periodically checks GHCR and upgrades `firefox_syncserver` with zero local compilation load.

---

## Deployment Options

### Option 1: Pre-built Image (Recommended - Fast & Low Resource)

Use this method to avoid compiling Rust on your server (saving 5–10 minutes of heavy CPU and memory usage):

In `docker-compose.yaml`:
```yaml
services:
  mariadb:
    container_name: firefox_mariadb
    image: mariadb:10.11
    restart: unless-stopped
    volumes:
      - mariadb_data:/var/lib/mysql
      - ./data/initdb.d/init.sql:/docker-entrypoint-initdb.d/init.sql
    environment:
      MYSQL_USER: sync
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MARIADB_AUTO_UPGRADE: "1"
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 5

  syncserver:
    container_name: firefox_syncserver
    image: ghcr.io/blubbsy/syncstorage-rs-docker:latest
    restart: unless-stopped
    ports:
      - "5000:8000"
    depends_on:
      mariadb:
        condition: service_healthy
    environment:
      LOGLEVEL: ${LOGLEVEL}
      SYNC_URL: ${SYNC_URL}
      SYNC_CAPACITY: ${SYNC_CAPACITY}
      SYNC_MASTER_SECRET: ${SYNC_MASTER_SECRET}
      METRICS_HASH_SECRET: ${METRICS_HASH_SECRET}
      DB_HOST: ${DB_HOST}
      DB_PORT: ${DB_PORT}
      DB_USER: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
      DB_SYNCSTORAGE_NAME: ${DB_SYNCSTORAGE_NAME}
      DB_TOKENSERVER_NAME: ${DB_TOKENSERVER_NAME}
      SYNC_SYNCSTORAGE_DATABASE_URL: mysql://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_SYNCSTORAGE_NAME}
      SYNC_TOKENSERVER_DATABASE_URL: mysql://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_TOKENSERVER_NAME}

  watchtower:
    container_name: firefox_watchtower
    image: containrrr/watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    command: --interval 86400 --cleanup firefox_syncserver

volumes:
  mariadb_data:
```

### Option 2: Local Source Build

Build directly from source using the local `Dockerfile`:
```bash
docker compose up -d --build
```

---

## Quick Start Guide

### 1. Clone the repository
```bash
git clone https://github.com/blubbsy/syncstorage-rs-docker.git
cd syncstorage-rs-docker
```

### 2. Configure Environment Variables
Copy `example.env` to `.env`:
```bash
cp example.env .env
```
Edit `.env` and fill in your secrets. Ensure `SYNC_MASTER_SECRET` and `METRICS_HASH_SECRET` are random 64-character hexadecimal strings:
```bash
# Generate random 64-character hex secrets:
openssl rand -hex 32
```

### 3. Start the Stack
```bash
docker compose up -d
```

### 4. Verify the Installation
Check that the heartbeat endpoint reports database status as `Ok`:
```bash
curl http://localhost:5000/__heartbeat__
# Expected response:
# {"version":"0.25.0","quota":{"enabled":false,"size":2097152000},"database":"Ok","status":"Ok"}
```

Check the active version:
```bash
curl http://localhost:5000/__version__
```

---

## Firefox Client Setup

Once the server is running behind your reverse proxy (with HTTPS):

1. Open Firefox and navigate to `about:config`.
2. Accept the risk warning.
3. Search for:
   ```text
   identity.sync.tokenserver.uri
   ```
4. Change its value to your server URL followed by `/1.0/sync/1.5`:
   ```text
   https://sync.yourdomain.com/1.0/sync/1.5
   ```
5. *(Optional debugging)* To view detailed sync logs in `about:sync-log`, set:
   ```text
   services.sync.log.appender.file.logOnSuccess = true
   ```
6. Sign in to your Firefox Account or trigger **Sync Now** from the Firefox menu.

---

## License

This project is open-source under the Mozilla Public License 2.0 (MPL-2.0), matching the upstream Mozilla `syncstorage-rs` codebase.
