# Production deployment

The **single-domain production stack** in `deploy/stack/production/` of the server repository serves the client, the API with its record IRIs, and Keycloak under **one HTTPS origin** behind [Caddy](https://caddyserver.com/), with certificates obtained and renewed automatically through ACME. One origin means there is no CORS to configure and the OIDC issuer is consistent for the browser and the server by construction, which removes the two classic failure modes of ad-hoc deployments.

```text
https://fdp.example.org/            browsers → client (SPA)
                                    RDF content negotiation → server (record IRIs)
https://fdp.example.org/fdp-api/…   the API (OpenAPI, SPARQL, /users, …)
https://fdp.example.org/auth/…      Keycloak (production mode)
https://fdp.example.org/mcp         MCP bridge (optional, --profile mcp)
```

Unlike the [evaluation stack](local.md), this one is production-shaped: Keycloak runs in `start` mode against its own PostgreSQL database, the realm template contains no demo users and takes its secrets from the environment, brute-force protection is on, and GraphDB and PostgreSQL are never published.

## Prerequisites

- A DNS name (`PUBLIC_HOST`) that resolves to the host from the internet.
- Ports 80 and 443 reachable from the internet (certificate issuance and serving).
- Docker Engine 24 or later with the Compose plugin. Images are pulled from the public GHCR packages.

## Deploy

```bash
git clone https://github.com/FAIRDataTeam/fdpneo-server.git
cd fdpneo-server/deploy/stack/production
cp .env.example .env
# set PUBLIC_HOST and ACME_EMAIL, and replace every CHANGE-ME with a real secret
docker compose up -d
```

Every `CHANGE-ME` is required; the stack refuses to start without them. A convenient way to generate values is `openssl rand -hex 24` per secret.

| Variable | Purpose |
|---|---|
| `PUBLIC_HOST` | The public DNS name, without scheme or port. Caddy obtains the certificate for it. |
| `ACME_EMAIL` | Contact for certificate issuance and expiry notices. |
| `IMAGE_TAG` | Image tag to pull, `latest` by default. Releases publish `X.Y.Z`, `X.Y` and `latest`; `main` follows unreleased development. |
| `SERVER_IMAGE_TAG`, `CLIENT_IMAGE_TAG` | Pin the server and the client to their own releases, for example `0.20.0` and `0.9.0`. Each falls back to `IMAGE_TAG`. |
| `EXPOSE_API_DOCS` | Serve Swagger and ReDoc (`false` by default; the OpenAPI spec is always served). |
| `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` | The server's operational database. |
| `KC_DB_PASSWORD` | Keycloak's own database role, created on first boot. |
| `KEYCLOAK_ADMIN`, `KEYCLOAK_ADMIN_PASSWORD` | Keycloak master-realm administrator (console at `/auth`). |
| `FDP_SERVER_CLIENT_SECRET` | Secret of the confidential `fdp-server` client, used by the user-management facade. One variable, imported into the realm and handed to the server. |
| `FDP_ADMIN_EMAIL`, `FDP_ADMIN_INITIAL_PASSWORD` | The bootstrap FDPneo administrator `fdp-admin` in the `fdp` realm. The password is temporary; Keycloak forces a change at first login. |
| `FDP_OIDC_AUDIENCE` | Token audience the realm's mapper stamps (`fdp`). |
| `FDP_INDEX_PING_TARGETS` | Optional. FDP Index instances to announce this FDP to. |
| `FDP_REMOTE_LABELS_*` | Optional. External label resolution for DOI, ORCID and vocabulary IRIs. |

First boot takes a couple of minutes (Keycloak schema, GraphDB repository creation, database migrations, default profile). Then:

- UI: `https://PUBLIC_HOST/`
- API health: `https://PUBLIC_HOST/fdp-api/healthz`
- Keycloak admin console: `https://PUBLIC_HOST/auth`, with the `KEYCLOAK_ADMIN` account

## First login and users

Sign in to FDPneo as `fdp-admin` with `FDP_ADMIN_INITIAL_PASSWORD`; Keycloak asks for a new password immediately. Create further users in the Keycloak admin console (realm `fdp`), assigning the `admin` or `steward` realm role, or through FDPneo's own Users screen, which is backed by the `/fdp-api/users` facade (see [User management](user-management.md)).

## How the routing works

The server owns the origin's IRI space (records dereference at the root, any path except `/fdp-api`), while the single-page application needs the same origin for its routes. Caddy splits the two on content negotiation, the way the FDP reference implementation does: requests preferring `text/html` (browsers) go to the client; everything else (`text/turtle`, `application/ld+json`, `*/*`, that is, machine clients) dereferences straight to the server. `/fdp-api/*` and `/auth/*` are plain path routes.

`BASE_URL` is `https://PUBLIC_HOST`, so minted IRIs, the OIDC issuer and the page origin all agree. The server reaches the public issuer URL through the Docker host (`extra_hosts: host-gateway`), so token validation works even where the network cannot hairpin its own public IP.

## MCP bridge (optional)

```bash
docker compose --profile mcp up -d
```

This serves the read-only agent bridge at `https://PUBLIC_HOST/mcp`. It needs pull access to the `ghcr.io/fairdatateam/fdp-mcp` image, which is private at the time of writing, or a sibling `mcp` checkout to `--build`. See [MCP bridge](../agents/mcp.md).

## Using nginx instead of Caddy

If the host already runs nginx, keep the same one-origin routing contract and replace only the edge.

1. Publish the application containers on loopback ports and drop the Caddy service:

    ```bash
    docker compose -f compose.yaml -f compose.nginx.yaml up -d --scale caddy=0
    ```

    `compose.nginx.yaml` binds client, server and Keycloak (and the bridge, with the profile) to `127.0.0.1:8090–8093`, reachable only through nginx.

2. Obtain a certificate, since nginx does not do ACME itself:

    ```bash
    sudo certbot certonly --webroot -w /var/www/certbot -d fdp.example.org
    ```

3. Install the site configuration `nginx/fdpneo.conf`, replacing `fdp.example.org` with your `PUBLIC_HOST` and checking the certificate paths, then:

    ```bash
    sudo nginx -t && sudo systemctl reload nginx
    ```

The configuration reproduces the Caddyfile: `/fdp-api` to the server, `/auth` to Keycloak, `/mcp` to the bridge, static files to the client, and for everything else an `Accept`-header map that sends `text/html` to the client and all other content negotiation to the server.

Forwarded headers matter. Keycloak (`KC_PROXY_HEADERS=xforwarded`) and the server's rate limiter (`FDP_RATELIMIT_TRUST_FORWARDED_FOR`) both key on `X-Forwarded-*`, and the server process only believes `X-Forwarded-Proto` and `X-Forwarded-For` from addresses listed in `FORWARDED_ALLOW_IPS` (uvicorn's setting, which by default trusts only `127.0.0.1`). The bundled compose sets it to `*` because the server port is never published; if nginx runs on the host, `127.0.0.1` is the right value. Without it the application sees every request as plain `http` from the proxy's IP, which produces wrong client IPs in the metrics geography and `http://` URLs in logs.

