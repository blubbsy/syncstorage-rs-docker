# Stage 1: Builder
# Uses rust:trixie for Rust 1.98+ (Edition 2024) and Python 3.12+ compatibility
FROM rust:trixie AS builder

ARG SYNC_VERSION=0.25.0

WORKDIR /app

# Install build-time dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    cmake \
    libmariadb-dev-compat \
    libmariadb-dev \
    python3-dev \
    python3-pip \
    python3-venv \
    pkg-config \
    && rm -rf /var/lib/apt/lists/*

# Install poetry in an isolated venv to avoid conflicts with Debian system packages
RUN python3 -m venv /opt/poetry-env && \
    /opt/poetry-env/bin/pip install --no-cache-dir poetry poetry-plugin-export && \
    ln -s /opt/poetry-env/bin/poetry /usr/local/bin/poetry

# Clone repo and checkout target release
RUN git clone https://github.com/mozilla-services/syncstorage-rs ./ \
    && git checkout ${SYNC_VERSION}

# Generate version.json so /__version__ reports actual version and commit
RUN printf '{"commit":"%s","version":"%s","source":"https://github.com/mozilla-services/syncstorage-rs","build":"%s"}\n' \
    "$(git rev-parse HEAD)" \
    "${SYNC_VERSION}" \
    "$(date -u +%Y-%m-%dT%H:%M:%SZ)" | tee /app/version.json syncserver/version.json 2>/dev/null || true

# Setup Python virtualenv (handles 0.24+ poetry and legacy requirements.txt)
RUN python3 -m venv /app/venv && \
    if [ -f pyproject.toml ]; then \
        poetry export --no-interaction --without dev --output /app/requirements.txt --without-hashes && \
        if [ -d tools/tokenserver ] && [ -f tools/tokenserver/pyproject.toml ]; then \
            cd tools/tokenserver && poetry export --no-interaction --without dev --output /app/tokenserver-requirements.txt --without-hashes && cd /app; \
        fi; \
    fi && \
    if [ -f /app/requirements.txt ]; then \
        /app/venv/bin/pip install --no-cache-dir -r /app/requirements.txt; \
    fi && \
    if [ -f /app/tokenserver-requirements.txt ]; then \
        /app/venv/bin/pip install --no-cache-dir -r /app/tokenserver-requirements.txt; \
    elif [ -f tools/tokenserver/requirements.txt ]; then \
        /app/venv/bin/pip install --no-cache-dir -r tools/tokenserver/requirements.txt; \
    fi

# Build syncserver and diesel CLI
RUN cargo install --path ./syncserver --no-default-features --features=mysql,py_verifier --locked \
    && cargo install diesel_cli --no-default-features --features 'mysql'

# Bundle migrations cleanly
RUN mkdir -p /app/bundled-migrations/syncstorage /app/bundled-migrations/tokenserver && \
    if [ -d syncstorage-mysql/migrations ]; then cp -r syncstorage-mysql/migrations/* /app/bundled-migrations/syncstorage/; fi && \
    if [ -d tokenserver-mysql/migrations ]; then \
        cp -r tokenserver-mysql/migrations/* /app/bundled-migrations/tokenserver/; \
    elif [ -d tokenserver-db/migrations ]; then \
        cp -r tokenserver-db/migrations/* /app/bundled-migrations/tokenserver/; \
    fi

# ---
# Stage 2: Runtime Image
FROM debian:trixie-slim

RUN groupadd --gid 1000 appuser && \
    useradd --uid 1000 --gid 1000 --shell /bin/bash --create-home appuser

RUN apt-get update && apt-get install -y --no-install-recommends \
    default-mysql-client \
    python3 \
    python3-venv \
    libpython3.13 \
    libmariadb3 \
    libcurl4t64 \
    ca-certificates \
    && apt-get clean && rm -rf /var/lib/apt/lists/* \
    && ldconfig

RUN mkdir -p /app /config && chown -R appuser:appuser /app /config
WORKDIR /app
USER appuser

COPY --from=builder /usr/local/cargo/bin/syncserver /usr/local/cargo/bin/
COPY --from=builder /usr/local/cargo/bin/diesel /usr/local/cargo/bin/
COPY --from=builder /app/bundled-migrations/syncstorage ./syncstorage-mysql/migrations
COPY --from=builder /app/bundled-migrations/tokenserver ./tokenserver-mysql/migrations
COPY --from=builder /app/version.json ./version.json
COPY --from=builder /app/venv ./venv
COPY --chown=appuser:appuser --chmod=755 entrypoint.sh /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
