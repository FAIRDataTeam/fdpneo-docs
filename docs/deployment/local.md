# Local deployment

The quickest way to run FDPneo is the **full-stack compose** in the server repository. One command brings up the client, the server and the backing services (GraphDB, PostgreSQL, Keycloak), and bootstraps the GraphDB repository, the database schema and the default metadata profile automatically on first boot.

This stack is for evaluation, demos and local use. It runs over plain HTTP on `localhost` with publicly known development credentials. For a real server, use the [production deployment](production.md).

## Prerequisites

- Docker Engine 24 or later with the Compose plugin (Docker Desktop on macOS and Windows includes it).
- `git`, to clone the server repository.
- Nothing else for the default path: the `ghcr.io/fairdatateam/fdpneo-server` and `ghcr.io/fairdatateam/fdpneo-client` images are public, so no registry login is needed.

## Start the stack

```bash
git clone https://github.com/FAIRDataTeam/fdpneo-server.git
cd fdpneo-server
cp deploy/stack/.env.example deploy/stack/.env
docker compose -f deploy/stack/compose.yaml --env-file deploy/stack/.env up -d
```

The defaults in `.env` are right for a single machine. First boot pulls the images and takes a few minutes while Keycloak imports its realm, the init container creates the `fdp` repository in GraphDB, the server runs its migrations and the default profile is applied. Watch progress with:

```bash
docker compose -f deploy/stack/compose.yaml --env-file deploy/stack/.env ps
```

When every service reports `healthy`, open:

| What | URL |
|---|---|
| Web client | <http://localhost:5173> |
| API | <http://localhost:8000/fdp-api/> (health at `/fdp-api/healthz`, self-description at `/fdp-api/config`) |
| OpenAPI specification | <http://localhost:8000/fdp-api/openapi.json> |
| Interactive API docs | <http://localhost:8000/fdp-api/docs> (Swagger) and <http://localhost:8000/fdp-api/redoc> |
| Keycloak admin console | <http://localhost:8081> (`admin` / `admin`) |

Sign in to the client with `admin` / `admin` (administrator and steward), `alice` / `alice` (steward) or `bob` / `bob` (no roles). See [Signing in and roles](../usage/signing-in.md).

!!! warning "Development credentials"
    The bundled Keycloak realm runs in development mode with the users above, the admin password `admin`, and the client secret `fdp-server-dev-secret`. They are fine for evaluation on your own machine and nothing else.

## What `.env` controls

The evaluation stack's `.env` sets the public host and ports, the image tag, the runtime mode and the development credentials.

| Variable | Default | Meaning |
|---|---|---|
| `PUBLIC_HOST` | `localhost` | Host the browser uses to reach the stack. Must resolve identically inside the containers, because the server reaches the Keycloak issuer at this host. |
| `CLIENT_PORT` | `5173` | Must stay `5173` with the bundled realm, which only allows OIDC redirect URIs at `http://localhost:5173`. |
| `SERVER_PORT` | `8000` | Published API port. |
| `KEYCLOAK_PORT` | `8081` | Published Keycloak port. |
| `MCP_PORT` | `8002` | Published port of the optional MCP bridge. |
| `IMAGE_TAG` | `main` | Image tag pulled from GHCR. Releases publish `X.Y.Z`, `X.Y` and `latest`. `SERVER_IMAGE_TAG` and `CLIENT_IMAGE_TAG` override it per component. |
| `ENVIRONMENT` | `production` | Server runtime mode (`production` or `development`). |
| `EXPOSE_API_DOCS` | `true` | Serve Swagger and ReDoc even in production mode. The OpenAPI spec is always served. |
| `POSTGRES_*`, `KEYCLOAK_ADMIN*`, `FDP_IDP_ADMIN_*` | dev values | Credentials; rotate for anything beyond evaluation. |
| `FDP_INDEX_PING_TARGETS` | unset | Comma-separated FDP Index base URLs to announce this FDP to. |
| `FDP_REMOTE_LABELS_*` | unset | Enable resolving labels of external IRIs (DOI, ORCID, vocabularies) over an allow-list of hosts. |

## Optional: the MCP bridge for agents

The [`fdp-mcp`](../agents/mcp.md) bridge can run alongside the stack at `http://localhost:8002/mcp`. It sits behind the `mcp` compose profile, so a default `up` skips it:

```bash
docker compose -f deploy/stack/compose.yaml --env-file deploy/stack/.env --profile mcp up -d
curl -fsS http://localhost:8002/healthz
```

