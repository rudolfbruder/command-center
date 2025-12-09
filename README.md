# Command Center docker stack

This repo ships a `docker-compose.yaml` that spins up the full local stack used by the project: MariaDB + phpMyAdmin, Redis, Elasticsearch (with TLS cert bootstrapping), Kibana, and a Selenium Chrome node.

## Dev-only usage and safety
- This stack is for local development only. It assumes you supply your own `.env` with non-production credentials.
- Generated Elasticsearch certs and keys are not committed and are gitignored (`docker/elasticsearch/certs/`). They will be created on first `docker compose up`.
- Keep any database dumps scrubbed; `docker/dumps/*.sql` is gitignored to avoid leaking data.
- phpMyAdmin uses a dev-friendly configuration; set `PMA_BLOWFISH_SECRET` in `.env` to avoid cookie auth warnings.

## Prerequisites
- Docker Desktop or Docker Engine + Compose v2
- A populated `.env` in the repo root with the variables referenced below

## Environment variables used by `docker-compose.yaml`
- Database: `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`, `FORWARD_DB_PORT` (defaults to 3306)
- phpMyAdmin: `PHPMYADMIN_PORT` (defaults to 8081)
- phpMyAdmin cookie encryption: `PMA_BLOWFISH_SECRET` (32-char random string; optional but recommended)
- Redis: `FORWARD_REDIS_PORT` (defaults to 6379)
- Elasticsearch/Kibana TLS + auth: `ELASTIC_PASSWORD`, `KIBANA_PASSWORD`, `CERTS_DIR` (mount target for generated certs)

## Services overview
- `db`: MariaDB 10 with data in `mariadb-eshop` volume; exposes `${FORWARD_DB_PORT:-3306}`.
- `phpmyadmin`: UI for MariaDB on `${PHPMYADMIN_PORT:-8081}`; mounts `docker/phpmyadmin/config.inc.php`.
- `redis`: Redis Alpine with data in `redis-eshop` volume; exposes `${FORWARD_REDIS_PORT:-6379}`.
- `setup`: One-off Elasticsearch container that generates TLS certs under `docker/elasticsearch/certs`; waits for ES and sets `kibana_system` password.
- `copy-elastic-certs`: Copies generated CA key/cert into `storage/app/private/elastic` for app consumption.
- `elasticsearch`: Single-node ES 8.7 with TLS + basic auth enabled; data in `elasticsearch-eshop-data`; listens on `9200`.
- `kibana`: Kibana 8.7 bound to `5601`, wired to the secured ES instance.
- `selenium`: Standalone Chrome 3.11 node for browser automation tests.

## Quick start
1) Create `.env` with required vars (see above). Ensure `CERTS_DIR` matches the mount path in `docker-compose.yaml` (defaults to `/usr/share/elasticsearch/config/certs` inside containers).
2) Start the stack:
   ```bash
   docker compose up -d
   ```
3) Verify services:
   - MariaDB: `mysql -h127.0.0.1 -P${FORWARD_DB_PORT:-3306} -u${DB_USERNAME} -p`
   - phpMyAdmin: http://localhost:${PHPMYADMIN_PORT:-8081}
   - Redis: `redis-cli -h 127.0.0.1 -p ${FORWARD_REDIS_PORT:-6379} ping`
   - Elasticsearch: `curl --cacert docker/elasticsearch/certs/ca/ca.crt -u elastic:${ELASTIC_PASSWORD} https://localhost:9200`
   - Kibana: https://localhost:5601 (use `kibana_system` / `${KIBANA_PASSWORD}`)

## Notes on cert generation
- The repo does not ship certs; `docker/elasticsearch/certs/` is empty and gitignored.
- The `setup` service runs first and creates a CA plus node certificates in `docker/elasticsearch/certs` on first run.
- `copy-elastic-certs` then copies `ca.crt` and `ca.key` into `storage/app/private/elastic` for downstream use.
- If you need to regenerate, delete the local `docker/elasticsearch/certs` contents and rerun `docker compose up`.

## Maintenance commands
- Stop the stack: `docker compose down`
- Recreate without wiping data: `docker compose up -d --build`
- Reset data (removes named volumes): `docker compose down -v`
- Tail logs for a service: `docker compose logs -f elasticsearch`
