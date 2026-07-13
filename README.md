# postgres_swarm

An Ansible role that deploys PostgreSQL (PostGIS image with WAL-G) as a
Docker Swarm stack, with optional WAL-G archiving/recovery to S3 and an
optional Prometheus `postgres-exporter` sidecar service.

## What the role does

1. Installs `python3-jsondiff` (required by the `community.docker.docker_stack`
   module) and ensures the host data directory exists.
2. Builds a Docker Compose v3.8 definition with a `postgres` service.
3. When `postgres_swarm_exporter` is `true`, adds a `postgres-exporter`
   service to the same stack.
4. Deploys the stack via `community.docker.docker_stack`.

## Requirements

- Ansible 2.8+
- Docker Engine running in Swarm mode on the target host
- The `community.docker` collection installed on the controller:
  `ansible-galaxy collection install community.docker`
- An **external** overlay network must already exist (see
  `postgres_swarm_network_name`); the role references it as `external: true`
  and does not create it.
- The `postgres` image is expected to bundle WAL-G and read its config from
  `/etc/postgresql/postgresql.conf`.

## Role Variables

All variables are defined in [`defaults/main.yml`](defaults/main.yml).

### PostgreSQL

| Variable | Default | Description |
| --- | --- | --- |
| `postgres_swarm_user` | `postgres` | Superuser name (`POSTGRES_USER` / `PGUSER`). |
| `postgres_swarm_password` | `password` | Superuser password (`POSTGRES_PASSWORD` / `PGPASSWORD`). |
| `postgres_swarm_db_name` | `my_db` | Database name (`POSTGRES_DB` / `PGDATABASE`). |
| `postgres_swarm_archive_mode` | `'off'` | WAL archive mode passed as `ARCHIVE_MODE`. |
| `postgres_swarm_archive_timeout` | `600` | `ARCHIVE_TIMEOUT` in seconds. |
| `postgres_swarm_pgdata` | `/var/lib/postgresql/data/pgdata` | `PGDATA` location inside the container. |
| `postgres_swarm_pghost` | `/var/run/postgresql` | `PGHOST` (local socket dir). |
| `postgres_swarm_recovery_target_time` | `2024-01-15 14:30:00` | PITR target (currently commented out in the compose). |

### WAL-G / S3

| Variable | Default | Description |
| --- | --- | --- |
| `postgres_swarm_aws_access_key_id` | `aws_id` | `AWS_ACCESS_KEY_ID`. |
| `postgres_swarm_aws_secret_access_key` | `secret` | `AWS_SECRET_ACCESS_KEY`. |
| `postgres_swarm_aws_region` | `ru-central1` | `AWS_REGION`. |
| `postgres_swarm_aws_endpoint` | `''` | `AWS_ENDPOINT` (set for S3-compatible storage). |
| `postgres_swarm_walg_s3_prefix` | `s3://walg-backup` | `WALG_S3_PREFIX` backup target. |
| `postgres_swarm_apprise_target` | `tgram://secret` | `APPRISE_TARGET` for backup notifications. |
| `postgres_swarm_recovery_walg` | `'false'` | `RECOVERY_WALG`; set to `true` to restore from WAL-G on start. |

WAL-G compression is hardcoded in the compose (`WALG_COMPRESSION_METHOD: brotli`,
`WALG_DELTA_MAX_STEPS: 6`).

### Docker Swarm

| Variable | Default | Description |
| --- | --- | --- |
| `postgres_swarm_stack_name` | `my_postgres_stack` | Swarm stack name. |
| `postgres_swarm_image` | `harbor.escapenavigator.com/library/postgis_with_walg:13-3.2-bis` | PostgreSQL/PostGIS+WAL-G image. |
| `postgres_swarm_volume_path` | `/var/lib/postgresql/data` | Host path bind-mounted into the container. |
| `postgres_swarm_target_host` | `postgres-host` | Swarm node hostname the services are pinned to (`node.hostname` constraint). |
| `postgres_swarm_network_name` | `project-network` | Name of the pre-existing external overlay network. |
| `postgres_swarm_shm_size` | `512m` | Size of the `/dev/shm` tmpfs mount for the postgres container. Accepts human-readable values (e.g. `512m`, `1g`) and is converted to bytes via `human_to_bytes`. Implemented as a tmpfs mount because Swarm ignores the compose `shm_size` key. |

### Prometheus postgres-exporter (optional)

| Variable | Default | Description |
| --- | --- | --- |
| `postgres_swarm_exporter` | `true` | Deploy the exporter sidecar alongside postgres. |
| `postgres_swarm_exporter_image` | `prometheuscommunity/postgres-exporter:latest` | Exporter image. |
| `postgres_swarm_exporter_port` | `9187` | Host port published for the exporter (maps to container `9187`). |
| `postgres_swarm_exporter_sslmode` | `disable` | `sslmode` used in the exporter's `DATA_SOURCE_NAME`. |

