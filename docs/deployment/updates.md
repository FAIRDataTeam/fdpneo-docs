# Updates

FDPneo tells administrators when a newer release exists, and can optionally apply it for them. The design is recorded in ADR-0028 (release tags and update notifications) and ADR-0030 (the optional host updater).

## Releases and image tags

Every release of the server and the web client publishes its image under three tags: the full version (`0.21.1`), the minor line (`0.21`) and `latest`. From server 0.21.1 and client 0.9.2 on, release images are built for `linux/amd64` and `linux/arm64` and signed. The `main` and commit tags are development builds, for `linux/amd64` only and unsigned.

Server and client have their own version numbers, so the compose stacks take `SERVER_IMAGE_TAG` and `CLIENT_IMAGE_TAG`, each falling back to `IMAGE_TAG`. Pin both to release versions in production. That makes a deployment reproducible and a rollback a one-line change.

Each release also carries a small `fdpneo-release.json` with its upgrade metadata: the oldest version it can be upgraded from, whether it needs manual steps, whether it fixes a vulnerability, and, for the client, the server version it requires.

### Verifying an image

Release images are signed by digest with Sigstore keyless signing, which binds the signature to the repository, its release workflow and the tag, and they carry a build-provenance attestation. Anyone can check them:

```bash
cosign verify ghcr.io/fairdatateam/fdpneo-server:0.21.1 \
  --certificate-identity https://github.com/FAIRDataTeam/fdpneo-server/.github/workflows/release.yml@refs/tags/v0.21.1 \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
gh attestation verify oci://ghcr.io/fairdatateam/fdpneo-server:0.21.1 --repo FAIRDataTeam/fdpneo-server
```

## Update notices

Once a day the server asks the GitHub Releases API whether a newer server or web client exists, and keeps the answer in its database. Administrators see the result in two places.

- **A banner** under the header, when either component is behind. It flags security releases and releases that need manual steps, and says when a client release needs a newer server first. Dismissing it lasts for the browser session.
- **The Updates panel** under avatar, Settings, with the running and latest versions, the time of the last check, any error from it, and a *Check now* button.

Nothing about the deployment is sent. GitHub sees the server's IP address and a `fdpneo-server/<version>` User-Agent. To switch the check off, set `FDP_UPDATES_CHECK_ENABLED=false` (no request is ever made, for air-gapped or restricted sites), or turn off the *Update check* setting, which an administrator can do without a restart.

## Updating by hand

1. Read the release notes, linked from the banner, for every version you cross. Some releases need one-off steps, such as the PostgreSQL 18 and Keycloak 26 scripts of server 0.19.
2. Take a full backup (see [Backup and restore](backup-and-restore.md)).
3. Set the new versions in `.env` (`SERVER_IMAGE_TAG`, `CLIENT_IMAGE_TAG`), then `docker compose pull && docker compose up -d`. The server runs its database migrations on start.
4. Check `GET /fdp-api/info` for the running version.

## The optional host updater

The host updater is a small agent that applies an update an administrator requests in the client. It is **off by default**. Update notices work without it.

!!! warning "Root-equivalent privilege"
    To replace containers the agent controls the Docker daemon. On a standard (rootful) Docker host that is equivalent to root, whether through the Docker socket or membership of the `docker` group. With rootless Docker or Podman it controls only the service user's own containers. Kubernetes is not supported; use your deployment tooling there. Only enable the agent if your security policy accepts that privilege.

### What it does

The agent polls the server about once a minute, which also tells the server it is connected. While it is, the Updates panel offers *Update to X* for the newest release it may apply, after a confirmation step. For each request it:

1. **Re-checks the release on GitHub itself.** It must be published and not a pre-release, carry its upgrade metadata without manual steps, accept the running version as a starting point, and, for a client release, find a new enough server.
2. **Pulls the image and verifies its signature.** It pins the digest it received and checks with `cosign` that the release workflow signed exactly that tag. An unsigned or wrongly signed image is refused before anything changes. Releases before server 0.21.1 and client 0.9.2 are unsigned, so apply those by hand.
3. **Takes a full encrypted backup** with the stack's `backup.sh`, and keeps the newest three. If the backup fails, nothing changes.
4. **Applies the update.** It writes `SERVER_IMAGE_TAG` or `CLIENT_IMAGE_TAG` in `.env` and recreates only that service.
5. **Waits for the new version** to report on `/fdp-api/info`.
6. **Reports back.** If the new version does not come up and no migration ran, it restores the previous tag and reports *Rolled back*. If a migration ran, the old version cannot run on the new schema, so it reports *Needs attention* with the backup's location, and restoring it is your decision.

It never edits compose files, never touches PostgreSQL, Keycloak, GraphDB or Caddy, never applies a release that needs manual steps, and never acts without an administrator's request. The panel lists recent requests with their status and backup location, and pending requests can be cancelled.

### Enabling it

1. **Mint an updater token.** As an administrator, open avatar, Access tokens, tick *Updater token* and generate it. It carries only the updater role, the server refuses it everywhere except the agent's endpoints, and it stops working if its owner loses the admin role.
2. **Store the backup passphrase** in a file only the agent can read:

    ```bash
    install -m 0600 /dev/null /srv/fdpneo/.backup.pass
    $EDITOR /srv/fdpneo/.backup.pass   # one line: the passphrase
    ```

3. **Add the settings to the stack's `.env`:**

    ```bash
    FDPNEO_ROOT=/srv/fdpneo                       # absolute path of the checkout
    FDP_UPDATER_KEY=fdpk_…                        # the updater token
    UPDATER_BACKUP_PASSPHRASE_FILE=/srv/fdpneo/.backup.pass
    ```

    Optional settings include `UPDATER_BACKUP_DIR`, `UPDATER_BACKUP_KEEP`, `UPDATER_POLL_SECONDS`, `UPDATER_READY_TIMEOUT`, `DOCKER_SOCKET` (rootless Docker) and `UPDATER_VERIFY_SIGNATURES=false` (only for mirrors that drop signatures).

4. **Start the agent** with its compose profile:

    ```bash
    docker compose --profile updater up -d --build --no-deps updater
    ```

    Keep `--no-deps`. Without it, `--build` also rebuilds every service with a `build:` section, which in the evaluation stack replaces the release images with local builds.

The agent can also run as a systemd service instead of a container; the server repository's [`deploy/updater/README.md`](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/deploy/updater/README.md) has the unit file and the full reference. To disable it, run `docker compose stop updater` (or start the stack without the profile) and revoke the updater token.
