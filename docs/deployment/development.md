# Development setup

For working on the server or the client code, run only the backing services in Docker and run the server and client on the host, so you get hot reload and a debugger. If you only want a running FDPneo, use the [local deployment](local.md) instead.

## Prerequisites

- Docker with the Compose plugin, for GraphDB, PostgreSQL and Keycloak.
- [uv](https://docs.astral.sh/uv/) for the server (`curl -LsSf https://astral.sh/uv/install.sh | sh`). The server needs Python 3.12 or later; uv installs it if needed.
- Node.js 20.19 or later (or 22.12 or later) with npm for the client.
- `git`, and two clones side by side, for example `fdpneo/server` and `fdpneo/client`.

## 1. Backing services

From the server repository:

```bash
docker compose -f deploy/compose.yaml up -d
docker compose -f deploy/compose.yaml ps     # wait until all are healthy
```

This development compose starts GraphDB on `:7200`, PostgreSQL on `:5432` (`fdp` / `fdp`, database `fdp`) and Keycloak on `:8080` (`admin` / `admin`) with the `fdp-dev` realm pre-imported. It does not start the server or the client. All credentials are development-only.

The server expects a GraphDB repository named `fdp`. If it does not exist, create it once in the GraphDB workbench at <http://localhost:7200> (Setup, Repositories, Create new repository, GraphDB repository, ID `fdp`), or preload a repository configuration through `deploy/graphdb/preload/`.

## 2. Server

```bash
cd server
cp .env.example .env                 # defaults match the development compose
uv sync                              # add --extra dev for the test and lint tooling
uv run fdp db migrate                # apply database migrations (Alembic)
uv run fdp profile apply             # bootstrap with the bundled default profile
uv run fastapi dev src/fdpneo_server/main.py    # http://localhost:8000, auto-reload
```

Sanity checks, remembering the `/fdp-api` prefix:

```bash
curl -s http://127.0.0.1:8000/fdp-api/healthz                       # liveness
curl -s http://127.0.0.1:8000/fdp-api/config                        # bootstrap JSON
curl -s http://127.0.0.1:8000/ -H 'Accept: text/turtle'              # root FDP record
open http://127.0.0.1:8000/fdp-api/docs                              # Swagger UI
```

The server reads its configuration from environment variables or `.env` through pydantic-settings; see [Configuration](configuration.md).

## 3. Client

```bash
cd client
cp .env.example .env                 # VITE_FDP_API_URL, VITE_PUBLIC_ORIGIN, OIDC fallbacks
npm install
npm run generate-api                 # optional: regenerate src/api/schema.ts from the running server's OpenAPI
npm run dev                          # http://localhost:5173
```

At start-up the client fetches `GET /fdp-api/config` from the server and takes the OIDC issuer and client id from there; the `VITE_OIDC_*` values are only a fallback for when that endpoint is unavailable.

Open <http://localhost:5173> and sign in with `admin` / `admin`.

## The origin coupling

Because the single-page application calls the API cross-origin and authenticates against Keycloak, three origins must agree. A mismatch produces "server unreachable" on writes or a failed login redirect.

1. `VITE_PUBLIC_ORIGIN` (client): the exact origin you open the application at.
2. `FDP_CORS_ALLOW_ORIGINS` (server): must include that origin.
3. The Keycloak client's redirect URIs: must allow that origin's `/auth/callback`.

`http://localhost:5173` and `http://127.0.0.1:5173` are different origins to the browser; pick one and use it everywhere. The client warns in development mode on a mismatch. Never use `*` for `FDP_CORS_ALLOW_ORIGINS` while credentials are allowed.

## Quality gate

The server's CI runs lint, format check, strict type check and the test suite. Run the same before opening a pull request:

```bash
uv run ruff check . && uv run ruff format --check . \
  && uv run pyright \
  && uv run pytest
```

Integration tests use testcontainers for GraphDB and PostgreSQL. If you change the public HTTP API, the OpenAPI specification changes with it and the client needs a coordinated `npm run generate-api`; flag this in your pull request.

The client uses Vitest for unit tests (colocated `*.spec.ts` files) and Playwright for end-to-end smoke tests:

```bash
npm run test:unit
npm run test:e2e
```

See [Contributing](../development/contributing.md) for the DCO sign-off and the pull-request process, and the server's [developer documentation](https://github.com/FAIRDataTeam/fdpneo-server/tree/main/docs/dev-docs) for code organisation, the request lifecycle and the key processes.