When enabled, the exporter connects to the postgres service over the overlay
network using DSN
`postgresql://<user>:<password>@postgres:5432/<db>?sslmode=<sslmode>`,
and exposes metrics on the published port for Prometheus to scrape.

### WAL-G backup cron (optional)

| Variable | Default | Description |
| --- | --- | --- |
| `postgres_swarm_backup_cron_enabled` | `true` | Install the cron jobs below on the target host. |
| `postgres_swarm_backup_cron_minute` | `'0'` | Minute field of the `backup-push` cron schedule. |
| `postgres_swarm_backup_cron_hour` | `'3'` | Hour field of the `backup-push` cron schedule. |
| `postgres_swarm_backup_cron_day` | `'*'` | Day-of-month field of the `backup-push` cron schedule. |
| `postgres_swarm_backup_cron_month` | `'*'` | Month field of the `backup-push` cron schedule. |
| `postgres_swarm_backup_cron_weekday` | `'*'` | Weekday field of the `backup-push` cron schedule. |
| `postgres_swarm_delete_cron_minute` | `'0'` | Minute field of the `delete before` cron schedule. |
| `postgres_swarm_delete_cron_hour` | `'1'` | Hour field of the `delete before` cron schedule. |
| `postgres_swarm_delete_cron_day` | `'*'` | Day-of-month field of the `delete before` cron schedule. |
| `postgres_swarm_delete_cron_month` | `'*'` | Month field of the `delete before` cron schedule. |
| `postgres_swarm_delete_cron_weekday` | `'*'` | Weekday field of the `delete before` cron schedule. |
| `postgres_swarm_backup_retention_days` | `7` | Backups older than this are pruned by `wal-g delete before FIND_FULL`. |
| `postgres_swarm_log_dir` | `/var/log/postgresql` | Directory for the cron jobs' log files. |

Each schedule field maps 1:1 to the corresponding `ansible.builtin.cron`
parameter (`minute`/`hour`/`day`/`month`/`weekday`), so any standard cron
expression is supported (e.g. `postgres_swarm_backup_cron_weekday: '0,3'` to
run twice a week, or `postgres_swarm_backup_cron_minute: '*/30'` for
half-hourly runs).

Since Swarm assigns each container a name with a random per-task suffix
(e.g. `my_postgres_stack_postgres.1.<id>`), the container to back up cannot
be hardcoded. Both cron jobs resolve it inline at run time via
`docker ps -q --filter label=com.docker.swarm.service.name=<stack>_postgres`
(an exact label match, so it can't collide with the `postgres-exporter`
service) and then run `wal-g.sh backup-push` / `wal-g.sh delete before
FIND_FULL ... --confirm` inside it, redirecting output to
`postgres_swarm_log_dir`.

## Example Playbook

```yaml
- name: Deploy PostgreSQL on Docker Swarm
  hosts: swarm_managers
  remote_user: root
  roles:
    - role: postgres_swarm
      vars:
        postgres_swarm_stack_name: app_postgres
        postgres_swarm_db_name: app
        postgres_swarm_user: app
        postgres_swarm_password: "{{ vault_pg_password }}"
        postgres_swarm_target_host: db-node-1
        postgres_swarm_network_name: app-overlay

        # WAL-G backups to S3-compatible storage
        postgres_swarm_archive_mode: 'on'
        postgres_swarm_walg_s3_prefix: s3://my-backups/pg
        postgres_swarm_aws_endpoint: https://storage.yandexcloud.net
        postgres_swarm_aws_access_key_id: "{{ vault_s3_key_id }}"
        postgres_swarm_aws_secret_access_key: "{{ vault_s3_secret }}"

        # Metrics
        postgres_swarm_exporter: true
        postgres_swarm_exporter_port: 9187
```

## Notes & Caveats

- **Secrets in env vars.** The DB password is passed via environment variables
  (also into the exporter DSN) and is visible in `docker service inspect`.
  Consider Docker secrets for production.
- **External network required.** The role does not create
  `postgres_swarm_network_name`; create the overlay network beforehand.
- **Single replica, pinned node.** Both services run with `replicas: 1` and a
  `node.hostname == {{ postgres_swarm_target_host }}` constraint, matching the
  host bind-mount for data persistence.
- The bundled `handlers/main.yml` and `meta/main.yml` still contain leftover
  metadata from the role's nginx-exporter origin and are not used by the
  PostgreSQL tasks.

## License

Apache-2.0 (see [LICENSE](LICENSE)).