## Operations

**Updates.** Administrators are told about new releases in the client (see [Configuration](configuration.md#update-notifications)). To update, set `SERVER_IMAGE_TAG` and `CLIENT_IMAGE_TAG` to the new releases, then run `docker compose pull && docker compose up -d`. A pinned tag also makes a rollback a one-line change. Read the release's upgrade notes first when it is marked as needing manual steps.

**Metrics roll-ups** run in-process (`FDP_METRICS_ROLLUP_IN_PROCESS=true`): raw request rows are aggregated to hourly and daily rows and pruned on a fixed interval, which is also the privacy boundary of ADR-0002, since raw rows carry the pseudonymous visitor hash. If you run more than one server replica, unset it and schedule `docker compose exec server fdp metrics rollup` (every five minutes or so) instead, so replicas do not race.

**Upgrading from a server before 0.17.** Recorded metrics were keyed by the URL the container saw (`http://<host>/…`), so per-record metrics never matched the `https://` record IRIs. Re-key the history once with `docker compose exec server fdp metrics rebase-resources --from http://<PUBLIC_HOST> --dry-run`, then again without `--dry-run`.

**Backups.** State lives in the `postgres-data` volume (operational state and Keycloak) and the `graphdb-data` volume (the knowledge graph); they must be backed up and restored together. `caddy-data` holds certificates, which are re-obtainable, but backing it up avoids ACME rate limits on rebuilds. To move the deployment to another host, or to keep a complete copy, use `../backup.sh` and `../restore.sh`; see [Operations](operations.md#moving-or-fully-restoring-a-bundled-stack).

**No public DNS yet?** Add `tls internal` inside the site block of the `Caddyfile` to smoke-test with a self-signed certificate. Browsers will warn and the server's issuer fetch will fail certificate verification, so login stays broken until real TLS is in place; this mode only checks routing.

## Hardening checklist

Most items of the [deployment-hardening checklist](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/security/deployment-hardening.md) (TLS, security headers, secret rotation, internal-only backing stores) are already satisfied by the shape of this stack. The remaining ones are operational: offsite backups, monitoring, and multi-factor authentication in the identity provider. In summary:

- TLS everywhere, with `BASE_URL` set to the server's public `https` URL, because it mints the persistent resource IRIs.
- Replace every development credential: a new realm and admin password, strong database credentials, and GraphDB security with a service user (`FDP_TRIPLESTORE_USERNAME` / `_PASSWORD`).
- Keep secrets in a secret manager or the environment, never in version control.
- List only the real client origin in `FDP_CORS_ALLOW_ORIGINS` if you deviate from the single-origin layout; never `*` with credentials.
- Back up both stores together.
- Monitor `GET /fdp-api/healthz` (liveness), `GET /fdp-api/readyz` (readiness, checks downstreams) and `GET /fdp-api/info` (build information).
