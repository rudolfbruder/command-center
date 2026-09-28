# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Shared local Docker infrastructure for Laravel e-commerce projects — not an application. There is no build, lint, or test suite; the only artifact is `docker-compose.yaml` plus config under `docker/`. Laravel apps live in other repos and connect to these services.

## Stack

| Service | Image | Host port (all bound to 127.0.0.1) |
|---|---|---|
| `db` | `mariadb:13.0` | `FORWARD_DB_PORT` (3306) |
| `phpmyadmin` | `phpmyadmin:5.2.3` | `PHPMYADMIN_PORT` (8081) |
| `redis` | `redis:8.10-alpine` | `FORWARD_REDIS_PORT` (6379) |
| `redisinsight` | `redis/redisinsight:3.8` (listens on 5540, data in `/data`) | `FORWARD_REDIS_INSIGHT_PORT` (8003) |
| `setup` | elasticsearch (one-shot cert generation) | — |
| `copy-elastic-certs` | `alpine:3.22` (one-shot) | — |
| `elasticsearch` | elasticsearch `${ES_VERSION:-9.5.3}` (single node, TLS + auth) | 9200 (HTTPS) |
| `kibana-password` | elasticsearch (one-shot, sets `kibana_system` password) | — |
| `kibana` | kibana `${ES_VERSION:-9.5.3}` (plain HTTP, log in as `elastic`) | 5601 |
| `selenium` | `selenium/standalone-chromium:4.49` | 4444 (WebDriver), 7900 (noVNC) |

All services share the bridge network named `command-center` (fixed name, no project prefix). External Laravel stacks join it as an `external` network.

## Commands

```bash
cp .env.example .env          # then set passwords
docker compose up -d
docker compose down           # keeps named volumes
docker compose down -v        # wipes all data volumes
docker compose logs -f elasticsearch

# connectivity checks
mysql -h127.0.0.1 -P${FORWARD_DB_PORT:-3306} -u${DB_USERNAME} -p
redis-cli -h 127.0.0.1 -p ${FORWARD_REDIS_PORT:-6379} ping
curl --cacert docker/elasticsearch/certs/ca/ca.crt -u elastic:${ELASTIC_PASSWORD} https://localhost:9200

# import a dump (docker/dumps is mounted at /tmp/dumps in the db container)
docker exec -i mariadb sh -c 'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" "$MARIADB_DATABASE" < /tmp/dumps/<file>.sql'
```

Regenerate ES certs: delete contents of `docker/elasticsearch/certs/` and `docker compose up` again.

## Architecture Notes

**Elasticsearch bootstrap chain**: `setup` (generates CA + node cert into `docker/elasticsearch/certs/`, skipped if `ca.zip`/`certs.zip` exist) → on `service_completed_successfully`, `elasticsearch` and `copy-elastic-certs` start → once ES is healthy, `kibana-password` sets the `kibana_system` password → `kibana` starts. One-shots use `service_completed_successfully` so partial `docker compose up <service>` works after they have exited.

**Cert handoff to apps**: `copy-elastic-certs` copies only `ca.crt` into `storage/app/private/elastic/`. That same directory is mounted into ES as `config/wordfiles/`, so synonym/stopword files for analyzers go there too.

**Upgrades**: MariaDB runs with `MARIADB_AUTO_UPGRADE=1`, so a major bump upgrades the data dir on start. Elasticsearch/Kibana can only move to a new major from the last minor of the previous one (for example 8.x → 8.19 → 9.x). To hop, run the stack once with `ES_VERSION=<last minor>` before the jump.

**Hunspell dictionaries** (sk_SK, cs_CZ, hu_HU, en_GB) are bind-mounted into `config/hunspell/` for multilingual analyzers.

**phpMyAdmin** uses config auth as root (`docker/phpmyadmin/config.inc.php`), taking the password from `PMA_PASSWORD` (= `DB_PASSWORD`). MariaDB only creates a root user, never `MARIADB_USER`: `DB_USERNAME` is expected to be `root`.

**Gitignored**: `.env`, `storage/`, `docker/elasticsearch/certs/`, `docker/dumps/*.sql`, `dump_mysql.zip`.

**Named volumes**: `mariadb-eshop`, `redis-eshop`, `redisinsight-eshop`, `elasticsearch-eshop-data`.
