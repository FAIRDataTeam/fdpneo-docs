# Operations

Day-to-day administration of a running FDPneo: the `fdp` command-line tool, scheduled jobs, backups and upgrades.

## The `fdp` command-line tool

The server ships an administrative CLI. In a compose deployment run it inside the server container; in a development checkout run it with `uv run fdp`.

```bash
# evaluation stack
docker compose -f deploy/stack/compose.yaml --env-file deploy/stack/.env exec server fdp --help
# production stack (from deploy/stack/production)
docker compose exec server fdp --help
```

| Command group | Commands | Purpose |
|---|---|---|
| `fdp db` | `migrate` | Run database migrations to the latest revision. |
| `fdp profile` | `validate`, `apply`, `info`, `export`, `migrate-modular`, `backfill-conformance` | Validate or apply a deployment profile, show the applied one, export the current state as a distributable profile, and migration helpers. |
| `fdp metrics` | `rollup`, `rebase-resources` | Run the metrics roll-ups once; re-key recorded metrics after a base-URL change. |
| `fdp search` | `reindex` | Rebuild the metadata search index from the triple store. |
| `fdp schema` | `sync`, `migrate-namespace` | Refetch remote-sourced SHACL schemas and republish the changed ones; relocate a profile's schemas into the schemas namespace. |
| `fdp ldp` | | LDP-conformance commands. |
| `fdp pid` | `w3id-config`, `w3id-pr`, … | Persistent-identifier helpers (W3ID redirect configuration). |
| `fdp backup` | `dump`, `restore`, `import` | Backup, restore and migration. |
| `fdp index` | | Index and discovery commands. |

Each group has `--help`.

## Cheat-sheet

| Task | Command |
|---|---|
| Show the applied profile | `fdp profile info` |
| Validate a profile bundle | `fdp profile validate <path>` |
| Apply a profile to an uninitialised FDP | `fdp profile apply [path]` (defaults to the bundled profile) |
| Export the current state as a profile | `fdp profile export <path>` |
| Factory reset | re-apply the profile with `--force`, or `POST /fdp-api/admin/reset` as an administrator. Destructive: restores schemas and policies to the profile baseline. |
| Database migrations | `fdp db migrate` |
| Roll up metrics | `fdp metrics rollup` |
| Rebuild the search index | `fdp search reindex` (after bulk imports or if search returns nothing) |
| Refresh external schemas | `fdp schema sync` |

## Scheduled jobs

**Metrics roll-ups.** Raw request rows are aggregated to hourly and daily rows and pruned on a fixed cadence (`FDP_METRICS_AGGREGATE_TO_HOURLY_AFTER_SECONDS`, `FDP_METRICS_DISCARD_HOURLY_AFTER_DAYS`). The bundled stacks run this in-process (`FDP_METRICS_ROLLUP_IN_PROCESS=true`). With more than one server replica, unset it and schedule `fdp metrics rollup` every few minutes from a single place so replicas do not race. The roll-up is also the privacy boundary of ADR-0002: raw rows carry the pseudonymous visitor hash and disappear with it.

**Search reindex** after bulk imports, and **schema sync** if you use remote-sourced schemas.

**Metrics geography** is optional: provide a MaxMind-format city database and set `FDP_METRICS_GEOIP_DATABASE_PATH`. The server repository's `scripts/fetch-geoip.sh` fetches the redistributable DB-IP lite build, and the Docker image bundles it. Without the file, country lookup degrades to "unknown".

## Backups

Durable state lives in two stores that must be backed up and restored together: the triple store (the knowledge graph) and PostgreSQL (operational state, and in the production stack also Keycloak's database). In the compose stacks these are the `graphdb-data` and `postgres-data` volumes; `caddy-data` holds certificates, which can be re-obtained.

For a faithful, store-independent backup of the metadata, use the backup commands (ADR-0016):

| Command | What it does |
|---|---|
| `fdp backup dump <dir>` | Dump every named graph, with a manifest and the audit graphs, to a directory. |
| `fdp backup restore <dir>` | Faithfully restore a dump into the triple store of the same deployment. |
| `fdp backup import <dir>` | Import records from another deployment, adopting them under this deployment's base IRI. |

After a restore or import, rebuild the search index with `fdp search reindex`. The server's developer documentation has a dedicated chapter on [backup, restore and migration](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/dev-docs/08-backup-restore.md).

## Upgrades

1. Read the [changelog](../development/changelog.md) for the versions you are crossing; some releases include one-off migration commands. The client's update notice and the **Instance settings, Updates** panel say which releases need manual steps.
2. Back up both stores.
3. Pull the new images and restart (`docker compose pull && docker compose up -d`). The server entrypoint runs `fdp db migrate` on start.
4. Check `GET /fdp-api/info` for the running version and `GET /fdp-api/readyz` for readiness.

A deployment that relies on ODRL policies and upgrades past 0.17.0 must set `FDP_ACCESS_CONTROL_MODE=odrl` (or switch *Access control* in the settings), because open access became the default. See [Access policies](../usage/policies.md#access-control-mode).

### Upgrading to server 0.19 (PostgreSQL 18, Keycloak 26)

Server 0.19 moves the bundled stacks to PostgreSQL 18 and Keycloak 26. Both need a one-off step on an existing deployment. After pulling the new compose files, run these two scripts from the stack directory (`deploy/stack`, or `deploy/stack/production` with `../` instead of `./`):

```bash
./upgrade-postgres-18.sh      # dump PostgreSQL 16, restore into 18
./fix-keycloak-audience.sh    # restore the `fdp` token audience on Keycloak 26
```

- **PostgreSQL.** A major version cannot open an older data directory. Until migrated, the `postgres` container exits with "old database(s) found". The script dumps every database, FDP and Keycloak alike, and keeps a copy of the old volume named `<project>_postgres-data-pg16-backup`. It then restores into PostgreSQL 18, checks the database list and restarts the stack. Remove the backup volume and the `postgres-upgrade-*` directory once you are satisfied.
- **Keycloak.** Keycloak 26 stopped emitting the `fdp` audience the way older realm files declared it, so every sign-in fails with "invalid bearer token". New installs import the corrected realm. An existing realm is never re-imported, so the script updates its audience mapper.

Both scripts are idempotent.

Pin `SERVER_IMAGE_TAG` and `CLIENT_IMAGE_TAG` to release tags rather than `main` for reproducible deployments.

## Health endpoints

| Endpoint | Meaning |
|---|---|
| `GET /fdp-api/healthz` | Liveness. |
| `GET /fdp-api/readyz` | Readiness; checks the downstream stores and identity provider. |
| `GET /fdp-api/info` | Name, version, environment, build information. |
| `GET /fdp-api/config` | Public self-description: base and serving URL, OIDC issuer and audience, applied profile name and version, enabled features. |
