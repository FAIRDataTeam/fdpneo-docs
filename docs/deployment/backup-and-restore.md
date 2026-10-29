# Backup and restore

FDPneo backs up a whole deployment, not only its metadata. An archive can restore the same instance after a failure, or recreate it on another host under a new host name. The design is recorded in ADR-0016 (faithful backup of the knowledge graph) and ADR-0029 (full-deployment backup).

## What a backup contains

| Scope | Contents | Format |
|---|---|---|
| **Records only** | Every named graph: records, their meta-metadata and their audit graphs. | `.zip` archive |
| **Full deployment** | The records plus the PostgreSQL state: runtime settings, API keys, saved searches, FDP Index targets, applied profiles, subject roles, the audit trail and the metrics history. | Encrypted archive (`.fdpbk`) |

A full backup is always encrypted with a passphrase (scrypt and AES-256-GCM, format `FDPNEO-ENC1`). Keep the passphrase somewhere safe, because without it the archive cannot be opened.

Raw metrics rows are never included, because they carry the pseudonymous visitor hash that the metrics design discards (ADR-0002). Derived tables, such as the search index and the authorization cache, are rebuilt after a restore instead of being copied.

## In the web client

Open **avatar, Backup & Restore** as an administrator.

**Create a backup.** Choose *Records only* or *Full deployment*. A full backup asks for the passphrase twice. The backup runs as a background job on the server, and *Download archive* appears when it is ready. While a full backup or restore runs, writes are paused and answered with `503` and a `Retry-After` header. Reads continue.

**Restore from a backup.** Upload an archive and choose what happens to records that already exist: *Merge* skips them, *Overwrite* replaces them. *Preview (dry run)* reports what would change without writing anything. An encrypted archive asks for its passphrase. A full restore also brings back settings and API keys, and the server must run the same version as when the backup was taken. Uploads are limited to 10 MiB. Restore larger archives with the command line.

Adopting records from another deployment under this deployment's identifier base (import with rebase) is intentionally a command-line action.

## On the command line

Run these inside the server container (see [Operations](operations.md#the-fdp-command-line-tool)). Each has `--help`.

| Command | What it does |
|---|---|
| `fdp backup dump` | Dump every named graph with a manifest. With `--full`, add the PostgreSQL state and encrypt the archive. |
| `fdp backup restore` | Restore an archive. A full archive first moves the database schema to the archive's revision, loads the rows, then migrates to the current version. |
| `fdp backup import` | Import records from another deployment. `--rebase` adopts them under this deployment's base IRI; it accepts directories, `.zip` and encrypted archives. |
| `fdp backup seal`, `unseal` | Encrypt or decrypt an archive. |
| `fdp backup verify` | Check an archive without restoring it. |

After a records-only restore or an import, rebuild the search index with `fdp search reindex`.

## Moving a bundled stack

Both compose stacks ship two scripts that wrap a full backup together with what lives outside FDPneo's own stores.

- **`backup.sh`** writes one encrypted bundle: the full archive, the Keycloak realm with its users, and `.env` if there is one. The server and Keycloak stop for the moment of the dump and start again. Keycloak's export keeps user IDs, so token subjects and API keys stay valid after a move.
- **`restore.sh BUNDLE.fdpbk`** recreates the deployment from a bundle, on a fresh checkout of the same release. It asks for the passphrase, lets you change `PUBLIC_HOST`, restores into empty volumes and starts the stack. It refuses a stack that already has data unless `FORCE=1` is set. A new host name is handled for you: records are rebased, the realm's URLs follow and metrics are re-keyed.

For the evaluation stack, run them from `deploy/stack`:

```bash
./backup.sh                                   # → fdpneo-backup-<timestamp>.fdpbk
./restore.sh fdpneo-backup-<timestamp>.fdpbk  # on the target machine
```

For the production stack, run `../backup.sh` and `../restore.sh` from `deploy/stack/production`, and point DNS at the new host before Caddy requests certificates. Set `BACKUP_PASSPHRASE_FILE` to a file holding the passphrase to avoid the prompt, for example in a scheduled job.

!!! warning "Test a restore"
    A backup is only as good as a restore you have tried. On the evaluation stack you can test one on the same machine with `docker compose … down -v` followed by `./restore.sh`, but `down -v` deletes all data, so only do this with a bundle you have just written.

The optional [host updater](updates.md#the-optional-host-updater) takes the same kind of bundle before every update it applies.
