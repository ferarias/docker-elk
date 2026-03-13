# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is `docker-elk`: a Docker Compose setup for the Elastic stack (Elasticsearch + Logstash + Kibana) version 7.17.x (End of Life). It is a minimal, template-oriented project — not an application with tests or a build pipeline in the traditional sense.

## Key Commands

**First-time setup** (initializes Elasticsearch users/roles from `.env`):
```sh
docker compose up setup
```

**Start the stack:**
```sh
docker compose up          # foreground
docker compose up -d       # detached
```

**Rebuild images** (required after branch switch or version change):
```sh
docker compose build
```

**Tear down and remove all data:**
```sh
docker compose down -v
```

**Enable an extension** (e.g., Metricbeat):
```sh
docker compose -f docker-compose.yml -f extensions/metricbeat/metricbeat-compose.yml up -d metricbeat
```

**Reset a user password via API:**
```sh
curl -XPOST 'http://localhost:9200/_security/user/elastic/_password' \
    -H 'Content-Type: application/json' \
    -u elastic:<current_password> \
    -d '{"password": "<new_password>"}'
```

## Architecture

### Core Services (`docker-compose.yml`)

- **`setup`** (profile: `setup`): One-off init container. Runs `setup/entrypoint.sh` which calls the Elasticsearch API to create/update users and roles defined in `setup/roles/*.json`. Only needs to run once initially; re-running resets passwords to values in `.env`.
- **`elasticsearch`**: Single-node cluster with X-Pack security enabled (trial license). Data persisted in a named Docker volume.
- **`logstash`**: Reads from `logstash/pipeline/logstash.conf`. Accepts Beats input (port 5044) and TCP (port 50000); outputs to Elasticsearch as `logstash_internal` user.
- **`kibana`**: UI at `http://localhost:5601`. Connects to Elasticsearch as `kibana_system` user.

### Configuration Pattern

All service configuration is file-mounted as read-only into containers:
- `elasticsearch/config/elasticsearch.yml`
- `logstash/config/logstash.yml` + `logstash/pipeline/logstash.conf`
- `kibana/config/kibana.yml`

**Configuration is not hot-reloaded** — restart the affected service after any config change.

### `.env` File

Central source of truth for:
- `ELASTIC_VERSION` — controls image tags for all services
- Passwords for all Elasticsearch users (`ELASTIC_PASSWORD`, `LOGSTASH_INTERNAL_PASSWORD`, `KIBANA_SYSTEM_PASSWORD`, etc.)

Passwords flow from `.env` → Docker Compose environment variables → service config files (e.g., `kibana.yml` references `${KIBANA_SYSTEM_PASSWORD}`).

### Extensions

Located in `extensions/`. Each extension is a separate Docker Compose override file (e.g., `extensions/metricbeat/metricbeat-compose.yml`) that adds services to the core stack. Extensions are opt-in and merged with `-f` flags. Available extensions: `filebeat`, `fleet`, `heartbeat`, `metricbeat`, `enterprise-search`, `curator`.

### Ports

| Port  | Service               |
|-------|-----------------------|
| 9200  | Elasticsearch HTTP    |
| 9300  | Elasticsearch TCP     |
| 5601  | Kibana                |
| 5044  | Logstash Beats input  |
| 50000 | Logstash TCP input    |
| 9600  | Logstash monitoring   |

## Adding Plugins

1. Add a `RUN` statement to the relevant `Dockerfile` (e.g., `logstash/Dockerfile`)
2. Add plugin configuration to the service config
3. Rebuild: `docker compose build`