At the time of writing the `ghcr.io/fairdatateam/fdp-mcp` image is not publicly pullable; either `docker login ghcr.io` with a token that can read the package, or check out the `mcp` repository as a sibling and add `--build`.

## Build from source instead of pulling

Check out the client repository next to the server (`fdpneo/server` and `fdpneo/client`), then:

```bash
docker compose -f deploy/stack/compose.yaml --env-file deploy/stack/.env up -d --build
```

The server build context is the server repository root; the client context is the sibling `../client`.

!!! note "Apple Silicon and other arm64 hosts"
    Release images from server 0.21.1 and client 0.9.2 onwards are published for both `linux/amd64` and `linux/arm64`. Pin them with `SERVER_IMAGE_TAG` and `CLIENT_IMAGE_TAG`, or use `IMAGE_TAG=latest`. The `main` development images and older releases are `linux/amd64` only, so on an arm64 host a pull of those fails with "no matching manifest". Build them from source instead, as above.

!!! note "GeoIP database month when building the server"
    The server image downloads the DB-IP city database at build time, and the month pinned in the `Dockerfile` ages out after a while (the download then fails with a 404). Pass the current month explicitly, for example `docker build --build-arg DBIP_CITY_VERSION=2026-10 -t ghcr.io/fairdatateam/fdpneo-server:main .` from the server repository, then start the stack without `--build`. The CI workflow and `scripts/fetch-geoip.sh` resolve the newest month automatically.

## Running on another host on your network

Set `PUBLIC_HOST` to a name or IP that resolves the same from the browser and from inside the containers (a LAN IP or DNS name), and adjust the ports if they clash. Login still only works over plain HTTP at the exact origin `http://<PUBLIC_HOST>:5173`, because the bundled realm restricts redirect URIs. For HTTPS or any public exposure, do not adapt this stack; use the [single-domain production stack](production.md).

## How the pieces fit together

| Service | Published | Role |
|---|---|---|
| `client` | `5173` → 80 | Vue single-page app served by nginx. The API URL is injected at start-up into `/config.js`. |
| `server` | `8000` | FastAPI API, mounted at `/fdp-api`, and the record IRI space (`BASE_URL`). |
| `mcp` | `8002` → 8000 | MCP bridge, opt-in via `--profile mcp`. |
| `keycloak` | `8081` → 8080 | OIDC login, browser-facing. |
| `graphdb` | internal | Triple store; `graphdb-init` creates the `fdp` repository. |
| `postgres` | internal | Operational state. |

The browser talks to the client for the UI, to the server for the API (CORS-allowed) and to Keycloak for login. The server reaches GraphDB, PostgreSQL and Keycloak over the internal network and resolves `PUBLIC_HOST` to the Docker host, so the OIDC issuer URL is identical for the browser and the server.

Bootstrapping is automatic and idempotent: `graphdb-init` creates the `fdp` repository from `deploy/graphdb/fdp-repo-config.ttl` if it is missing, the server entrypoint runs `fdp db migrate`, and the application auto-applies the bundled default profile (`FDP_PROFILE_AUTO_APPLY=true`) and runs a named-graph isolation self-test.

Access control starts in **open** mode: every published record is publicly readable and `admin` and `steward` may write, with no ODRL evaluation. Set `FDP_ACCESS_CONTROL_MODE=odrl` in `.env`, or switch *Access control* under avatar, Settings as an administrator, to enforce per-record [access policies](../usage/policies.md).

## Stop, update, reset

```bash
# stop, keep data
docker compose -f deploy/stack/compose.yaml --env-file deploy/stack/.env down

# update to the latest images
docker compose -f deploy/stack/compose.yaml --env-file deploy/stack/.env pull
docker compose -f deploy/stack/compose.yaml --env-file deploy/stack/.env up -d

# stop and wipe all data (GraphDB, PostgreSQL, Keycloak)
docker compose -f deploy/stack/compose.yaml --env-file deploy/stack/.env down -v
```

Administrative commands run inside the server container, for example `docker compose -f deploy/stack/compose.yaml --env-file deploy/stack/.env exec server fdp profile info`. See [Operations](operations.md).

## Next steps

- [Signing in and roles](../usage/signing-in.md), then the rest of the usage guide.
- [Configuration](configuration.md) for the server and client settings behind the stack.
- [Development setup](development.md) if you want to change the server or client code.
